# Changelog

All notable changes to this package are documented in this file. The format is based on
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added

- `camera_devices(&list)` lists the cameras (`CameraDevice`: `id`, `name`, `kind`) and
  `CameraDeviceView(style, id)` shows one: on macOS the built-in camera, external (USB) ones,
  Continuity Camera and Desk View; on iOS each lens; on Android each Camera2 id; on the web the
  browser's video inputs. `CameraView(style, front)` is unchanged.
- `CameraFrame_copy_to(frame, &copy)`: an owned copy of a frame (`CameraFrameCopy`, its buffer
  reused) to convert or analyze on a worker thread; docs/frames.md has the pattern.
- macOS: `ui agent`'s screenshot shows the camera (the view gives its latest frame through
  ui's `UiNativeSnapshot`; it was black). The frame for it is kept only under the driver
  (`ui serve`, `ui agent`, `ui test`) when frames are off.

### Fixed

- macOS: `use_camera_frames` asked for 1280x720 got 1920x1080 (the camera keeps its format
  under the session preset): the output scales to the size asked for, cropping to its shape.

### Changed

- The docs say up front that `use_camera_frames`' handler runs on the UI thread and that a
  full-size conversion there freezes the app (worse at `-O0` under `ui serve`).

- The README says which platforms have the camera (the live camera on the Linux and Windows
  desktops only in a `--webview` build), how an unsupported one reports it, and that the
  package is in Preview, tested against stand-ins.

## [0.2.4] - 2026-11-21

### Added

- `camera_supported()`: whether there is a camera to show, before a `CameraView` mounts. False
  on the iOS simulator, a phone without a camera, a browser without `getUserMedia`, a Mac
  without a camera, Linux and Windows. docs/README.md says what each platform does.
- macOS: the photo library through an open panel, and the live `CameraView`, `camera_capture`
  and `use_camera_frames` through AVFoundation. `camera_take_photo` answers `PhotoUnavailable`
  (macOS has no system camera screen).
- Linux and Windows: the photo library through the system's file dialog. The camera is
  unavailable there: `camera_take_photo`, a `CameraView` and `camera_capture` answer
  `PhotoUnavailable`, and `use_camera_frames` gets no frames.
- CI builds the example on the native desktop hosts (`tools/ci_desktop.sh`).

### Fixed

- Native desktop builds of an app using the package link (before, the externs were missing on
  macOS, Linux and Windows).

## [0.2.3]

### Changed

- Depends on ui 0.7.0.

## [0.2.2]

### Changed

- The package moved to the eskiu-packages organization; the old URL redirects. Depends on ui 0.6.1.

## [0.2.1]

### Changed

- Depends on ui 0.6.0.

## [0.2.0]

### Added

- Live frames: `use_camera_frames(CameraFrameOptions, fn(*CameraFrame))` gets the mounted
  `CameraView`'s frames on the UI thread, borrowed by pointer from the platform's buffer for
  the handler's call (no copy, no text message). A frame that arrives while the handler is
  busy replaces the one waiting, so nothing queues. `CameraFrame` has the size, `format`
  (`CameraNV12`, `CameraNV21`, `CameraI420`, `CameraYUV420`, `CameraRGBA`), up to three
  `CameraPlane`s with row and pixel strides and sizes, `rotation`, `timestamp_ns`, `front`,
  `matrix` (`CameraBT601`, `CameraBT709`) and `full_range`. iOS lends the
  `AVCaptureVideoDataOutput`'s NV12 `CVPixelBuffer`, Android a `YUV_420_888` `Image`'s
  planes, the web the `<video>` read back as RGBA. See [docs/frames.md](docs/frames.md).
- `CameraFrame_to_rgba` (optionally turned upright), `CameraFrame_out_width` and
  `out_height`, `camera_yuv_coeffs` and `camera_yuv_to_rgb`: BT.601 and BT.709, full and
  limited range, in pure Eskiu.
- iOS links CoreMedia and CoreVideo.

### Fixed

- iOS: a camera view taken off the screen stops its session before it returns (sessions start
  and stop in order on one queue), so a session started right after, such as AR on the next
  screen, gets the camera at once instead of about nine seconds later.
