# Flash Storage Quick Reference

## Overview
Client ID to serial number mappings, **topic subscriptions**, and **topic names** are **automatically stored in flash memory** and persist across power cycles.

## Platform Support

| Platform | Storage Type | Library | Capacity |
|----------|-------------|---------|----------|
| ESP32 | NVS (Preferences) | `Preferences.h` | ~100,000 writes/sector |
| Arduino AVR | EEPROM | `EEPROM.h` | ~100,000 writes |
| ESP8266 | EEPROM emulation | `EEPROM.h` | ~10,000 writes |
| Arduino SAMD | Flash | `FlashStorage.h` | ~10,000 writes |

## Automatic Storage

Storage operations happen **automatically**:

```cpp
broker.begin();                    // ← Loads mappings, subscriptions, topic names AND ping config from flash automatically
broker.registerClient("ESP32_001"); // ← Saves to flash automatically
broker.unregisterClient(0x10);     // ← Saves to flash automatically

// Subscriptions AND topic names are persisted automatically
// When client subscribes, their subscription AND topic name are saved
// When client reconnects, subscriptions are restored WITH topic names

// Ping configuration is also persisted automatically
broker.enableAutoPing(true);       // ← Saves to flash automatically
broker.setPingInterval(10000);     // ← Saves to flash automatically
broker.setMaxMissedPings(3);       // ← Saves to flash automatically
```

## Manual Storage Control

For advanced use cases:

```cpp
// Manual save (automatic saves disabled)
broker.saveMappingsToStorage();
broker.saveSubscriptionsToStorage();
broker.saveTopicNamesToStorage();
broker.savePingConfigToStorage();

// Manual load (useful after external changes)
broker.loadMappingsFromStorage();
broker.loadSubscriptionsFromStorage();
broker.loadTopicNamesFromStorage();
broker.loadPingConfigFromStorage();

// Clear all stored data
broker.clearStoredMappings();
broker.clearStoredSubscriptions();
broker.clearStoredTopicNames();
broker.clearStoredPingConfig();
```

## Storage Size

```
Default configuration:
- Maximum clients: 50
- Serial length: 32 chars
- Max subscriptions per client: 10
- Max stored topic names: 20
- Topic name length: 32 chars
- Storage per client mapping: 34 bytes
- Storage per subscription set: 23 bytes (1 + 10×2 + 1)
- Storage per topic name: 35 bytes (2 + 32 + 1)
- Total client mapping storage: ~1700 bytes
- Total subscription storage: ~1150 bytes
- Total topic name storage: ~700 bytes
- Ping configuration storage: 6 bytes (bool + ulong + uint8_t)

Total memory usage:
Client Mappings:
- Header: 4 bytes (magic + count + nextID)
- Client data: 34 × 50 = 1700 bytes
- Subtotal: 1704 bytes

Subscriptions:
- Header: 3 bytes (magic + count)
- Subscription data: 23 × 50 = 1150 bytes
- Subtotal: 1153 bytes

Topic Names:
- Header: 3 bytes (magic + count)
- Topic data: 35 × 20 = 700 bytes
- Subtotal: 703 bytes

Ping Configuration:
- Enabled flag: 1 byte (bool)
- Interval: 4 bytes (unsigned long)
- Max missed pings: 1 byte (uint8_t)
- Subtotal: 6 bytes

Total: ~3566 bytes (EEPROM size: 8192 bytes)
```

## Configuration

Edit `CANPubSub.h` to customize:

```cpp
#define MAX_CLIENT_MAPPINGS 50          // More clients = more storage
#define MAX_SERIAL_LENGTH 32            // Longer serials = more storage
#define MAX_STORED_SUBS_PER_CLIENT 10   // More subscriptions per client = more storage
#define MAX_STORED_TOPIC_NAMES 20       // More topic names = more storage
#define MAX_TOPIC_NAME_LENGTH 32        // Longer topic names = more storage
#define EEPROM_SIZE 8192                // Total EEPROM size (Arduino, increased for topic names)
```

