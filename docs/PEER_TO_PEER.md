# Peer-to-Peer Messaging Feature

## Summary

Added secure peer-to-peer messaging capability allowing registered clients with permanent IDs to communicate directly with each other.

## Implementation Details

### New Message Type
- `CAN_PS_PEER_MSG (0x09)` - Dedicated message type for client-to-client communication

### Security Model
- **Permanent IDs (1-100)**: Can send and receive peer messages
- **Temporary IDs (101+)**: Cannot participate in peer messaging
- Both sender and receiver validated by broker before forwarding

### Deduplication
- Prevents duplicate peer messages within 50ms window
- Tracks last sender ID and message content
- Automatic on both client and broker sides

### Self-Message Detection
- Clients can detect when they receive their own messages
- Special handling in `onDirectMessage()` callback
- Useful for testing and verification

### Architecture
```
Client A (Permanent ID)  →  Broker (Validates & Forwards)  →  Client B (Permanent ID)
```

### API Additions

#### Client Class
```cpp
// Send peer-to-peer message
bool sendPeerMessage(uint8_t targetClientId, const String& message)

// Receive via existing callback
void onDirectMessage(DirectMessageCallback callback)
```

#### Message Handling
- Standard frames (≤8 bytes)
- Extended frames (>8 bytes) automatically
- Broker validates sender/receiver have permanent IDs
- Messages forwarded transparently

## Files Modified

### Source Code
- `CANPubSub.h` - Added CAN_PS_PEER_MSG, sendPeerMessage(), handlePeerMessageReceived()
- `CANPubSub.cpp` - Implemented broker and client handlers for peer messaging

### Documentation
- `PUBSUB_API.md` - Added sendPeerMessage() API documentation
- `SERIAL_NUMBER_MANAGEMENT.md` - Updated with ID ranges and peer messaging benefits
- `QUICK_REFERENCE.md` - Added peer messaging quick reference

### Examples
- `PeerToPeer/PeerToPeer.ino` - NEW: Dedicated peer messaging example
- `PeerToPeer/README.md` - NEW: Example documentation
- `ClientWithSerial/ClientWithSerial.ino` - Added unified msg:id:text command

## Usage Example

### Programmatic API
```cpp
// Setup
CANPubSubClient client(CAN);
client.begin("ESP32_ABC123");  // Get permanent ID

// Send to broker (ID 0)
client.sendDirectMessage("status request");

// Send to peer (ID 2)
client.sendPeerMessage(2, "Hello from client 1!");

// Receive messages (both broker and peers)
client.onDirectMessage([](uint8_t senderId, const String& msg) {
  if (senderId == 0) {
    Serial.println("From broker: " + msg);
  } else {
    Serial.println("From peer " + String(senderId) + ": " + msg);
  }
});
```

### Serial Command Interface
When using `ClientWithSerial` example, use the unified `msg:` command:
```
msg:0:text     → Send to broker (ID 0)
msg:2:hello    → Send peer message to client ID 2
msg:5:data     → Send peer message to client ID 5
```

**Self-Message Detection:**
```cpp
client.onDirectMessage([](uint8_t senderId, const String& msg) {
  if (senderId == client.getClientId()) {
    Serial.println("Received my own message!");  // Self-message
  } else if (senderId == 0) {
    Serial.println("From broker: " + msg);
  } else {
    Serial.println("From peer " + String(senderId) + ": " + msg);
  }
});
```

## Key Benefits

1. **Secure**: Only registered clients can participate
2. **Transparent**: Uses existing onDirectMessage() callback
3. **Flexible**: Supports standard and extended messages
4. **Validated**: Broker ensures both endpoints have permanent IDs
5. **Easy**: Simple API - just provide target ID and message
6. **Reliable**: Automatic message deduplication within 50ms window
7. **Smart**: Self-message detection for testing and verification

## Limitations

- Requires both clients to have permanent IDs (registered with serial numbers)
- No delivery confirmation (fire-and-forget)
- Broker must be running to forward messages
- Maximum message length: ~120 bytes

## Testing

To test peer messaging:

1. Start broker: Upload `BrokerWithSerial.ino`
2. Start client A: Upload `PeerToPeer.ino` with `MY_SERIAL = "DEVICE_001"`
3. Start client B: Upload `PeerToPeer.ino` with `MY_SERIAL = "DEVICE_002"`
4. Client A will get ID 1, Client B will get ID 2
5. Messages will flow: A → Broker → B

## Backward Compatibility

- Existing clients continue to work unchanged
- Temporary clients (without serial numbers) remain functional for pub/sub
- Only new peer messaging feature requires permanent IDs
- No breaking changes to existing API
