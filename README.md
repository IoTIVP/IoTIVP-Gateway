# 🌉 IoTIVP-Gateway v1.0  
### **Binary → Core → Verify Pipeline**

IoTIVP-Gateway is the bridge layer connecting all IoTIVP components.

It handles:

1. Binary decoding  
2. TLV → Core field mapping  
3. Core hash computation  
4. Integrity scoring via IoTIVP-Verify  
5. Unified output for cloud, robotics, n8n, dashboards  

---

# 🪜 Data Flow

```
IoTIVP-Binary  →  IoTIVP-Core  →  IoTIVP-Verify  →  Applications
```

---

# 🔧 Example

```python
from gateway import process_binary_packet
from iotivp_binary import BinaryConfig
from iotivp_verify import VerifyConfig

result = process_binary_packet(
    packet_bytes,
    shared_secret=b"demo-secret",
    binary_cfg=BinaryConfig(...),
    verify_cfg=VerifyConfig(...)
)

print(result)
```

---

# 📤 Gateway Output

```json
{
  "core_packet": { ... },
  "verify_result": {
    "valid": true,
    "integrity_score": 94
  }
}
```

---

# 📘 Features

- ✔ Converts TLV to structured fields  
- ✔ Recomputes hash via Core rules  
- ✔ Applies full Verify engine  
- ✔ Plug-and-play with n8n node  
- ✔ Robust decoding for low-level sensors  

---

# 🧱 TLV Field Mapping

```
0x01 -> temperature  
0x02 -> humidity  
0x03 -> battery  
```

Extendable for robotics, GPS, accelerometers, etc.

---

# 🔐 Why IoTIVP-Gateway?

It is the **glue** that makes IoTIVP:

- interoperable  
- deployable  
- cloud-ready  
- automation-ready  

IoTIVP-Gateway is what makes the entire protocol practical.