## Power Cycle Testing

```cpp
// Step 1: Register clients and subscribe to topics
uint8_t client1 = broker.registerClient("TEST_001");
uint8_t client2 = broker.registerClient("TEST_002");

// Client subscribes to topics (manually simulated or through actual client)
// Subscriptions AND topic names are automatically saved

// Step 2: Check count
Serial.println(broker.getRegisteredClientCount()); // → 2
Serial.println(broker.getSubscriptionCount());     // → # of active topics

// Step 3: Power off device

// Step 4: Power on and check
broker.begin();
Serial.println(broker.getRegisteredClientCount()); // → 2 (persisted!)
// Topic names are also loaded from flash

// Step 5: Client reconnects with same serial number
// Subscriptions are automatically restored WITH topic names!
```

## Storage Verification

```cpp
void setup() {
  broker.begin();
  
  // Check if data was loaded
  uint8_t count = broker.getRegisteredClientCount();
  if (count > 0) {
    Serial.print("Loaded ");
    Serial.print(count);
    Serial.println(" clients from flash");
  } else {
    Serial.println("Fresh start (no stored data)");
  }
}
```

## Data Format

```
Flash Memory Layout:
┌─────────────────────────────────────────────┐
│ CLIENT MAPPINGS SECTION                     │
├─────────────────────────────────────────────┤
│ Magic Number (0xCABE)         [2 bytes]    │ ← Validates client data
├─────────────────────────────────────────────┤
│ Mapping Count                  [1 byte]     │
├─────────────────────────────────────────────┤
│ Next Client ID                 [1 byte]     │
├─────────────────────────────────────────────┤
│ Client Mapping #1             [34 bytes]    │
│   - Client ID                  [1 byte]     │
│   - Serial Number             [32 bytes]    │
│   - Active Flag                [1 byte]     │
├─────────────────────────────────────────────┤
│ Client Mapping #2             [34 bytes]    │
├─────────────────────────────────────────────┤
│ ...                                          │
├─────────────────────────────────────────────┤
│ Client Mapping #50            [34 bytes]    │
├─────────────────────────────────────────────┤
│ SUBSCRIPTIONS SECTION                       │
├─────────────────────────────────────────────┤
│ Sub Magic Number (0xCAFF)     [2 bytes]    │ ← Validates subscription data
├─────────────────────────────────────────────┤
│ Subscription Count             [1 byte]     │
├─────────────────────────────────────────────┤
│ Client Subscriptions #1       [23 bytes]    │
│   - Client ID                  [1 byte]     │
│   - Topic Hashes (×10)        [20 bytes]    │
│   - Topic Count                [1 byte]     │
├─────────────────────────────────────────────┤
│ Client Subscriptions #2       [23 bytes]    │
├─────────────────────────────────────────────┤
│ ...                                          │
├─────────────────────────────────────────────┤
│ Client Subscriptions #50      [23 bytes]    │
├─────────────────────────────────────────────┤
│ PING CONFIGURATION SECTION                  │
├─────────────────────────────────────────────┤
│ Auto-Ping Enabled              [1 byte]     │ ← bool
├─────────────────────────────────────────────┤
│ Ping Interval (ms)             [4 bytes]    │ ← unsigned long
├─────────────────────────────────────────────┤
│ Max Missed Pings               [1 byte]     │ ← uint8_t
├─────────────────────────────────────────────┤
│ TOPIC NAMES SECTION                         │
├─────────────────────────────────────────────┤
│ Topic Magic Number (0xFEED)   [2 bytes]    │ ← Validates topic name data
├─────────────────────────────────────────────┤
│ Topic Name Count               [1 byte]     │
├─────────────────────────────────────────────┤
│ Stored Topic Name #1          [35 bytes]    │
│   - Topic Hash                 [2 bytes]    │
│   - Topic Name                [32 bytes]    │
│   - Active Flag                [1 byte]     │
├─────────────────────────────────────────────┤
│ Stored Topic Name #2          [35 bytes]    │
├─────────────────────────────────────────────┤
│ ...                                          │
├─────────────────────────────────────────────┤
│ Stored Topic Name #20         [35 bytes]    │
└─────────────────────────────────────────────┘
```

