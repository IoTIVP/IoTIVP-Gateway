# IoTIVP-Gateway v1.0  
**Internet of Things Integrity Verification Protocol — Gateway Layer**

IoTIVP-Gateway is the **bridge** between:

- Raw IoTIVP-Binary packets coming from devices  
- IoTIVP-Core JSON  
- IoTIVP-Verify integrity scoring

It is designed to run on:

- Edge gateways  
- LoRa / BLE receivers  
- Cloud functions (Lambda, Cloud Run, etc.)  
- n8n / automation platforms (as a script or node backend)

---

## 1. Role in the IoTIVP Ecosystem

```text
+------------------------------------------------------------+
|                       Applications                         |
|   Dashboards, Robotics Control, Analytics Pipelines, etc.  |
+------------------------------------------------------------+
|                    IoTIVP-Verify Engine                    |
+------------------------------------------------------------+
|                      IoTIVP-Gateway                        |
|   (Binary → Core → Verify → Unified Result JSON)           |
+------------------------------------------------------------+
|                IoTIVP-Core / IoTIVP-Binary                 |
+------------------------------------------------------------+
|                   Device Firmware Layer                    |
+------------------------------------------------------------+
```

**Inputs:**

- Raw IoTIVP-Binary packets (bytes or hex)  
- Shared secrets / configuration  

**Outputs:**

- IoTIVP-Core JSON  
- IoTIVP-Verify result (integrity score + flags)  
- A unified JSON object ready for logs, DB, n8n, or robotics.

---

## 2. Expected Dependencies

The gateway uses:

- `IoTIVP-Binary` reference implementation  
  - `iotivp_binary.py`  
- `IoTIVP-Core` hash helper  
  - `iotivp_core_hash.py`  
- `IoTIVP-Verify` engine  
  - `iotivp_verify.py`  

Folder example:

```text
IoTIVP-Gateway/
 ├─ README.md
 ├─ gateway.py
 ├─ requirements.txt  (optional)
```

You can either:

- vendor the modules locally, or  
- install them as packages if you publish them to PyPI later.

---

## 3. TLV → Field Name Mapping

The gateway needs a mapping between TLV type IDs and Core field names.

Example base mapping:

```python
TYPE_FIELD_MAP = {
    0x01: "temperature",
    0x02: "humidity",
    0x03: "battery",
    # 0x10–0x1F etc. can be robotics, GPS, etc.
}
```

Encoding conventions for this reference:

- `temperature` → int16 fixed-point, value = °C × 10  
- `humidity` → uint8, 0–100 %  
- `battery` → uint8, 0–100 %

---

## 4. Reference Implementation (`gateway.py`)

Create a file:

`gateway.py`

```python
from typing import Dict, Any, Tuple
from iotivp_binary import BinaryConfig, decode_packet
from iotivp_core_hash import compute_core_hash
from iotivp_verify import VerifyConfig, verify_packet

# Simple mapping from TLV type IDs to Core field names
TYPE_FIELD_MAP = {
    0x01: "temperature",  # fixed-point x10
    0x02: "humidity",     # percent
    0x03: "battery",      # percent
    # extend with robotics, GPS, etc.
}


def tlv_to_core_fields(tlv_fields) -> Dict[str, Any]:
    """
    Convert TLV (type_id, value_bytes) list into IoTIVP-Core 'fields' dict.
    """
    core_fields: Dict[str, Any] = {}

    for type_id, value_bytes in tlv_fields:
        name = TYPE_FIELD_MAP.get(type_id)
        if name is None:
            # Unknown / unregistered type, skip or store as raw
            continue

        # Example decoding rules for reference profile:
        if name == "temperature":
            # int16 fixed-point (x10)
            if len(value_bytes) != 2:
                continue
            raw = int.from_bytes(value_bytes, "big", signed=True)
            core_fields[name] = raw / 10.0

        elif name in ("humidity", "battery"):
            # uint8 0–100
            if len(value_bytes) != 1:
                continue
            core_fields[name] = int.from_bytes(value_bytes, "big")

        else:
            # Future: more complex fields
            core_fields[name] = value_bytes.hex()

    return core_fields


def process_binary_packet(
    packet_bytes: bytes,
    shared_secret: bytes,
    binary_cfg: BinaryConfig,
    verify_cfg: VerifyConfig,
) -> Dict[str, Any]:
    """
    Full pipeline:
      1) Decode IoTIVP-Binary packet
      2) Map TLV → Core fields
      3) Build IoTIVP-Core packet dict
      4) Compute Core hash
      5) Run IoTIVP-Verify engine
      6) Return combined result

    Args:
        packet_bytes: incoming IoTIVP-Binary packet
        shared_secret: secret used in Core hash
        binary_cfg: BinaryConfig (must match encoder)
        verify_cfg: VerifyConfig (must match hash settings)

    Returns:
        {
          "core_packet": { ... IoTIVP-Core JSON ... },
          "verify_result": { ... integrity result ... }
        }
    """
    # 1) Decode Binary
    decoded = decode_packet(packet_bytes, cfg=binary_cfg)

    header = decoded["header"]
    timestamp = decoded["timestamp"]
    device_id = decoded["device_id"]
    nonce = decoded["nonce"]
    tlv_fields = decoded["fields"]

    # 2) TLV → Core fields
    core_fields = tlv_to_core_fields(tlv_fields)

    # 3) Build temporary Core packet (hash to be filled)
    core_packet = {
        "header": header,
        "timestamp": timestamp,
        "device_id": device_id,
        "nonce": nonce,
        "fields": core_fields,
        "hash": ""  # placeholder
    }

    # 4) Compute Core hash
    core_hash = compute_core_hash(core_packet, shared_secret, hash_len=verify_cfg.hash_len)
    core_packet["hash"] = core_hash

    # 5) Verify
    verify_result = verify_packet(core_packet, shared_secret, verify_cfg)

    # 6) Return unified structure
    return {
        "core_packet": core_packet,
        "verify_result": verify_result,
    }


def process_binary_hex(
    packet_hex: str,
    shared_secret: bytes,
    binary_cfg: BinaryConfig,
    verify_cfg: VerifyConfig,
) -> Dict[str, Any]:
    """
    Helper for hex string input (e.g. from logs or MQTT).
    """
    packet_bytes = bytes.fromhex(packet_hex)
    return process_binary_packet(packet_bytes, shared_secret, binary_cfg, verify_cfg)
```

