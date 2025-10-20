# Smart Treadmill Control System

Convert a weather-damaged Pacemaster Pro Elite into an intelligent training system with real-time running dynamics analysis and workout integration.

## Features

- **Real-time sensor integration**: Stryd power meter, Garmin HRM-600, hall effect speed sensors
- **Smart treadmill control**: GPIO-based button automation with safety interlocks
- **Workout integration**: Garmin Connect API, FIT/TCX file parsing
- **AI running form analysis**: Multi-camera pose estimation and biomechanical feedback (v2.0)
- **Touchscreen interface**: 10.1" display with real-time metrics

## Tech Stack

- **Language**: Rust
- **Hardware**: Raspberry Pi 5, custom sensor array
- **GUI**: egui/eframe
- **AI**: OpenCV, ONNX Runtime, MediaPipe Pose
- **Connectivity**: BLE (btleplug), ANT+, WiFi

## Development Phases

1. **Core System**: BLE sensors, button control, basic GUI
2. **Enhanced Integration**: ANT+ support, data logging, cloud sync
3. **AI Running Dynamics**: Computer vision, pose estimation, form analysis
4. **Advanced AI**: Fatigue prediction, injury risk assessment

## Getting Started

```bash
cargo run
```

See `docs/smart-treadmill-design-v2.md` for detailed architecture and implementation plans.
