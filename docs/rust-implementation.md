# Rust Implementation Plan for Pynab

## Overview

This document provides a comprehensive implementation plan for rewriting Pynab's core services in Rust while maintaining 100% compatibility with the existing Python ecosystem.

## Goals

- **Performance**: Reduce CPU usage and memory footprint by 50-80%
- **Reliability**: Eliminate runtime errors through Rust's type system
- **Compatibility**: Maintain full protocol and API compatibility
- **Maintainability**: Create a more structured, testable codebase

## Architecture Overview

### Core Components

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   nabd-rust     │    │   nabboot-rust  │    │ nabclockd-rust  │
│   (TCP Server)  │◄──►│   (Startup)     │◄──►│   (Time/Sleep)  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                        │                        │
         ▼                        ▼                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Hardware Abstraction Layer                    │
│                          (nabio-rust)                           │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Hardware Drivers                             │
│        (LEDs, Ears, Audio, RFID, Button)                       │
└─────────────────────────────────────────────────────────────────┘
```

## Project Structure

```
pynab-rust/
├── Cargo.toml
├── crates/
│   ├── nabio/                 # Hardware abstraction layer
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── led.rs         # WS2812B LED control
│   │   │   ├── ears.rs        # Stepper motor ear control  
│   │   │   ├── audio.rs       # ALSA audio interface
│   │   │   ├── rfid.rs        # RFID reader interface
│   │   │   └── button.rs      # GPIO button interface
│   │   └── Cargo.toml
│   │
│   ├── nabprotocol/           # Protocol definitions
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── packets.rs     # Packet type definitions
│   │   │   ├── codec.rs       # JSON serialization
│   │   │   └── client.rs      # Protocol client
│   │   └── Cargo.toml
│   │
│   ├── nabservice/            # Service framework
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── base.rs        # Base service traits
│   │   │   ├── recurrent.rs   # Recurrent service trait
│   │   │   └── config.rs      # Configuration management
│   │   └── Cargo.toml
│   │
│   ├── nabd/                  # Core daemon
│   │   ├── src/
│   │   │   ├── main.rs
│   │   │   ├── server.rs      # TCP server
│   │   │   ├── state.rs       # State machine
│   │   │   ├── choreography.rs # Choreography engine
│   │   │   └── hardware.rs    # Hardware coordination
│   │   └── Cargo.toml
│   │
│   ├── nabboot/               # Boot service
│   │   ├── src/
│   │   │   ├── main.rs
│   │   │   └── service.rs
│   │   └── Cargo.toml
│   │
│   └── nabclockd/             # Clock service
│       ├── src/
│       │   ├── main.rs
│       │   └── service.rs
│       └── Cargo.toml
│
├── scripts/
│   ├── build.sh               # Build script
│   ├── install.sh             # Installation script
│   └── migrate.sh             # Migration script
│
└── tests/
    ├── integration/           # Integration tests
    └── compatibility/         # Compatibility tests
```

## Implementation Phases

### Phase 1: Core Infrastructure (4-6 weeks)
1. **Hardware Abstraction Layer (nabio)**
   - LED control with WS2812B driver
   - Stepper motor ear control
   - ALSA audio interface
   - GPIO button handling
   - RFID reader interfaces (CR14, ST25R391X)

2. **Protocol Layer (nabprotocol)**
   - JSON packet serialization/deserialization
   - TCP client/server framework
   - Packet type definitions
   - Error handling

3. **Service Framework (nabservice)**
   - Base service trait
   - Recurrent service trait
   - Configuration management
   - Logging integration

### Phase 2: Core Daemon (6-8 weeks)
1. **State Machine**
   - Idle, asleep, interactive states
   - State transition logic
   - Event handling

2. **TCP Server**
   - Multi-client connection handling
   - Request/response routing
   - Service registration

3. **Choreography Engine**
   - Choreography parsing and execution
   - LED animation system
   - Audio synchronization

4. **Hardware Coordination**
   - Hardware resource management
   - Concurrent access control
   - Error recovery

### Phase 3: Essential Services (4-6 weeks)
1. **nabboot** - System startup and initialization
2. **nabclockd** - Time management and sleep scheduling
3. **nabtaichid** - Random choreography service

### Phase 4: Testing and Integration (4-6 weeks)
1. **Unit Tests** - Comprehensive test coverage
2. **Integration Tests** - Service interaction testing
3. **Compatibility Tests** - Python service compatibility
4. **Performance Tests** - Benchmarking and optimization

## Key Design Decisions

### Memory Management
- Use `Arc<Mutex<T>>` for shared state
- Prefer `tokio::sync::RwLock` for high-read scenarios
- Use channels for inter-task communication
- Implement custom memory pools for audio buffers

### Error Handling
```rust
#[derive(thiserror::Error, Debug)]
pub enum NabError {
    #[error("Hardware error: {0}")]
    Hardware(String),
    
    #[error("Protocol error: {0}")]
    Protocol(String),
    
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
    
