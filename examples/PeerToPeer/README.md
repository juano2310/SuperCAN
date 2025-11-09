# Peer-to-Peer Messaging Example

This example demonstrates direct client-to-client messaging using the SuperCAN library.

## Overview

Peer-to-peer messaging allows registered clients (those with serial numbers and permanent IDs) to send messages directly to each other without going through the broker's callback system.

## Features

- ✅ Direct client-to-client communication
- ✅ Security: Only clients with permanent IDs (1-100) can participate
- ✅ Automatic broker forwarding and validation
- ✅ Support for both standard and extended messages

## Requirements

- **Serial Number Required**: Both sender and receiver must be registered with serial numbers
- **Permanent ID Range**: Client IDs must be in range 1-100 (not temporary IDs 101+)
- **Broker**: Must be running on the CAN bus to forward messages

## How It Works

```
Client A (ID 1)                  Broker                    Client B (ID 2)
      |                            |                             |
      |  PEER_MSG [1→2]: "Hello"  |                              |
      |--------------------------->|                             |
      |                            | Validates sender has ID < 101
      |                            | Validates target has ID < 101
      |                            |  PEER_MSG [1→2]: "Hello"    |
      |                            |---------------------------->|
      |                            |                             |
```

## Usage

### Sending a Peer Message

```cpp
uint8_t targetClientId = 2;
String message = "Hello from peer!";

if (client.sendPeerMessage(targetClientId, message)) {
  Serial.println("Message sent successfully");
} else {
  Serial.println("Failed - check both have permanent IDs");
}
```

### Receiving Peer Messages

Peer messages are received via the `onDirectMessage()` callback:

```cpp
client.onDirectMessage([](uint8_t senderId, const String& msg) {
  if (senderId == 0) {
    Serial.println("Message from broker: " + msg);
  } else {
    Serial.print("Peer message from client ");
    Serial.print(senderId);
    Serial.print(": ");
    Serial.println(msg);
  }
});
```

## Configuration

Change these values for each device:

```cpp
const String MY_SERIAL = "DEVICE_001";  // Unique serial number
const uint8_t TARGET_CLIENT_ID = 2;     // ID of peer to message
```

## Hardware Setup

### ESP32 with Built-in CAN

No additional hardware required - uses internal CAN controller.

### Arduino with MCP2515

Connect MCP2515 module:
- CS → Pin 5
- INT → Pin 4
- SCK → Pin 18
- MISO → Pin 19
- MOSI → Pin 23

## Running the Example

1. **Start the broker** on one device
2. **Upload this sketch** to two or more ESP32/Arduino boards
3. **Change `MY_SERIAL`** to be unique for each device
4. **Adjust `TARGET_CLIENT_ID`** to target different clients
5. **Monitor Serial output** to see messages

## Expected Output

```
=== SuperCAN Peer-to-Peer Messaging Example ===
My Serial Number: DEVICE_001

CAN bus initialized at 500kbps
Connecting to broker...
Connected! Client ID: 1

Peer-to-peer messaging enabled!
Only clients with permanent IDs (1-100) can use this feature.
Temporary clients (101+) cannot send or receive peer messages.

Sending peer message to client 2: Hello from DEVICE_001 at 5234
✓ Message sent successfully

--- Peer Message Received ---
From Client ID: 2
Message: Thanks for your message!
-----------------------------
```

## Limitations

- Maximum message length: ~120 bytes (extended frames used automatically)
- Only clients with permanent IDs can participate
- Broker must be running to forward messages
- No delivery confirmation (fire-and-forget)

## Command Interface

When using the `ClientWithSerial` example, you can send messages using the unified `msg:` command:

```
msg:0:text     - Send to broker (ID 0)
msg:2:text     - Send peer message to client ID 2
msg:5:hello    - Send peer message to client ID 5
```

## See Also

- `ClientWithSerial.ino` - Interactive client with unified messaging command
- `BrokerWithSerial.ino` - Broker that manages registered clients
- `SERIAL_NUMBER_MANAGEMENT.md` - Documentation on serial numbers and ID ranges
- `PUBSUB_API.md` - Complete API reference including `sendPeerMessage()`
