# Smart Treadmill Control System - Design Document v2

## Project Overview

**Goal:** Convert a weather-damaged Pacemaster Pro Elite into an intelligent training system with real-time running dynamics analysis, workout integration, and belt slip detection.

**Version 1.0:** Basic monitoring and control via button automation
**Version 2.0:** AI-powered running form analysis with multiple camera angles

## System Architecture

### Hardware Components

#### Existing Equipment (Functional)
- **Treadmill:** Pacemaster Pro Elite (commercial grade)
- **Motor:** Pacific Scientific PWM3640-5657-7 (3HP servo motor)
- **Controller:** Aerobics Inc. 9501001 Rev.J (professional PWM controller)
- **Console:** Original button panel (will tap into button contacts)

#### Smart System Hardware
- **Main Computer:** Raspberry Pi 5 (8GB RAM)
- **Display:** Waveshare 10.1" 10-point capacitive touchscreen
- **Speed Sensors:** 2x Hall effect sensors
  - Motor shaft sensor (encoder verification)
  - Rear roller sensor (belt speed measurement)
- **Button Interface:** GPIO-controlled relays/optocouplers (16x)
- **Cameras (v2):** 3x USB3 cameras @ 120fps
  - Side view (sagittal plane)
  - Rear view (frontal plane)
  - Foot strike view (45° angle)

#### Wireless Sensors
- **Power/Pace:** Stryd pod (BLE)
- **Heart Rate:** Polar HRM-600 (BLE/ANT+)
- **Watch:** Garmin Fenix 7s Pro (future ANT+ integration)

## Software Architecture (Rust-based)

