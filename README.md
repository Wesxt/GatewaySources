# GatewaySources

The library provides built-in Gateway servers allowing easy integration with web interfaces, dashboards, or IoT platforms. Both gateways act as a bridge over an existing `Router` or `KNXService` (like `KNXnetIPServer` or `KNXUSBConnection`).

The messages API payloads are exported as `WSClientPayload`, `WSServerPayload`, `MQTTCommandPayload`, and `MQTTStatePayload` so you can use them directly in your TypeScript projects.

## WebSocket Gateway

The `KNXWebSocketGateway` provides a simple, JSON-based bidirectional API over WebSockets, only if you install the dependency or don´t use "--noOptional".

```typescript
import { KNXWebSocketGateway } from "knx.ts/server";
// Or use: import { KNXWebSocketGateway } from 'knx.ts' if exported from root.

const wsGateway = new KNXWebSocketGateway({
  port: 8080,
  knxContext: router, // Provide a Router or any connected KNXService instance
});

wsGateway.start();
```

**WebSocket API JSON Payloads:**

- **Read / Querying**: Request to read from the KNX bus or query cached values.
  - Request: `{ "action": "read", "groupAddress": "1/2/3" }`
  - Response: `{ "action": "read_result", "groupAddress": "1/2/3", "data": ... }`
  - Query Cache: `{ "action": "query", "groupAddress": "1/2/3", "onlyLatest": true }`

- **Write**: Send telegrams to the KNX bus.
  - Command: `{ "action": "write", "groupAddress": "1/2/3", "value": 22.5, "dpt": "9.001" }`

- **Subscribe / Unsubscribe**: Listen to bus events.
  - Subscribe: `{ "action": "subscribe", "groupAddress": "1/2/3" }` (Use `"*"` for all addresses).
  - Event Response: `{ "action": "event", "groupAddress": "1/2/3", "decodedValue": ... }`

- **Configure DPT**: Let the cache know which DPT corresponds to an address.
  - Command: `{ "action": "config_dpt", "groupAddress": "1/2/3", "dpt": "1.001" }`

## MQTT Gateway

The `KNXMQTTGateway` connects to an existing MQTT broker or sets up an embedded one using `aedes`, only if you install the dependency or don´t use "--noOptional".

```typescript
import { KNXMQTTGateway } from "knx.ts/server";

const mqttGateway = new KNXMQTTGateway({
  embeddedBroker: { port: 1883 }, // Or use brokerUrl: "mqtt://your-broker:1883"
  knxContext: router,
  topicPrefix: "knx", // Default is "knx"
});

await mqttGateway.start();
```

**MQTT API Structure:**

- **State Updates**: Whenever the KNX bus emits data, the MQTT gateway publishes the decoded payload to:
  `[prefix]/state/[groupAddress]` (e.g. `knx/state/1/2/3`) with retained flag and JSON: `{ "decodedValue": ... }`.

- **Commands (Write / Read / Config DPT)**: Send JSON payloads to the following command topics.
  - Send Write Command: `[prefix]/command/write/[groupAddress]`
    - Payload: `{ "value": 22.5, "dpt": "9.001" }` or just the raw value if DPT is globally cached.
  - Send Read Command: `[prefix]/command/read/[groupAddress]`
  - Configure DPT: `[prefix]/command/config_dpt/[groupAddress]`
    - Payload: `{ "dpt": "9.001" }`

## Modbus Gateway (Bidirectional KNX/MQTT/Modbus)

The `ModbusGateway` allows bidirectional mapping between Modbus registers (coils, discrete inputs, holding registers, input registers) and KNX Group Addresses or MQTT Topics, only if you install the dependency or don´t use "--noOptional".

**Features Options**:

- Connect as a Modbus Master (Client) to read from slave devices (via TCP or RTU) and push state via KNX or MQTT. You can define specific target slaves (`slaveId`) per mapping!
- Run as a Modbus Slave (Server) offering a register memory map so PLCs or SCADAs can read/write data from/to your KNX/MQTT network.
- Support for 32-bit composite registers (`float32`, `uint32`, `int32`) and their little-endian word-swapped equivalents.
- Advanced payload templating using variable interpolation such as `{{value}}` and multiple logical extractions based on bitmasks configurations `{{<maskKey>}}` for both KNX `valueTemplate` and MQTT `publishTemplate`.
- Inject dynamic mappings at runtime via MQTT or programmatic methods (`addMapping` / `setMappings`).

### Master Mode Example (Reading RS485 Meter mapped to KNX)

```typescript
import { ModbusGateway, Router } from "knx.ts";

const myRouter = new Router();
// Provide a connection to myRouter...

const gatewayMaster = new ModbusGateway({
  mode: "master",
  protocol: "rtu",
  path: "COM3", // e.g., "/dev/ttyUSB0" on Linux
  baudRate: 9600,
  modbusId: 1, // Target slave ID
  knxContext: myRouter,
  defaultPollingInterval: 1000,
  mappings: [
    {
      type: "holding",
      address: 10,   // Holding Register 10
      scale: 0.1,    // Scales e.g. 2250 to 225.0
      slaveId: 1,    // Can override default target ID per mapping
      knx: {
        groupAddress: "1/1/1",
        dpt: "9.020", // Millivolt / Temperature / etc.
        // Needs an object template compliant with KnxDataEncoder.encodeThis()
        valueTemplate: { value: "{{value}}" } 
      }
    }
  ]
});

await gatewayMaster.start();
```

### Slave Mode Example (Emulating a TCP PLC mapped to MQTT)

```typescript
import { ModbusGateway } from "knx.ts";

const gatewaySlave = new ModbusGateway({
  mode: "slave",
  protocol: "tcp",
  host: "0.0.0.0",
  port: 502,
  modbusId: 10, // My Server's Slave ID
  mqtt: {
    brokerUrl: "mqtt://127.0.0.1:1883"
  },
  mappings: [
    {
      type: "coil",
      address: 5,   // Coil 5
      mqtt: {
        topic: "building/lights/floor1",
        publishTemplate: "{\"state\": {{value}}}" 
      }
    }
  ]
});

await gatewaySlave.start();
```

### Dynamic Runtime Mappings (via MQTT)

By default, if the Modbus Gateway is running with an `mqtt` config, it will subscribe to the `modbus/config/add_mapping` topic.
You can send a JSON payload to this topic to add a new register mapping dynamically during execution!

**Topic:** `modbus/config/add_mapping`
**Payload:**

```json
{
  "type": "holding",
  "address": 100,
  "dataType": "uint16",
  "interval": 2000,
  "slaveId": 2,
  "masks": {
    "status": 248,
    "mode": 7
  },
  "knx": {
    "groupAddress": "2/2/2",
    "dpt": "6020",
    "valueTemplate": { "status": "{{status}}", "mode": "{{mode}}" }
  },
  "mqtt": {
    "topic": "building/air/livingroom",
    "publishTemplate": "{\"raw\": {{value}}, \"mode\": {{mode}}}"
  }
}
```