---

## 5. Example Usage

Example quick test (assuming `iotivp_binary`, `iotivp_core_hash`, and `iotivp_verify` are available):

Create `test_gateway.py`:

```python
from iotivp_binary import BinaryConfig, encode_packet
from iotivp_verify import VerifyConfig
from gateway import process_binary_packet

# Shared secret for Core hashing
SECRET = b"demo-secret"

# Binary profile (must match your encoder)
binary_cfg = BinaryConfig(
    timestamp_len=2,
    device_id_len=2,
    nonce_len=1,
    hash_len=4,
    hash_alg="blake2s"
)

verify_cfg = VerifyConfig(
    max_age_seconds=60,
    hash_alg="blake2s",
    hash_len=4,
    field_ranges={
        "temperature": {"min": -40, "max": 85},
        "humidity": {"min": 0, "max": 100},
        "battery": {"min": 0, "max": 100},
    }
)

# Build a sample packet via the Binary encoder
header = 0x01
timestamp = 513
device_id = 42
nonce = 7

temp_celsius = int(23.5 * 10)   # 23.5°C → 235
hum_percent = 60
battery = 91

tlv_fields = [
    (0x01, temp_celsius.to_bytes(2, "big")),  # temperature
    (0x02, hum_percent.to_bytes(1, "big")),   # humidity
    (0x03, battery.to_bytes(1, "big")),       # battery
]

packet_bytes = encode_packet(
    header=header,
    timestamp=timestamp,
    device_id=device_id,
    tlv_fields=tlv_fields,
    cfg=binary_cfg,
    nonce=nonce,
)

result = process_binary_packet(packet_bytes, SECRET, binary_cfg, verify_cfg)

print("Core Packet:")
print(result["core_packet"])
print("\nVerify Result:")
print(result["verify_result"])
```

Run:

```bash
python test_gateway.py
```

You should see:

- A decoded **IoTIVP-Core JSON**  
- A **verification result** with `integrity_score` and flags  

---

## 6. How Gateways Use This in Production

In a real deployment, the gateway will:

1. Receive packets (UDP, MQTT, LoRaWAN app payload, BLE, etc.)  
2. Feed the raw bytes into `process_binary_packet`  
3. Store `core_packet` + `verify_result` into:
   - time-series DB (e.g., InfluxDB, Timescale)  
   - message queues (Kafka, NATS, MQTT topics)  
   - HTTP/webhooks (n8n, custom APIs)  
4. Optionally discard packets with low integrity scores (e.g., `< 70`)  

---

## 7. Versioning

- IoTIVP-Gateway v1.0  
- Compatible with:
  - IoTIVP-Binary v1.0  
  - IoTIVP-Core v1.5  
  - IoTIVP-Verify v1.5  

---

IoTIVP-Gateway makes the whole IoTIVP ecosystem **practical and pluggable**  
for real devices, real networks, and real automation flows.
