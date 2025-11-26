<p align="center">
  <img src="https://img.shields.io/badge/Protocol-IoTIVP%20Gateway-blue?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Bridge-Binary→Core→Verify-green?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Use-Edge%20%7C%20Cloud%20%7C%20n8n-yellow?style=for-the-badge"/>
</p>

# 🌉 IoTIVP-Gateway v1.0  
### **Binary → Core → Verify — Unified Pipeline**

The IoTIVP-Gateway ties the entire protocol together by:

1. Decoding IoTIVP-Binary  
2. Mapping TLV → Core JSON  
3. Recomputing Core hash  
4. Running the Verify engine  
5. Outputting trusted packets  

---

# 🪜 Data Flow

```
IoTIVP-Binary
      ↓
IoTIVP-Core
      ↓
IoTIVP-Verify
      ↓
Applications (Cloud, n8n, Robotics)
```

---

# 🔧 Python Example

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

# 🧩 Features

- ✔ Full Binary → Core → Verify pipeline  
- ✔ TLV + hashing support  
- ✔ Edge & cloud friendly  
- ✔ Perfect for n8n integrations  
- ✔ Robotics-compatible  

---

**IoTIVP-Gateway makes IoTIVP deployable across real-world systems.**
