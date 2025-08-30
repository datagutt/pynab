# Packet Reference Guide

## Overview

This document provides a comprehensive reference for all packet types used in the Nabd protocol, including detailed field specifications, examples, and Rust type definitions for implementation.

## Packet Categories

1. **Registration Packets** - Service registration and connection management
2. **Information Packets** - Status and capability queries  
3. **State Management** - Rabbit state control
4. **Hardware Control** - Direct hardware commands
5. **Choreography** - Complex animation sequences
6. **Event Packets** - Hardware and system events
7. **Service Communication** - Inter-service messaging
8. **Response Packets** - Command responses and errors

## Rust Type Definitions

### Base Packet Structure

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum NabPacket {
    // Registration
    #[serde(rename = "register")]
    Register {
        service: String,
        #[serde(skip_serializing_if = "Option::is_none")]
        request_id: Option<String>,
        #[serde(skip_serializing_if = "Option::is_none")]
        protocol_version: Option<String>,
    },

    // Information
    #[serde(rename = "info")]
    Info {
        #[serde(skip_serializing_if = "Option::is_none")]
        request_id: Option<String>,
    },

    // State Management
    #[serde(rename = "state")]
    State {
        state: NabState,
        #[serde(skip_serializing_if = "Option::is_none")]
        request_id: Option<String>,
    },

    // Hardware Control - LEDs
    #[serde(rename = "led")]
    Led {
        led: u8,
        color: String,
        #[serde(skip_serializing_if = "Option::is_none")]
        request_id: Option<String>,
    },

    #[serde(rename = "leds")]
    Leds {
        leds: Vec<LedCommand>,
        #[serde(skip_serializing_if = "Option::is_none")]
        request_id: Option<String>,
    },

    // Hardware Control - Ears
    #[serde(rename = "ears")]
    Ears {
        left: i16,
        right: i16,
        #[serde(skip_serializing_if = "Option::is_none")]
        request_id: Option<String>,
    },

    // Hardware Control - Audio
    #[serde(rename = "play_audio")]
    PlayAudio {
        audio_file: String,
        #[serde(skip_serializing_if = "Option::is_none")]
        request_id: Option<String>,
    },

    #[serde(rename = "record_audio")]
    RecordAudio {
        duration: f64,
        #[serde(skip_serializing_if = "Option::is_none")]
        request_id: Option<String>,
    },

    #[serde(rename = "stop_audio")]
    StopAudio {
        #[serde(skip_serializing_if = "Option::is_none")]
        request_id: Option<String>,
    },

    // Choreography
    #[serde(rename = "choreography")]
    Choreography {
        choreography: serde_json::Value,
        #[serde(skip_serializing_if = "Option::is_none")]
        request_id: Option<String>,
    },

    #[serde(rename = "sequence")]
    Sequence {
        sequence: Vec<NabPacket>,
        #[serde(skip_serializing_if = "Option::is_none")]
        request_id: Option<String>,
    },

    // Sleep Management
    #[serde(rename = "sleep")]
    Sleep {
        #[serde(skip_serializing_if = "Option::is_none")]
        request_id: Option<String>,
    },

    #[serde(rename = "wakeup")]
    Wakeup {
        #[serde(skip_serializing_if = "Option::is_none")]
        request_id: Option<String>,
    },

    // Events
    #[serde(rename = "button_event")]
    ButtonEvent {
        event: ButtonEventType,
    },

    #[serde(rename = "ear_event")]
    EarEvent {
        ear: EarSide,
        position: i16,
    },

    #[serde(rename = "rfid_event")]
    RfidEvent {
        uid: String,
        picture: u8,
        #[serde(skip_serializing_if = "Option::is_none")]
        app: Option<String>,
        #[serde(skip_serializing_if = "Option::is_none")]
        data: Option<serde_json::Value>,
    },

    #[serde(rename = "asr_event")]
    AsrEvent {
        text: String,
        confidence: f64,
        #[serde(skip_serializing_if = "Option::is_none")]
        intent: Option<AsrIntent>,
    },

    #[serde(rename = "state_event")]
    StateEvent {
        state: NabState,
        previous_state: NabState,
    },

    // Service Communication
    #[serde(rename = "service_request")]
    ServiceRequest {
        service: String,
        request: serde_json::Value,
        #[serde(skip_serializing_if = "Option::is_none")]
        request_id: Option<String>,
    },

    #[serde(rename = "service_response")]
    ServiceResponse {
        response: serde_json::Value,
        request_id: String,
    },

    // Response
    #[serde(rename = "response")]
    Response {
        status: ResponseStatus,
        request_id: String,
        #[serde(skip_serializing_if = "Option::is_none")]
        info: Option<DaemonInfo>,
        #[serde(skip_serializing_if = "Option::is_none")]
        error: Option<ErrorInfo>,
    },
}