## Error Handling

```cpp
// Check if load was successful
if (!broker.loadMappingsFromStorage()) {
  Serial.println("No valid data in storage");
  // This is normal for first run
}

// Check if save was successful
if (!broker.saveMappingsToStorage()) {
  Serial.println("Save failed!");
  // Rare - indicates storage hardware issue
}

// Verify magic number
if (loadFailed) {
  // Either:
  // 1. First run (no data yet)
  // 2. Flash corruption
  // 3. Different version
  broker.clearStoredMappings(); // Start fresh
}
```

## Best Practices

### ✅ DO:
- Let the library handle storage automatically
- Check `getRegisteredClientCount()` after `begin()`
- Use serial numbers < 32 characters
- Test power cycle behavior
- Periodically verify storage integrity

### ❌ DON'T:
- Manually write to EEPROM addresses used by library
- Call `saveMappingsToStorage()` excessively (wear)
- Store >50 clients without increasing `MAX_CLIENT_MAPPINGS`
- Use random serial numbers (defeats persistence)
- Ignore load/save return values in production

## Wear Management

Flash/EEPROM has limited write cycles. The library minimizes writes:

```cpp
// Each of these triggers writes to mapping storage:
broker.registerClient("NEW_001");      // New registration
broker.unregisterClient(0x10);         // State change
broker.updateClientSerial(0x10, "X");  // Update

// Each of these triggers writes to subscription storage:
// (called automatically via addSubscription/removeSubscription)
client.subscribe("sensor/temp");       // New subscription
client.unsubscribe("sensor/temp");     // Remove subscription

// Each of these triggers writes to ping config storage:
broker.enableAutoPing(true);           // Enable/disable auto-ping
broker.setPingInterval(10000);         // Change ping interval
broker.setMaxMissedPings(3);           // Change max missed pings

// These do NOT trigger writes:
broker.getClientIdBySerial("ESP32");   // Read only
broker.getSerialByClientId(1);         // Read only
broker.listRegisteredClients(...);     // Read only
broker.getSubscriptionCount();         // Read only
broker.isAutoPingEnabled();            // Read only
broker.getPingInterval();              // Read only
broker.getMaxMissedPings();            // Read only
```

**Estimated lifetime:**
- 50 clients with average 5 subscription changes/day each
- 250 writes/day (subscriptions)
- 50 writes/day (registrations)
- 5 writes/day (ping config changes)
- Total: 305 writes/day × 365 days = 111,325 writes/year
- ESP32: 100,000 writes = **11+ months per sector** (but ESP32 has wear leveling)
- Arduino AVR: 100,000 writes = **11+ months** (heavy use)
- ESP8266: 10,000 writes = **1 month** (very heavy use - minimize subscription changes)

**Recommendation**: For production systems with frequent changes, consider:
1. Using ESP32 (has built-in wear leveling)
2. Batching subscription changes when possible
3. Avoiding rapid subscribe/unsubscribe cycles
4. Setting ping configuration once during initial setup, not frequently

## Troubleshooting

### Mappings not persisting?

```cpp
// Check platform
#ifdef ESP32
  Serial.println("Using ESP32 Preferences");
#else
  Serial.println("Using EEPROM");
#endif

// Verify save
if (broker.saveMappingsToStorage()) {
  Serial.println("Save OK");
} else {
  Serial.println("Save FAILED");
}

// Check after reboot
broker.begin();
Serial.print("Loaded: ");
Serial.println(broker.getRegisteredClientCount());
```

### Storage full?

```cpp
uint8_t count = broker.getRegisteredClientCount();
if (count >= MAX_CLIENT_MAPPINGS) {
  Serial.println("Storage full! Increase MAX_CLIENT_MAPPINGS");
}
```

### Corruption?

