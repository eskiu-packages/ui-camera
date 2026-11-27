# Live frames

> **The handler runs on the UI thread.** While it runs the interface waits. Converting a
> 1080p frame to RGBA there (two million pixels; in a `ui serve` build, compiled at `-O0`,
> several times slower) freezes the app. Copy the frame with `CameraFrame_copy_to` (one
> `memcpy` a plane) and convert or analyze the copy on a worker thread, as in
> [Threads](#threads), or ask for a smaller size.

`use_camera_frames` hands the mounted `CameraView`'s camera to Eskiu frame by frame, as the
platform delivers it: YUV planes on iOS and Android, RGBA on the web. It is for code that
looks at the pictures as they come (a barcode or face detector, a liveness check, a video
encoder, a texture upload), as opposed to `camera_capture`, which saves one photo.

```eskiu
import <camera>;

uint8[640 * 640 * 4] pixels;

void Scanner() {
    Column(Style{gap: 12}) {
        CameraView(Style{height: 360}, false);
        use_camera_frames(CameraFrameOptions{width: 640, height: 480, fps: 15}, void(*CameraFrame frame) {
            // Borrowed for this call: read it, convert it or copy it, then return.
            int w = frame.out_width(true);
            int h = frame.out_height(true);
            frame.to_rgba(&pixels[0], w * 4, true);
            detect(&pixels[0], w, h);
        });
    }
}
```

## How a frame gets to Eskiu

The pixels are never copied into a message. The native side keeps the newest frame the
camera produced and posts a one-word event (`frame` on the `camera_frame` channel) when one is
waiting and no event is posted yet. When the event reaches Eskiu on the UI thread,
`use_camera_frames` borrows that frame through `camera_native_frame_acquire`, which fills a
small array with its size, layout and plane addresses; the handler reads the platform's own
buffer by pointer, and `camera_native_frame_release` gives it back when the handler returns.

| Platform | Source | Buffer lent to Eskiu |
|---|---|---|
| iOS | `AVCaptureVideoDataOutput` on the view's session, `420YpCbCr8BiPlanarFullRange` | The `CVPixelBuffer`, base address locked read-only for the call: NV12 |
| macOS | The same, its output scaled to the size asked for | The same: NV12 |
| Android | A `YUV_420_888` `ImageReader` on the view's Camera2 session | The `Image`'s planes (direct buffers, read in place): NV12, NV21, I420 or the general three-plane layout, as the device gives it |
| Web | The view's `<video>` (getUserMedia), on a timer at the asked rate | The picture drawn into a canvas and read back as RGBA into one buffer in the app's memory, reused |

### Back-pressure

A frame that arrives while the handler runs, or while the event waits behind other work on the
UI thread, replaces the one waiting; the one it replaces is dropped. So a slow handler gets
fewer, newer frames, and nothing queues up. The platform holds at most two frames for Eskiu:
the one waiting and the one lent. On iOS the output also discards frames that arrive late
(`alwaysDiscardsLateVideoFrames`).

### Threads

The handler runs on the UI thread, like every `ui` handler, so it can set signals. Heavy work
there holds up the interface. A frame's pointers are not valid after the handler returns, so
work that takes longer than a frame starts from a copy: `CameraFrame_copy_to(frame, &copy)`
copies the planes into a `CameraFrameCopy` (its buffer is kept and reused, so a warm copy
allocates nothing), whose `frame` reads the same way and stays valid until the next copy into
it or `CameraFrameCopy_free`.

A worker thread with a mailbox of one: the handler copies the newest frame in when the worker
is free (a frame that comes while it works is dropped, as frames are), and the worker posts
its result back with `ui_native_emit`, which any thread may call; `use_native_event` gets it
on the UI thread.

```eskiu
import <threading>;
import <camera>;

extern void ui_native_emit(string channel, string payload);

struct Mailbox { Mutex lock; Cond ready; CameraFrameCopy copy; bool full; bool busy; }
Mailbox box;
uint8[1280 * 720 * 4] pixels;

void start_worker() {
    Mutex_init(&box.lock);
    Cond_init(&box.ready);
    *Mailbox m = &box;
    thread_create(void() {
        while (true) {
            Mutex_lock(&m.lock);
            while (!m.full) { Cond_wait(&m.ready, &m.lock); }
            m.full = false;
            m.busy = true;
            Mutex_unlock(&m.lock);
            // The copy is the worker's until busy is false again.
            *CameraFrame f = &m.copy.frame;
            f.to_rgba(&pixels[0], f.out_width(true) * 4, true);
            // ... run the model on pixels ...
            ui_native_emit("detections", "3 faces");
            Mutex_lock(&m.lock);
            m.busy = false;
            Mutex_unlock(&m.lock);
        }
    });
}

void Scanner() {
    CameraView(Style{height: 360}, false);
    use_camera_frames(CameraFrameOptions{width: 1280, height: 720, fps: 30}, void(*CameraFrame frame) {
        Mutex_lock(&box.lock);
        if (!box.busy && !box.full) {
            frame.copy_to(&box.copy);
            box.full = true;
            Cond_signal(&box.ready);
        }
        Mutex_unlock(&box.lock);
    });
    use_native_event("detections", void(string result) { /* set a signal */ });
}
```

The web has one thread: there, keep the work small or the size low. On the native hosts the
app's code is compiled at `-O0` under `ui serve`; `ui build` and `ui release` compile it at
`-O2`, so measure a frame's cost there.

## The API

```eskiu
enum CameraFormat { CameraNV12, CameraNV21, CameraI420, CameraYUV420, CameraRGBA }
enum CameraMatrix { CameraBT601, CameraBT709 }

struct CameraPlane { *uint8 data; int row_stride; int pixel_stride; int64 size; }

struct CameraFrame {
    int width; int height; int format; int plane_count; CameraPlane[3] planes;
    int rotation; int64 timestamp_ns; bool front; int matrix; bool full_range;
}

struct CameraFrameOptions { int width = 1280; int height = 720; int fps = 30; }

void use_camera_frames(CameraFrameOptions options, escaping fn(*CameraFrame)->void on_frame);
```

- **Size.** `width` and `height` in the options are a wish, as the sensor sees it
  (landscape). Android picks the smallest `YUV_420_888` size that holds it; iOS the smallest
  session preset (640x480, 1280x720, 1920x1080, 3840x2160) that holds it; the web scales
  the video down to fit it, never up. macOS gives exactly the size asked for: the camera keeps
  its own format (a 1080p camera stays at 1920x1080 under the 1280x720 preset), and the
  output scales its picture to the size, cropping it to that shape (centered) when the shapes
  differ; a size larger than the camera's comes at the camera's size. The frame says what you
  got.
- **Rate.** At most `fps` frames a second: Android sets the auto-exposure range with the
  highest top at or under it, iOS the device's minimum frame duration when its format
  allows that rate, the web its timer. In the dark a camera may give fewer.
- **Layout.** `format` names it and `plane_count` planes are set. `row_stride` is the bytes
  from one row to the next (often more than the width), `pixel_stride` from one sample to the
  next in a row, `size` the bytes readable from `data`. Chroma planes are half the width and
  height, rounded up.

| `format` | Planes | Chroma |
|---|---|---|
| `CameraNV12` | Y, then UV interleaved | `planes[1]`: U at even bytes, V right after; `pixel_stride` 2 |
| `CameraNV21` | Y, then VU interleaved | `planes[1]`: V first, U right after |
| `CameraI420` | Y, U, V | `pixel_stride` 1 |
| `CameraYUV420` | Y, U, V | samples `pixel_stride` bytes apart (Android's general `YUV_420_888`) |
| `CameraRGBA` | RGBA | none; 4 bytes a pixel (the web) |

- **Orientation.** Frames are as the sensor gives them, not turned. `rotation` (0, 90, 180,
  270) is how far to turn the frame clockwise for it to be upright as the screen is held at
  the moment the frame is lent. `front` is the front camera: its frames are what the sensor
  sees, not mirrored the way the live view shows them.
- **Color.** `matrix` (`CameraBT601` or `CameraBT709`) and `full_range` say how the YUV maps
  to RGB: iOS reads the buffer's YCbCr matrix attachment and asks for full range; Android's
  camera YUV is full-range BT.601 (JFIF).
- **Time.** `timestamp_ns` is the platform's capture time on a monotonic clock (iOS the
  sample's presentation time on the host clock, Android `Image.getTimestamp`, the web
  `performance.now`): use it to space frames, not as a wall-clock time.

### Converting to RGBA

`CameraFrame_to_rgba(frame, out, out_stride, upright)` writes RGBA (alpha 255) for CPU
consumers in pure Eskiu, with the frame's own matrix and range, from any of the layouts.
With `upright` it also turns the picture by `rotation`; `out_width(upright)` and
`out_height(upright)` give the size `out` needs (width and height swap for 90 and 270).

The conversion is fixed point (16 fraction bits). `camera_yuv_coeffs(matrix, full_range)` and
`camera_yuv_to_rgb(coeffs, y, u, v, rgb)` convert single pixels; `tests/frames.esk` checks
them against the BT.601 and BT.709 formulas in doubles, full and limited range, within one
step.

### On the GPU

NV12 and NV21 are the layouts GPUs sample directly: the Y plane as an 8-bit single-channel
texture of `width` x `height`, the chroma plane as an 8-bit two-channel texture of half that,
uploaded row by row with `row_stride` as the bytes per row. A shader does the YUV to RGB step
with the frame's `matrix` and `full_range`. I420 and YUV420 are three single-channel
textures.

## Notes and limits

- Frames come from the mounted `CameraView`; with none mounted, `use_camera_frames` gets
  nothing. The view may be small. The camera follows the view (its side, or the device
  `CameraDeviceView` names).
- Turning frames on or off, or changing the options, opens the camera's session again, so
  the live view blinks once.
- On iOS, while frames are on the session runs at their preset, so `camera_capture` photos
  are at most that size.
- On Android the frame stream, the preview and the still capture share one session; a
  frame size over 1920x1080 is not guaranteed alongside them and may give
  `PhotoUnavailable`.
- The simulator has no camera. The Android emulator's virtual camera and a browser's
  camera work.