    #[error("Serialization error: {0}")]
    Serialization(#[from] serde_json::Error),
}

pub type Result<T> = std::result::Result<T, NabError>;
```

### Async Architecture
- Use `tokio` for async runtime
- Implement async traits for hardware interfaces
- Use channels for message passing between services
- Handle backpressure with bounded channels

### Configuration Management
```rust
#[derive(Deserialize, Debug)]
pub struct NabConfig {
    pub server: ServerConfig,
    pub hardware: HardwareConfig,
    pub services: HashMap<String, ServiceConfig>,
}

impl NabConfig {
    pub fn load() -> Result<Self> {
        // Load from /opt/pynab/venv/lib/python3.9/site-packages/nabd/nabd.ini
    }
}
```

## Hardware Interface Specifications

### LED Control
```rust
pub trait LedController: Send + Sync {
    async fn set_color(&self, led: u8, color: Rgb) -> Result<()>;
    async fn set_all(&self, colors: &[Rgb; 5]) -> Result<()>;
    async fn pulse(&self, led: u8, color: Rgb, duration: Duration) -> Result<()>;
}
```

### Ear Control
```rust
pub trait EarController: Send + Sync {
    async fn move_to(&self, ear: Ear, position: i16) -> Result<()>;
    async fn get_position(&self, ear: Ear) -> Result<i16>;
    async fn calibrate(&self, ear: Ear) -> Result<()>;
}
```

### Audio Control
```rust
pub trait AudioController: Send + Sync {
    async fn play(&self, audio: AudioData) -> Result<PlaybackHandle>;
    async fn record(&self, duration: Duration) -> Result<AudioData>;
    async fn set_volume(&self, volume: u8) -> Result<()>;
}
```

## Protocol Implementation

### Packet Types
```rust
#[derive(Serialize, Deserialize, Debug, Clone)]
#[serde(tag = "type")]
pub enum NabPacket {
    #[serde(rename = "info")]
    Info { request_id: Option<String> },
    
    #[serde(rename = "state")]
    State { state: NabState },
    
    #[serde(rename = "ears")]
    Ears { left: i16, right: i16 },
    
    #[serde(rename = "led")]
    Led { led: u8, color: String },
    
    // ... other packet types
}
```

### Service Communication
```rust
#[async_trait]
pub trait NabService: Send + Sync {
    async fn start(&self) -> Result<()>;
    async fn stop(&self) -> Result<()>;
    async fn handle_packet(&self, packet: NabPacket) -> Result<Option<NabPacket>>;
    async fn get_info(&self) -> Result<ServiceInfo>;
}
```

## Performance Targets

| Metric | Current (Python) | Target (Rust) | Improvement |
|--------|------------------|---------------|-------------|
| Memory Usage | ~80MB | ~20MB | 75% reduction |
| CPU Usage (idle) | ~5% | ~1% | 80% reduction |
| Startup Time | ~10s | ~2s | 80% reduction |
| Response Latency | ~50ms | ~5ms | 90% reduction |
| Audio Latency | ~200ms | ~50ms | 75% reduction |

## Dependencies

### Core Dependencies
```toml
[dependencies]
tokio = { version = "1.0", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
thiserror = "1.0"
anyhow = "1.0"
tracing = "0.1"
tracing-subscriber = "0.3"
config = "0.13"
async-trait = "0.1"
```

### Hardware Dependencies
```toml
alsa = "0.7"           # ALSA audio interface
rppal = "0.14"         # Raspberry Pi GPIO
smart-leds = "0.3"     # WS2812B LED control
embedded-hal = "0.2"   # Hardware abstraction
```

## Build System

### Cross Compilation
```bash
# Install cross compilation target
rustup target add armv7-unknown-linux-gnueabihf

# Build for Raspberry Pi
cargo build --target armv7-unknown-linux-gnueabihf --release
```

### Packaging
```bash
# Create debian package
cargo deb --target armv7-unknown-linux-gnueabihf
```

## Testing Strategy

### Unit Tests
- Test each hardware interface independently
- Mock hardware for CI/CD pipeline
- Test packet serialization/deserialization
- Test state machine transitions

### Integration Tests
- Test service communication
- Test hardware coordination
- Test choreography execution
- Test error recovery

### Compatibility Tests
- Test with existing Python services
- Test protocol compatibility
- Test configuration compatibility
- Test migration scenarios

### Performance Tests
- Benchmark hardware operations
- Measure memory usage
- Test under load conditions
- Validate latency requirements

## Migration Strategy

1. **Side-by-side deployment** - Run Rust services alongside Python
2. **Gradual replacement** - Replace services one at a time
3. **Compatibility validation** - Continuous testing during migration
4. **Rollback procedures** - Quick rollback if issues arise

## Risk Mitigation

### Technical Risks
- **Hardware compatibility**: Extensive testing on target hardware
- **Protocol compatibility**: Automated compatibility testing
- **Performance regression**: Continuous benchmarking

### Operational Risks
- **Migration complexity**: Phased rollout with rollback plans
- **Service disruption**: Blue-green deployment strategy
- **User impact**: Maintain existing functionality during transition

## Success Metrics

- **Performance**: Meet all performance targets
- **Reliability**: 99.9% uptime during migration
- **Compatibility**: 100% protocol compatibility
- **Quality**: 90%+ test coverage
- **Maintainability**: Reduced complexity metrics