### Core Technology Stack
```toml
[dependencies]
# Async runtime
tokio = { version = "1.35", features = ["full"] }

# BLE communication
btleplug = "0.11"
bluez-async = "0.7"

# GPIO control
rppal = "0.17"

# GUI framework
egui = "0.25"
eframe = "0.25"

# Computer vision & AI
opencv = { version = "0.88", features = ["opencv-4", "contrib"] }
ort = "1.16"  # ONNX Runtime for AI inference
candle = "0.4"  # Rust-native neural networks

# Data processing
nalgebra = "0.32"
ndarray = "0.15"

# Workout parsing
fit-file = "0.1"
gpx = "0.8"

# Networking
reqwest = { version = "0.11", features = ["json"] }
tungstenite = "0.21"  # WebSocket for real-time updates

# Serialization
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

### System Control Flow

```
┌──────────────────────────────────────────────────────┐
│                   User Interface                      │
│                  (10.1" Touch Display)                │
└─────────────────┬────────────────────────────────────┘
                  │
┌─────────────────▼────────────────────────────────────┐
│              Main Control Loop (Tokio)                │
│                                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ Workout  │  │  Sensor  │  │  AI Running      │  │
│  │ Manager  │  │  Fusion  │  │  Dynamics (v2)   │  │
│  └──────────┘  └──────────┘  └──────────────────┘  │
└─────────────────┬────────────────────────────────────┘
                  │
     ┌────────────┼────────────┬───────────────┐
     ▼            ▼            ▼               ▼
┌─────────┐ ┌──────────┐ ┌──────────┐  ┌──────────┐
│ Button  │ │   BLE    │ │  Speed   │  │ Cameras  │
│ Control │ │ Sensors  │ │ Sensors  │  │   (v2)   │
└─────────┘ └──────────┘ └──────────┘  └──────────┘
     │           │             │              │
 Treadmill    Stryd +      Hall Effect    USB3 120fps
  Console     HRM-600       Sensors        Cameras
```

## Version 1.0: Core Monitoring & Control

### Data Collection Modules

#### BLE Sensor Manager
```rust
pub struct SensorData {
    // From Stryd
    pub pace: f32,           // min/mile
    pub power: u16,          // watts
    pub cadence: u8,         // steps/min
    pub stride_length: f32,  // meters
    pub vertical_osc: f32,   // cm
    pub ground_time: u16,    // ms
    
    // From HRM-600
    pub heart_rate: u8,      // bpm
    pub hrv: Option<u16>,    // ms
    
    // From Speed Sensors
    pub motor_speed: f32,    // mph
    pub belt_speed: f32,     // mph
    pub slip_percentage: f32,
}
```

#### Button Controller
- Maps GPIO pins to treadmill console buttons
- Implements safety interlocks
- Provides smooth speed/incline ramping
- Emergency stop override

#### Workout Integration
- Garmin Connect API integration
- FIT/TCX file parsing
- Structured workout execution
- Real-time target tracking

### Display Features

#### Main Metrics Grid
- Heart rate with zones
- Pace from Stryd (accurate regardless of incline)
- Power output
- Cadence
- Belt speed & slip detection
- Workout progress

#### Workout Guidance
- Current interval/step display
- Time/distance remaining
- Target zones (HR/Pace/Power)
- Audio/visual cues for transitions

## Version 2.0: AI Running Dynamics

### Computer Vision Pipeline

#### Camera Configuration
```rust
pub struct CameraSetup {
    side_camera: Camera,    // 1920x1080 @ 120fps
    rear_camera: Camera,    // 1920x1080 @ 120fps  
    foot_camera: Camera,    // 1280x720 @ 240fps
}
```

#### Pose Estimation Models
- **Primary:** MediaPipe Pose (33 3D landmarks)
- **Secondary:** OpenPose (25 keypoints)
- **Specialized:** Custom foot strike model (ONNX)

### Running Metrics Analysis

#### Biomechanical Measurements
```rust
pub struct RunningDynamics {
    // Sagittal Plane (Side View)
    pub forward_lean: f32,        // degrees
    pub vertical_oscillation: f32, // cm
    pub stride_length: f32,        // meters
    pub stride_angle: f32,         // degrees at max extension
    
    // Frontal Plane (Rear View)
    pub hip_drop: f32,            // degrees
    pub knee_valgus: f32,         // degrees
    pub step_width: f32,          // cm
    
    // Foot Strike Analysis
    pub strike_type: StrikeType,  // Heel/Midfoot/Forefoot
    pub contact_time: u16,        // ms
    pub pronation_angle: f32,     // degrees
    pub strike_position: f32,     // cm relative to COM
}

pub enum StrikeType {
    HeelStrike,
    MidfootStrike,
    ForefootStrike,
}
```

#### Real-time Feedback System
```rust
pub struct FormCoach {
    pub corrections: Vec<FormCorrection>,
    pub efficiency_score: f32,
    pub injury_risk_flags: Vec<RiskFactor>,
}

pub struct FormCorrection {
    pub metric: String,
    pub current_value: f32,
    pub target_value: f32,
    pub cue: String,
    pub priority: Priority,
}
```

### AI Model Architecture

#### Pose Estimation Pipeline
1. **Frame Capture** (120-240 fps)
2. **Preprocessing** (resize, normalize)
3. **Inference** (ONNX Runtime on Pi 5)
4. **Landmark Extraction** (3D positions)
5. **Temporal Smoothing** (Kalman filter)
6. **Metric Calculation** (biomechanical angles)

#### Custom Models
- Foot strike classifier (CNN)
- Running efficiency predictor (RNN)
- Fatigue detection (LSTM)
- Injury risk assessment (ensemble)

## Development Phases

### Phase 1: Core System (Current)
**Timeline:** 3-4 weeks
- [x] Raspberry Pi 5 setup with touchscreen
- [ ] BLE sensor integration (Stryd + HRM-600)
- [ ] Hall effect speed sensors
- [ ] Button control interface
- [ ] Basic GUI with metrics display
- [ ] Garmin workout import

**Deliverables:**
- Working display showing all sensor data
- Button control of treadmill
- Belt slip detection
- Basic workout following

### Phase 2: Enhanced Integration
**Timeline:** 2-3 weeks
- [ ] ANT+ integration for Fenix watch
- [ ] Advanced workout modes
- [ ] Data logging and export
- [ ] Web dashboard
- [ ] Strava/Garmin Connect upload

**Deliverables:**
- Full ecosystem integration
- Historical data analysis
- Cloud sync capability

### Phase 3: AI Running Dynamics (v2)
**Timeline:** 6-8 weeks
- [ ] Camera mounting system design
- [ ] OpenCV integration
- [ ] Pose estimation implementation
- [ ] Custom model training
- [ ] Real-time feedback system

**Deliverables:**
- 3-camera setup
- Real-time form analysis
- Personalized coaching cues
- Form improvement tracking

### Phase 4: Advanced AI Features
**Timeline:** 4-6 weeks
- [ ] Fatigue prediction
- [ ] Injury risk assessment
- [ ] Gait pattern learning
- [ ] Performance optimization
- [ ] Virtual coach personality

**Deliverables:**
- Predictive analytics
- Personalized training plans
- Long-term progress tracking

## Technical Requirements

### Performance Targets
- **Sensor Update Rate:** 10Hz minimum
- **Display Refresh:** 60 fps
- **Button Response:** <50ms latency
- **BLE Latency:** <100ms
- **Pose Estimation:** 30 fps minimum
- **AI Inference:** <33ms per frame

### Safety Requirements
- **Emergency Stop:** Hardware interrupt, <100ms
- **Speed Limits:** User-configurable
- **Slip Detection:** Auto-stop at >10% slip
- **Thermal Monitoring:** CPU/GPU temperature limits
- **Failsafe Mode:** Manual operation fallback

### System Resources (Pi 5)
- **CPU Usage:** <60% average
- **RAM Usage:** <4GB for v1, <6GB for v2
- **Storage:** 32GB minimum (128GB for v2)
- **GPU:** VideoCore VII for AI inference
- **Network:** WiFi for cloud sync

## Data Architecture

### Local Storage
```rust
pub struct TrainingSession {
    pub id: Uuid,
    pub timestamp: DateTime<Local>,
    pub workout: Option<Workout>,
    pub sensor_data: Vec<SensorData>,
    pub dynamics_data: Option<Vec<RunningDynamics>>, // v2
    pub video_clips: Option<Vec<VideoSegment>>,      // v2
    pub summary: SessionSummary,
}
```

### Cloud Sync
- SQLite for local storage
- PostgreSQL for cloud database
- S3-compatible storage for videos
- GraphQL API for data access

## Hardware BOM

### Version 1.0
- Raspberry Pi 5 (8GB): $80
- 10.1" Touchscreen: Existing
- 2x Hall Effect Sensors: $20
- Magnets & Mounting: $10
- Relay Board (16ch): $25
- Wire & Connectors: $15
- Power Supply: $20
**Phase 1 Total: ~$170**

### Version 2.0 Additions
- 3x USB3 Cameras: $150
- Camera Mounts: $50
- USB3 Hub: $30
- Additional Storage: $40
- Cooling System: $25
**Phase 3 Total: ~$295**

**Project Total: ~$465**

## Risk Assessment & Mitigation

### Technical Risks
| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| BLE Connection Drops | Medium | Low | Implement reconnection logic |
| Button Control Reliability | Low | High | Use optical isolation, test extensively |
| Belt Slip False Positives | Medium | Medium | Calibrate thresholds, use averaging |
| AI Inference Too Slow | Medium | Medium | Use GPU acceleration, optimize models |
| Camera Vibration | High | Medium | Design dampened mounts |

### Safety Risks
| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| Emergency Stop Failure | Low | Critical | Hardware interrupt, multiple paths |
| Unintended Speed Change | Low | High | Rate limiting, confirmation |
| User Distraction | Medium | Medium | Clear UI, audio cues |

## Success Metrics

### Version 1.0
- [ ] All sensors reporting accurately
- [ ] <1% packet loss on BLE
- [ ] Belt slip detection ±2% accuracy  
- [ ] Workout execution without manual intervention
- [ ] 10+ hour continuous operation

### Version 2.0
- [ ] Pose estimation at 30+ fps
- [ ] Running metrics ±5% vs. lab measurement
- [ ] Form feedback within 2 seconds
- [ ] 90%+ user satisfaction with coaching
- [ ] Measurable form improvement over time

---

**Next Action:** Implement Rust-based BLE sensor integration starting with Stryd pod connection.