```cpp
// Reset to factory defaults
broker.clearStoredMappings();
ESP.restart(); // or reset device
```

## Ping Configuration Persistence

### Overview

Ping monitoring settings are **automatically saved to flash** and restored on power cycle:

- **ping:on/off** - Auto-ping enabled/disabled state
- **interval:ms** - Ping interval in milliseconds  
- **maxmissed:n** - Maximum missed pings before disconnect

### How It Works

```cpp
// First boot - configure ping monitoring
broker.begin();
broker.enableAutoPing(true);      // ← Saved to flash
broker.setPingInterval(10000);    // ← Saved to flash (10 seconds)
broker.setMaxMissedPings(3);      // ← Saved to flash

// Power cycle...

// Second boot - settings automatically restored!
broker.begin();                    // ← Loads ping config from flash
// Auto-ping is enabled with 10s interval and max 3 missed pings
// No need to reconfigure!
```

### Benefits

✅ **Configuration survives power loss** - Settings persist across reboots  
✅ **No reconfiguration needed** - Broker remembers your settings  
✅ **Consistent behavior** - Same monitoring parameters every time  
✅ **Easy deployment** - Configure once, works forever  

### Default Values

If no ping configuration is stored (first boot or after clear), defaults are:

- Auto-ping: **disabled** (`false`)
- Ping interval: **5000 ms** (5 seconds)
- Max missed pings: **2**

## Subscription Persistence Details

### How It Works

When a client subscribes to a topic:
1. Subscription is added to broker's active subscription table
2. Client's subscription list is saved to flash automatically
3. On power cycle, broker loads subscription data
4. When client reconnects (with same serial number):
   - Broker recognizes client by serial number
   - Restores all subscriptions from flash
   - Client receives messages immediately without re-subscribing

### Example Flow

```cpp
// === First Boot ===
broker.begin();
// Client connects with serial "SENSOR_001", gets ID 1 (first client)
// Client subscribes to "temperature" and "humidity"
// Subscriptions saved to flash automatically

// === Power Cycle ===

// === Second Boot ===
broker.begin();
// Loads client mapping: "SENSOR_001" → 1
// Loads subscriptions: 1 → ["temperature", "humidity"]

// Client reconnects with serial "SENSOR_001"
// Gets same ID 1
// Subscriptions automatically restored!
// Client immediately receives temperature/humidity messages
```

### Benefits

✅ **Zero reconfiguration** - Clients work immediately after power loss  
✅ **Seamless recovery** - Network topology preserved  
✅ **Reduced CAN traffic** - No need to re-send subscription requests  
✅ **Better reliability** - No lost messages during reconnection  

### Limitations

⚠️ Maximum 10 subscriptions per client (configurable via `MAX_STORED_SUBS_PER_CLIENT`)  
⚠️ Maximum 20 topic names stored (configurable via `MAX_STORED_TOPIC_NAMES`)  
⚠️ Topic names truncated to 32 characters (configurable via `MAX_TOPIC_NAME_LENGTH`)  
⚠️ Subscriptions persist even if client unregisters (use `clearStoredSubscriptions()` to clean up)

## Example: Storage Test

See `examples/CANPubSubStorageTest/` or `examples/BrokerWithSerial/` for complete examples.

```cpp
// Register test clients
broker.registerClient("TEST_001");
broker.registerClient("TEST_002");

// Power cycle device...

// After reboot
broker.begin();
Serial.print("Loaded: ");
Serial.println(broker.getRegisteredClientCount()); // → 2

// List them
broker.listRegisteredClients([](uint8_t id, const String& sn, bool active) {
  Serial.print("ID: ");
  Serial.print(id, DEC);
  Serial.print(" → ");
  Serial.println(sn);
});

// Subscriptions are automatically restored when clients reconnect
```

---

**Key Point**: Flash storage makes your CAN network configuration **persistent and reliable**, with client mappings, subscriptions, **and topic names** surviving power cycles! Clients reconnecting after power loss will automatically have their subscriptions restored with the correct topic names.
