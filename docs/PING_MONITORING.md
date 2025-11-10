# Ping/Pong Connection Monitoring

## Overview

The SuperCAN library now includes automatic connection monitoring through periodic ping/pong exchanges. The **broker** actively monitors client health by pinging them at regular intervals, and automatically marks unresponsive clients as inactive.

## How It Works

### Broker-Side (Active Monitoring)

1. **Periodic Pings**: The broker sends PING messages to all registered clients at a configurable interval
2. **Response Tracking**: Tracks PONG responses from each client
3. **Missed Ping Counter**: Increments a counter for each unanswered ping
4. **Auto-Disconnect**: Marks clients as inactive after exceeding max missed pings
5. **Callbacks**: Triggers `onClientDisconnect` callback when a client times out

### Client-Side (Passive Response)

1. **Auto-Response**: Clients automatically respond to broker pings with PONG
2. **Callback Notification**: Optional `onPong()` callback for logging/monitoring
3. **Manual Ping**: Clients can still manually ping the broker if needed

## Configuration

### Broker Configuration

```cpp
CANPubSubBroker broker(CAN);

// Set ping interval (default: 5000ms)
broker.setPingInterval(5000);  // Ping every 5 seconds

// Set max missed pings before timeout (default: 2)
broker.setMaxMissedPings(2);   // Mark inactive after 2 missed pings

// Enable automatic ping monitoring
broker.enableAutoPing(true);
```

### Client Configuration

```cpp
CANPubSubClient client(CAN);

// Register callback to log pong responses
client.onPong([]() {
  Serial.println("PONG received from broker");
});
```

## API Reference

### Broker Methods

#### `void setPingInterval(unsigned long intervalMs)`
Set the interval between ping broadcasts to all clients.
- **Parameters**: `intervalMs` - Interval in milliseconds (minimum: 1000ms recommended)
- **Default**: 5000ms

#### `unsigned long getPingInterval()`
Get the current ping interval.
- **Returns**: Current interval in milliseconds

#### `void enableAutoPing(bool enable)`
Enable or disable automatic ping monitoring.
- **Parameters**: `enable` - true to enable, false to disable
- **Default**: false (disabled)

#### `bool isAutoPingEnabled()`
Check if automatic ping is enabled.
- **Returns**: true if enabled, false if disabled

#### `void setMaxMissedPings(uint8_t maxMissed)`
Set the maximum number of consecutive missed pings before marking a client inactive.
- **Parameters**: `maxMissed` - Maximum missed pings (1-10 recommended)
- **Default**: 2

#### `uint8_t getMaxMissedPings()`
Get the current max missed pings threshold.
- **Returns**: Current threshold value

### Client Methods

#### `void onPong(void (*callback)())`
Register a callback that is called when a PONG response is received from the broker.
- **Parameters**: `callback` - Function to call on PONG receipt
- **Example**:
```cpp
client.onPong([]() {
  Serial.print("Pong at ");
  Serial.println(millis());
});
```

#### `bool ping()`
Manually send a ping to the broker (clients can still initiate pings).
- **Returns**: true if ping was sent successfully

## Protocol Details

### Message Format

**PING (0x06)**
```
[PING_ID] [Sender_ID] [Target_ID]
```

**PONG (0x07)**
```
[PONG_ID] [Sender_ID] [Target_ID]
```

### Timing Behavior

- **Broker ping cycle**: Every `pingInterval` milliseconds
- **Missed ping increment**: +1 for each ping without response
- **Missed ping reset**: Set to 0 when PONG received
- **Timeout calculation**: `missedPings >= maxMissedPings`

Example with default settings (5s interval, 2 max missed):
- T=0s: Broker sends ping (missed=1)
- T=5s: No response, broker sends ping (missed=2)
- T=5s: Client marked inactive, `onClientDisconnect()` called

## Usage Examples

### Basic Broker with Auto-Ping

```cpp
#include <SUPER_CAN.h>

CANPubSubBroker broker(CAN);

void setup() {
  CAN.begin(500E3);
  broker.begin();
  
  // Configure auto-ping
  broker.setPingInterval(5000);
  broker.setMaxMissedPings(2);
  broker.enableAutoPing(true);
  
  // Register disconnect callback
  broker.onClientDisconnect([](uint8_t clientId) {
    Serial.print("Client ");
    Serial.print(clientId);
    Serial.println(" timed out");
  });
}

void loop() {
  broker.loop();  // Handles pings automatically
}
```

### Client with Pong Logging

```cpp
#include <SUPER_CAN.h>

CANPubSubClient client(CAN);

void setup() {
  CAN.begin(500E3);
  
  // Log pong responses
  client.onPong([]() {
    Serial.println("Broker ping received, responded with pong");
  });
  
  client.begin("CLIENT-001");
}

void loop() {
  client.loop();  // Responds to pings automatically
}
```

## Notes

- **Only registered clients** (with serial numbers) are monitored via auto-ping
- Temporary clients (without serial numbers) are not tracked
- Ping/pong uses standard CAN frames (not extended)
- Small delay (5ms) between pings to prevent bus congestion
- Client disconnection does NOT remove stored subscriptions (they're restored on reconnect)
- Manual pings from clients still work as before

## See Also

- [PEER_TO_PEER.md](PEER_TO_PEER.md) - Peer-to-peer messaging
- [SERIAL_NUMBER_MANAGEMENT.md](SERIAL_NUMBER_MANAGEMENT.md) - Client registration
- [GETTING_STARTED.md](GETTING_STARTED.md) - Basic setup guide