// Supporting types
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "lowercase")]
pub enum NabState {
    Idle,
    Asleep,
    Interactive,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LedCommand {
    pub led: u8,
    pub color: String,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "lowercase")]
pub enum ButtonEventType {
    Down,
    Up,
    Click,
    #[serde(rename = "double_click")]
    DoubleClick,
    #[serde(rename = "long_press")]
    LongPress,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "lowercase")]
pub enum EarSide {
    Left,
    Right,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AsrIntent {
    pub name: String,
    pub parameters: HashMap<String, serde_json::Value>,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "lowercase")]
pub enum ResponseStatus {
    Ok,
    Error(String),
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DaemonInfo {
    pub connections: Vec<ConnectionInfo>,
    pub hardware: HardwareInfo,
    pub state: NabState,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub version: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ConnectionInfo {
    pub service: String,
    pub connected_at: String, // ISO 8601 timestamp
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct HardwareInfo {
    pub model: String,
    pub ears: bool,
    pub leds: u8,
    pub sound_input: bool,
    pub sound_output: bool,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub rfid: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ErrorInfo {
    pub code: String,
    pub message: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub details: Option<String>,
}
```

## Detailed Packet Specifications

### 1. Registration Packets

#### Register
**Purpose**: Register a service with nabd  
**Direction**: Client → Server

```json
{
  "type": "register",
  "service": "service_name",
  "request_id": "unique_id",
  "protocol_version": "1.0"
}
```

**Fields**:
- `service` (required): Service identifier (alphanumeric + underscore)
- `request_id` (optional): Unique request identifier
- `protocol_version` (optional): Protocol version, defaults to "1.0"

**Validation**:
- Service name: 1-64 characters, alphanumeric + underscore only
- Protocol version: Semantic version string (e.g., "1.0", "1.2.3")

**Response**:
```json
{
  "type": "response",
  "status": "ok",
  "request_id": "unique_id"
}
```

**Error Responses**:
- `INVALID_SERVICE_NAME`: Service name format invalid
- `SERVICE_ALREADY_REGISTERED`: Service name already in use
- `PROTOCOL_VERSION_UNSUPPORTED`: Protocol version not supported

### 2. Information Packets

#### Info Request
**Purpose**: Request daemon status and capabilities  
**Direction**: Client → Server

```json
{
  "type": "info",
  "request_id": "unique_id"
}
```

**Response**:
```json
{
  "type": "response",
  "status": "ok",
  "request_id": "unique_id",
  "info": {
    "connections": [
      {
        "service": "nabclockd",
        "connected_at": "2023-12-25T10:30:00Z"
      }
    ],
    "hardware": {
      "model": "nabaztag_v2",
      "ears": true,
      "leds": 5,
      "sound_input": true,
      "sound_output": true,
      "rfid": "st25r391x"
    },
    "state": "idle",
    "version": "1.0.0"
  }
}
```

**Hardware Models**:
- `nabaztag_v1`: Original Nabaztag (no microphone, no RFID)
- `nabaztag_v2`: Nabaztag:tag with microphone and RFID

**RFID Types**:
- `cr14`: Original CR14 I2C RFID reader
- `st25r391x`: Upgraded SPI NFC reader
- `null`: No RFID capability

### 3. State Management

#### State Change
**Purpose**: Change rabbit state  
**Direction**: Client → Server

```json
{
  "type": "state",
  "state": "asleep",
  "request_id": "unique_id"
}
```

**States**:
- `idle`: Normal operation, responsive to interactions
- `asleep`: Low power mode, limited responsiveness
- `interactive`: Actively engaged with user

**State Transitions**:
- `idle` ↔ `asleep`: Sleep/wake transitions
- `idle` → `interactive`: User interaction detected
- `interactive` → `idle`: Interaction timeout

#### State Event
**Purpose**: Notify of state changes  
**Direction**: Server → All Clients

```json
{
  "type": "state_event",
  "state": "asleep",
  "previous_state": "idle"
}
```

### 4. Hardware Control

#### LED Control

**Single LED**:
```json
{
  "type": "led",
  "led": 2,
  "color": "ff0000",
  "request_id": "unique_id"
}
```

**All LEDs**:
```json
{
  "type": "leds",
  "leds": [
    {"led": 0, "color": "ff0000"},
    {"led": 1, "color": "00ff00"},
    {"led": 2, "color": "0000ff"},
    {"led": 3, "color": "ffff00"},
    {"led": 4, "color": "ff00ff"}
  ],
  "request_id": "unique_id"
}
```

**LED Index**: 0-4 (5 LEDs total, circular arrangement)  
**Color Format**: 6-character hex string (RRGGBB)  
**Special Colors**:
- `000000`: Off/black
- `ffffff`: White (maximum brightness)

**Validation**:
- LED index must be 0-4
- Color must be valid 6-character hex
- Commands are queued if hardware is busy

#### Ear Control

```json
{
  "type": "ears",
  "left": 10,
  "right": -5,
  "request_id": "unique_id"
}
```

**Position Range**: -17 to +17
- `-17`: Fully backward
- `0`: Center/neutral position  
- `+17`: Fully forward

**Movement Characteristics**:
- Speed: ~2-3 seconds for full range
- Resolution: 120 steps per position unit
- Accuracy: ±1 position unit

#### Audio Control

**Play Audio File**:
```json
{
  "type": "play_audio",
  "audio_file": "/opt/pynab/nabclockd/sounds/chime.wav",
  "request_id": "unique_id"
}
```

**Record Audio**:
```json
{
  "type": "record_audio",
  "duration": 5.0,
  "request_id": "unique_id"
}
```

**Stop Audio Playback**:
```json
{
  "type": "stop_audio",
  "request_id": "unique_id"
}
```

**Audio Formats Supported**:
- WAV (uncompressed): Primary format
- MP3: Via software decode (mpg123)

**File Path Validation**:
- Must be absolute path
- File must exist and be readable
- Audio format must be supported

### 5. Choreography System

#### Choreography Command
```json
{
  "type": "choreography",
  "choreography": [
    {"tempo": 10, "colors": ["ff0000", "00ff00", "0000ff", "ffff00", "ff00ff"]},
    {"tempo": 5, "left": 8, "right": -8},
    {"tempo": 15, "audio": "/path/to/sound.wav"},
    {"tempo": 20, "colors": ["000000", "000000", "000000", "000000", "000000"]}
  ],
  "request_id": "unique_id"
}
```

**Choreography Step Fields**:
- `tempo`: Duration in 10ms units (1 = 10ms, 10 = 100ms)
- `colors`: Array of 5 LED colors (hex strings)
- `left`/`right`: Ear positions (-17 to +17)
- `audio`: Audio file path for playback

**Execution Model**:
- Steps execute sequentially
- Each step duration defined by `tempo`
- Hardware commands within step execute simultaneously
- Audio playback is non-blocking

#### Command Sequence
```json
{
  "type": "sequence",
  "sequence": [
    {"type": "led", "led": 0, "color": "ff0000"},
    {"type": "ears", "left": 10, "right": -10},
    {"type": "play_audio", "audio_file": "/path/to/sound.wav"}
  ],
  "request_id": "unique_id"
}
```

**Sequence Execution**:
- Commands execute in order
- Each command waits for previous to complete
- Failure of one command aborts sequence
- Use for precise timing requirements

### 6. Event Packets

#### Button Events
```json
{
  "type": "button_event",
  "event": "click"
}
```

**Event Types**:
- `down`: Button pressed down
- `up`: Button released
- `click`: Short press (50ms-500ms)
- `double_click`: Two clicks within 500ms
- `long_press`: Held for >2000ms

#### RFID Events
```json
{
  "type": "rfid_event",
  "uid": "04:52:de:ad:be:ef:42",
  "picture": 255,
  "app": "nabclockd",
  "data": {
    "alarm_time": "07:30",
    "enabled": true
  }
}
```

**Fields**:
- `uid`: Tag unique identifier (colon-separated hex)
- `picture`: Visual picture ID (0-255)
- `app`: Associated service name (optional)
- `data`: Service-specific tag data (optional)

**UID Format**:
- ISO14443A: 4 or 7 bytes, hex with colons
- Example: `04:52:de:ad:be:ef:42`

#### Ear Events
```json
{
  "type": "ear_event",
  "ear": "left",
  "position": 5
}
```

**Triggered When**:
- Ear movement completes
- Position changes due to external force
- Calibration completes

#### ASR Events
```json
{
  "type": "asr_event",
  "text": "set alarm for seven thirty",
  "confidence": 0.85,
  "intent": {
    "name": "set_alarm",
    "parameters": {
      "time": "07:30"
    }
  }
}
```

**Fields**:
- `text`: Raw recognized speech
- `confidence`: Recognition confidence (0.0-1.0)
- `intent`: Parsed intent and parameters (optional)

### 7. Service Communication

#### Service Request
```json
{
  "type": "service_request",
  "service": "nabweatherd",
  "request": {
    "type": "get_forecast",
    "location": "Paris, France",
    "days": 3
  },
  "request_id": "unique_id"
}
```

**Purpose**: Inter-service communication via nabd proxy

**Routing**: nabd forwards request to target service

#### Service Response
```json
{
  "type": "service_response",
  "request_id": "unique_id",
  "response": {
    "type": "forecast_data",
    "forecast": [
      {
        "date": "2023-12-25",
        "temperature": 15,
        "condition": "sunny"
      }
    ]
  }
}
```

### 8. Sleep Management

#### Sleep Command
```json
{
  "type": "sleep",
  "request_id": "unique_id"
}
```

**Effects**:
- Transitions to `asleep` state
- Dims/turns off LEDs
- Moves ears to neutral position
- Reduces system responsiveness

#### Wakeup Command
```json
{
  "type": "wakeup",
  "request_id": "unique_id"
}
```

**Effects**:
- Transitions to `idle` state
- Restores LED brightness
- Re-enables full interaction

### 9. Response Packets

#### Success Response
```json
{
  "type": "response",
  "status": "ok",
  "request_id": "unique_id"
}
```

#### Error Response
```json
{
  "type": "response",
  "status": "error",
  "request_id": "unique_id",
  "error": {
    "code": "HARDWARE_ERROR",
    "message": "LED controller not responding",
    "details": "I2C communication timeout after 5 attempts"
  }
}
```

## Error Codes Reference

### Protocol Errors
- `INVALID_PACKET`: Malformed JSON or missing required fields
- `UNKNOWN_COMMAND`: Unrecognized packet type  
- `MISSING_REQUEST_ID`: Request ID required but not provided
- `PROTOCOL_VERSION_UNSUPPORTED`: Unsupported protocol version

### Hardware Errors
- `HARDWARE_ERROR`: Generic hardware operation failure
- `LED_ERROR`: LED controller error
- `EAR_ERROR`: Ear motor controller error
- `AUDIO_ERROR`: Audio system error
- `RFID_ERROR`: RFID reader error
- `BUTTON_ERROR`: Button input error

### State Errors
- `STATE_ERROR`: Command not valid in current state
- `INVALID_STATE`: Unknown state requested
- `STATE_TRANSITION_DENIED`: State transition not allowed

### Parameter Errors
- `INVALID_PARAMETER`: Parameter value out of range or wrong type
- `MISSING_PARAMETER`: Required parameter not provided
- `PARAMETER_OUT_OF_RANGE`: Numeric parameter exceeds valid range

### Resource Errors
- `RESOURCE_BUSY`: Hardware resource currently in use
- `RESOURCE_UNAVAILABLE`: Requested resource not available
- `QUEUE_FULL`: Command queue is full

### Service Errors
- `SERVICE_NOT_FOUND`: Target service not connected
- `SERVICE_ERROR`: Service-specific error occurred
- `SERVICE_TIMEOUT`: Service did not respond in time

### Audio Errors
- `AUDIO_FILE_NOT_FOUND`: Audio file does not exist
- `AUDIO_FORMAT_UNSUPPORTED`: Audio format not supported
- `AUDIO_DEVICE_ERROR`: Audio hardware error

### RFID Errors
- `RFID_NO_TAG`: No RFID tag detected
- `RFID_READ_ERROR`: Failed to read RFID tag
- `RFID_WRITE_ERROR`: Failed to write RFID tag
- `RFID_TAG_INCOMPATIBLE`: RFID tag type not supported

## Packet Validation Rules

### General Rules
1. All packets must be valid JSON
2. Required fields must be present
3. Field types must match specification
4. String lengths must be within limits
5. Numeric values must be within valid ranges

### Field Limits
- Service names: 1-64 characters, alphanumeric + underscore
- Request IDs: 1-128 characters
- Color strings: Exactly 6 hex characters
- Audio file paths: Absolute paths, max 1024 characters
- Text fields: Max 4096 characters (UTF-8)

### Numeric Ranges
- LED index: 0-4
- Ear position: -17 to +17
- Button event timeout: 0.05-30.0 seconds
- Audio duration: 0.1-300.0 seconds
- Choreography tempo: 1-10000 (10ms to 100s)

## Implementation Notes

### JSON Serialization
- Use standard JSON encoding (UTF-8)
- Preserve field order for debugging
- Handle null values appropriately
- Validate all fields before processing

### Error Handling
- Always provide meaningful error messages
- Include context in error details
- Log errors for debugging
- Return appropriate HTTP-style error codes

### Performance Considerations
- Keep packets small (< 64KB recommended)
- Avoid deep nesting in choreography data
- Use efficient JSON parsing
- Implement proper timeout handling

### Security Considerations
- Validate all file paths for security
- Sanitize user input in text fields
- Limit packet size to prevent DoS
- Rate limit command frequency per service