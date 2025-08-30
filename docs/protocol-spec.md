# Nabd Protocol Specification

## Overview

The Nabd protocol is a JSON-based TCP protocol running on port 10543 that enables communication between the core daemon (nabd) and various services. This specification defines the complete protocol for Rust implementation compatibility.

## Connection Model

### TCP Server
- **Port**: 10543 (configurable)
- **Address**: 127.0.0.1 (local only)
- **Connection Type**: Persistent TCP connections
- **Encoding**: UTF-8 JSON messages
- **Message Delimiter**: Newline character (`\n`)

### Client Connection Lifecycle
1. **Connect** - TCP connection established
2. **Register** - Service sends registration packet
3. **Active** - Bidirectional message exchange
4. **Disconnect** - Connection closed (graceful or abrupt)

## Message Format

All messages are JSON objects terminated by a newline character.

### Base Message Structure
```json
{
  "type": "packet_type",
  "request_id": "optional_request_id",
  ...additional_fields
}
```

### Message Types
- **Commands**: Client → Server requests
- **Responses**: Server → Client responses to commands
- **Events**: Server → Client notifications
- **Info**: Bidirectional status/capability messages

## Packet Types

### 1. Registration Packets

#### Service Registration
**Direction**: Client → Server  
**Purpose**: Register a service with nabd

```json
{
  "type": "register",
  "service": "service_name",
  "request_id": "unique_id"
}
```

**Response**:
```json
{
  "type": "response",
  "request_id": "unique_id",
  "status": "ok"
}
```

### 2. Information Packets

