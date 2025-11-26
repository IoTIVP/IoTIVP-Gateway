# n8n Node Specification — IoTIVP Verify (Gateway)

## 1. Node Name and Purpose

**Node Name:** `IoTIVP Verify (Gateway)`  
**Category:** IoT / Security / Data Integrity  

This node takes a raw **IoTIVP-Binary** packet (hex string), runs it through the **IoTIVP-Gateway pipeline**, and returns:

- Decoded **IoTIVP-Core** JSON
- **IoTIVP-Verify** result (integrity score + flags)
- A unified output for downstream nodes (logging, alerts, robotics, dashboards)

---

## 2. Inputs and Outputs (n8n-style)

### 2.1 Inputs

- **Mode:** Regular node (not trigger)
- **Input format:** n8n item(s) with a JSON field:

```json
{
  "packet_hex": "01020304abcd..."
}
```

Optionally:

- `device_id` (if you want to override / tag logs)
- `profile` (e.g. `"environment"`, `"robotics"`)

### 2.2 Outputs

Each output item will look like:

```json
{
  "core_packet": {
    "header": 1,
    "timestamp": 1732212000,
    "device_id": 42,
    "nonce": 7,
    "fields": {
      "temperature": 23.5,
      "humidity": 60,
      "battery": 91
    },
    "hash": "baf24977"
  },
  "verify_result": {
    "valid": true,
    "integrity_score": 94.0,
    "dimensions": {
      "hash_validity": 1.0,
      "timestamp_validity": 0.9,
      "nonce_behavior": 1.0,
      "value_anomalies": 0.9,
      "device_behavior": 1.0
    },
    "flags": {
      "hash_mismatch": false,
      "timestamp_expired": false,
      "nonce_reuse": false,
      "value_out_of_range": [],
      "device_suspicious": false
    }
  }
}
```

This is the object n8n will pass to later nodes (e.g. IF, HTTP Request, Slack, database).

---

## 3. Node Parameters (Config Panel)

When you click the node in n8n, you should see these settings:

### 3.1 Required Parameters

1. **Shared Secret (string)**
   - Label: `Shared Secret`
   - Type: string (or credential later)
   - Used by: IoTIVP-Core hash + IoTIVP-Verify

2. **Timestamp Max Age (seconds)**
   - Label: `Max Age (seconds)`
   - Default: `60`
   - Used by: IoTIVP-Verify `max_age_seconds`

3. **Hash Algorithm**
   - Label: `Hash Algorithm`
   - Type: dropdown
   - Options: `blake2s`, `sha256`
   - Default: `blake2s`

4. **Hash Length (bytes)**
   - Label: `Hash Length`
   - Default: `4`
   - Used to truncate digest

5. **Binary Layout**
   - Group: `Binary Settings`
   - Fields:
     - `Timestamp Length` (default: `2`)
     - `Device ID Length` (default: `2`)
     - `Nonce Length` (default: `1`)
   - These must match firmware.

### 3.2 Optional Parameters

6. **Field Ranges (JSON)**
   - Label: `Field Ranges`
   - Type: JSON text input
   - Example:
     ```json
     {
       "temperature": { "min": -40, "max": 85 },
       "humidity": { "min": 0, "max": 100 },
       "battery": { "min": 0, "max": 100 }
     }
     ```
   - Passed into IoTIVP-Verify for anomaly detection.

7. **Drop Low-Integrity Packets**
   - Label: `Drop Score < Threshold`
   - Type: boolean (default: `false`)
   - Additional field when `true`: `Threshold` (e.g. `70`)

---

## 4. Internal Flow (Node Logic)

Inside this node, the logic will be equivalent to:

```text
For each incoming item:
    1) Read input.item.json.packet_hex
    2) Convert hex → bytes
    3) Create BinaryConfig from node settings
    4) Create VerifyConfig from node settings
    5) Call process_binary_packet(packet_bytes, secret, binary_cfg, verify_cfg)
    6) Attach result to item.json and pass on
```

### 4.1 Pseudo-code (TypeScript-style for n8n)

```ts
const packetHex = item.json.packet_hex as string;

const binaryCfg = {
  timestamp_len: node.params.timestampLength,
  device_id_len: node.params.deviceIdLength,
  nonce_len: node.params.nonceLength,
  hash_len: node.params.hashLength,
  hash_alg: node.params.hashAlgorithm,
};

const verifyCfg = {
  max_age_seconds: node.params.maxAgeSeconds,
  hash_alg: node.params.hashAlgorithm,
  hash_len: node.params.hashLength,
  field_ranges: node.params.fieldRangesJson,
};

const secret = node.params.sharedSecret;

const result = process_binary_hex(packetHex, secret, binaryCfg, verifyCfg);

item.json.core_packet = result.core_packet;
item.json.verify_result = result.verify_result;

return item;
```

In your actual implementation, this logic will live in the node’s `execute()` function.

---

## 5. Example n8n Workflow

Simple example:

1. **Webhook Trigger**
   - Receives JSON:
     ```json
     {
       "packet_hex": "010201...ff"
     }
     ```

2. **IoTIVP Verify (Gateway) Node**
   - Uses the above spec
   - Outputs `core_packet` + `verify_result`

3. **IF Node**
   - Condition: `verify_result.integrity_score >= 80`

4. **True branch**
   - Write to database or send to robotics system.

5. **False branch**
   - Log as suspicious / send alert / ignore.

---

## 6. Future Enhancements

- Support multiple **input modes**:
  - Direct `packet_hex` input
  - `packet_bytes` (Base64)
- Support **different profiles** (`environment`, `robotics`, `ble`) with pre-set field ranges.
- Add **credential type** for the shared secret instead of plain string.
- Add **metrics output** for dashboards:
  - moving average of integrity scores
  - count of dropped packets
  - anomaly breakdowns

---

## 7. Implementation Status

This file defines the **specification** for the n8n node.

Implementation steps (future):

1. Create a custom n8n node package:
   - `n8n-nodes-iotivp-verify`
2. Implement the `IoTIVP Verify (Gateway)` node using:
   - `process_binary_hex` from IoTIVP-Gateway
   - `VerifyConfig` and `BinaryConfig`
3. Install into your n8n instance as a custom node.
4. Test with `test_gateway.py`-generated packets.

For now, this spec serves as:

- Documentation for how the node should behave
- A contract for any implementation (you or collaborators)
- A reference for showcasing IoTIVP’s integration story