#### Info Request
**Direction**: Client → Server  
**Purpose**: Request daemon status and capabilities

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
  "request_id": "unique_id",
  "status": "ok",
  "info": {
    "connections": [
      {
        "service": "service_name",
        "connected_at": "2023-01-01T12:00:00Z"
      }
    ],
    "hardware": {
      "model": "nabaztag_v1",
      "ears": true,
      "leds": 5,
      "sound_input": true,
      "sound_output": true,
      "rfid": "cr14"
    },
    "state": "idle"
  }
}
```

### 3. State Management

#### State Change Command
**Direction**: Client → Server  
**Purpose**: Request state change

```json
{
  "type": "state",
  "state": "idle|asleep|interactive",
  "request_id": "unique_id"
}
```

#### State Event
**Direction**: Server → Client  
**Purpose**: Notify of state changes

```json
{
  "type": "state_event",
  "state": "idle",
  "previous_state": "asleep"
}
```

### 4. Hardware Control

#### LED Commands
```json
{
  "type": "led",
  "led": 0,
  "color": "ff0000",
  "request_id": "unique_id"
}
```

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

#### Ear Commands
```json
{
  "type": "ears",
  "left": 0,
  "right": 10,
  "request_id": "unique_id"
}
```

Position range: -17 to 17 (0 = center, negative = backward, positive = forward)

#### Audio Commands

**Play Audio**:
```json
{
  "type": "play_audio",
  "audio_file": "/path/to/audio.wav",
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

**Stop Audio**:
```json
{
  "type": "stop_audio",
  "request_id": "unique_id"
}
```

### 5. Choreography System

#### Choreography Command
```json
{
  "type": "choreography",
  "choreography": "choreography_data",
  "request_id": "unique_id"
}
```

#### Choreography Format
Choreographies are JSON arrays of timed actions:
```json
[
  {"tempo": 10, "colors": ["ff0000", "00ff00", "0000ff", "ffff00", "ff00ff"]},
  {"tempo": 5, "left": 5, "right": -5},
  {"tempo": 15, "audio": "/path/to/sound.wav"},
  {"tempo": 20, "colors": ["000000", "000000", "000000", "000000", "000000"]}
]
```

### 6. Event Packets

#### Button Event
**Direction**: Server → Clients  
**Purpose**: Notify of button press/release

```json
{
  "type": "button_event",
  "event": "down|up|click|double_click|long_press"
}
```

#### Ear Event
**Direction**: Server → Clients  
**Purpose**: Notify of ear position changes

```json
{
  "type": "ear_event",
  "ear": "left|right",
  "position": 5
}
```

#### RFID Event
**Direction**: Server → Clients  
**Purpose**: Notify of RFID tag detection

```json
{
  "type": "rfid_event",
  "uid": "04:52:de:ad:be:ef:42",
  "picture": 255,
  "app": "service_name",
  "data": "tag_specific_data"
}
```

#### ASR Event (Automatic Speech Recognition)
**Direction**: Server → Clients  
**Purpose**: Notify of speech recognition results

```json
{
  "type": "asr_event",
  "text": "recognized text",
  "confidence": 0.85,
  "intent": {
    "name": "intent_name",
    "parameters": {"param1": "value1"}
  }
}
```

### 7. Service Interaction

#### Service Request
**Direction**: Client → Server → Target Client  
**Purpose**: Inter-service communication

```json
{
  "type": "service_request",
  "service": "target_service",
  "request": {
    "type": "custom_request",
    "data": "request_specific_data"
  },
  "request_id": "unique_id"
}
```

#### Service Response
**Direction**: Target Client → Server → Original Client

```json
{
  "type": "service_response",
  "request_id": "unique_id",
  "response": {
    "type": "custom_response",
    "data": "response_specific_data"
  }
}
```

### 8. Command Sequencing

#### Command Sequence
**Direction**: Client → Server  
**Purpose**: Execute multiple commands in sequence

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

### 9. Sleep Management

#### Sleep Command
**Direction**: Client → Server  
**Purpose**: Put rabbit to sleep

```json
{
  "type": "sleep",
  "request_id": "unique_id"
}
```

#### Wakeup Command
**Direction**: Client → Server  
**Purpose**: Wake rabbit up

```json
{
  "type": "wakeup",
  "request_id": "unique_id"
}
```

### 10. Error Handling

#### Error Response
**Direction**: Server → Client  
**Purpose**: Report command errors

```json
{
  "type": "response",
  "request_id": "unique_id",
  "status": "error",
  "error": {
    "code": "ERROR_CODE",
    "message": "Human readable error message",
    "details": "Additional error details"
  }
}
```

**Error Codes**:
- `INVALID_PACKET`: Malformed JSON or missing required fields
- `UNKNOWN_COMMAND`: Unrecognized packet type
- `HARDWARE_ERROR`: Hardware operation failed
- `STATE_ERROR`: Command not valid in current state
- `RESOURCE_BUSY`: Hardware resource is busy
- `INVALID_PARAMETER`: Parameter value out of range
- `SERVICE_NOT_FOUND`: Target service not connected
- `TIMEOUT`: Operation timed out

## Protocol State Machine

### Connection States
1. **DISCONNECTED** - No connection
2. **CONNECTED** - TCP connection established
3. **REGISTERED** - Service registered with nabd
4. **ACTIVE** - Normal operation mode

### State Transitions
```
DISCONNECTED --[connect]--> CONNECTED
CONNECTED --[register]--> REGISTERED  
REGISTERED --[disconnect]--> DISCONNECTED
REGISTERED --> ACTIVE (after successful registration)
ACTIVE --[disconnect]--> DISCONNECTED
```

## Message Flow Examples

### Service Registration Flow
1. Service connects to nabd on port 10543
2. Service sends registration packet
3. Nabd responds with success/failure
4. Service is now registered and can receive events

### Hardware Control Flow
1. Service sends LED command to nabd
2. Nabd validates command and current state
3. Nabd executes hardware operation
4. Nabd sends response to service

### Event Broadcasting Flow
1. Hardware event occurs (button press, RFID detection, etc.)
2. Nabd creates event packet
3. Nabd broadcasts event to all registered services
4. Services process event independently

## Timing and Performance

### Message Limits
- **Maximum message size**: 64KB
- **Maximum queue depth**: 1000 messages per connection
- **Connection timeout**: 30 seconds for registration
- **Heartbeat interval**: 60 seconds (optional)

### Performance Requirements
- **Command response time**: < 100ms for hardware commands
- **Event broadcast time**: < 50ms to all services
- **Maximum concurrent connections**: 20 services

## Protocol Versioning

### Version Negotiation
Services can specify protocol version during registration:

```json
{
  "type": "register",
  "service": "service_name",
  "protocol_version": "1.0",
  "request_id": "unique_id"
}
```

### Backward Compatibility
- Version 1.0: Current protocol (as specified)
- Future versions must maintain backward compatibility
- New fields can be added (ignored by older implementations)
- Existing field semantics cannot change

## Security Considerations

### Access Control
- Local connections only (127.0.0.1)
- No authentication required (local trust model)
- Services run under same user account

### Data Validation
- All JSON must be valid and well-formed
- Required fields must be present
- Parameter ranges must be validated
- String lengths must be bounded

### Resource Limits
- Connection limits prevent DoS
- Message queue limits prevent memory exhaustion
- Command rate limiting may be implemented

## Implementation Notes

### JSON Serialization
- Use standard JSON encoding
- Numbers: integers for discrete values, floats for continuous
- Colors: 6-character hex strings (RRGGBB)
- Timestamps: ISO 8601 format
- Binary data: Base64 encoding

### Connection Handling
- TCP_NODELAY should be enabled
- Keep-alive may be used
- Graceful shutdown on SIGTERM
- Proper cleanup of resources on disconnect

### Error Recovery
- Clients should reconnect on connection loss
- Commands may be retried on timeout
- Hardware errors should be logged and reported
- State inconsistencies should trigger resync

### Threading Model
- Async/await preferred for I/O operations
- Hardware operations may require dedicated threads
- Message queues for inter-thread communication
- Proper synchronization for shared state

## Testing Requirements

### Protocol Compliance Tests
- Message format validation
- State transition validation
- Error condition handling
- Performance requirements

### Integration Tests
- Multi-service communication
- Hardware operation sequences
- Event broadcasting
- Connection lifecycle

### Load Tests
- Maximum connection handling
- Message throughput
- Memory usage under load
- Recovery from overload conditions