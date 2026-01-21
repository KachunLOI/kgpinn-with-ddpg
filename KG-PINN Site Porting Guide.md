# KG-PINN Site Porting Guide

This guide describes how to port the KG-PINN model to a new data center site.

---

## (1) Normalization Config

**What changes:** The min/max (or mean/std) ranges used for feature-wise scaling of temperature/flow/load variables.

### Example: `normalization.yaml`

```yaml
temperature:
  output_room_T_TH:      {min: 12.0, max: 45.0}   # Room temperature (26 sensors)
  output_room_T_cr:      {min: 12.0, max: 28.0}   # CRAH supply temperature (8 sensors)
  output_room_T_return:  {min: 22.0, max: 45.0}   # CRAH return temperature (8 sensors)

airflow:
  input_crah_u_cr:       {min: 10.0, max: 90.0}   # CRAH fan speed (8 sensors)

load:
  IT_Load:               {min: 0.0,  max: 3.6e4}  # IT load (144 sensors)
```

### Code Hook

```python
norm = load_yaml("normalization.yaml")

def normalize(x, key):
    r = norm[key["group"]][key["name"]]
    return (x - r["min"]) / (r["max"] - r["min"] + 1e-8)
```

---

## (2) Layout & Zoning

**What changes:** Entity instances and mappings that reflect aisle topology, row count, CRAH placement, and rack membership per zone.

### Example: `layout_map.json`

```json
{
  "site_id": "GPU_HALL_2",
  "zones": {
    "Z_ROOM": {
      "type": "room",
      "sensor_count": 26,
      "sensors": ["output_room_T_TH_01", "output_room_T_TH_02", "...", "output_room_T_TH_26"]
    },
    "Z_COLD": {
      "type": "cold_aisle",
      "rack_count": 144,
      "racks": ["A01", "A02", "...", "L12"]
    },
    "Z_HOT": {
      "type": "hot_aisle",
      "rack_count": 144,
      "racks": ["A01", "A02", "...", "L12"]
    }
  },
  "it_load": {
    "sensor_count": 144,
    "sensors": ["IT_Load_A01", "IT_Load_A02", "...", "IT_Load_L12"]
  },
  "crah_units": {
    "unit_count": 8,
    "CRAH_01": {
      "supply_temp": "output_room_T_cr01_01",
      "return_temp": "output_room_T_return_01",
      "fan_speed": "input_crah_u_cr01_01"
    },
    "CRAH_02": {
      "supply_temp": "output_room_T_cr01_02",
      "return_temp": "output_room_T_return_02",
      "fan_speed": "input_crah_u_cr01_02"
    },
    "CRAH_08": {
      "supply_temp": "output_room_T_cr01_08",
      "return_temp": "output_room_T_return_08",
      "fan_speed": "input_crah_u_cr01_08"
    }
  }
}
```

### Code Hook

```python
kg = KG.load("kg_base_schema.json")
layout = json.load(open("layout_map.json"))

kg.upsert_entities_from_layout(layout)
kg.upsert_zone_membership(layout["zones"])
kg.upsert_crah_links(layout["crah_units"])
kg.save("kg_site_GPU_HALL_2.json")
```

---

## (3) Airflow Configuration & Rule Coefficients

**What changes:** Which rule-set is active and its coefficients (e.g., heat transfer `hA`, leakage factor `eta`, specific heat `c_p`).

### Example: `kg_rules.yaml`

```yaml
airflow_mode: "single_sided"   # or "double_sided"

coefficients:
  # Zone-1 (Cold Aisle)
  hA:           520.0    # effective heat transfer coefficient (W/K)
  eta:          0.08     # leakage factor for non-recoverable heat loss
  c_p:          1005.0   # specific heat capacity of air (J/(kg·K))
  
  # Zone-2 (Hot Aisle)
  n_zones:      4        # number of cold aisle zones for averaging
  
  # Zone-3 (Return)
  delta_t:      1.0     # sampling interval (s)
  T_return_set: 35.0     # return temperature setpoint (°C)
```

### Code Hook

```python
rules = load_yaml("kg_rules.yaml")

if rules["airflow_mode"] == "single_sided":
    kg.activate_rulepack("RULEPACK_SINGLE_SIDE")
else:
    kg.activate_rulepack("RULEPACK_DOUBLE_SIDE")

kg.set_rule_coefficients(rules["coefficients"])
kg.save("kg_site_calibrated.json")
```

### Calibration Routine (Lightweight)

- **Input:** 24–72h site data
- **Output:** Best-fit coefficients to minimize intermediate target errors (e.g., predicted zone temperatures)

```python
def calibrate_coeffs(data, init_coeffs):
    # minimize error between KG-inferred intermediates and measurements/CFD references
    # e.g., optimize [hA, eta, c_p, n_zones, delta_t, T_return_set] with bounded search
    return best_coeffs

coeffs = calibrate_coeffs(site_data, init_coeffs=rules["coefficients"])
kg.set_rule_coefficients(coeffs)
```

---

## (4) Rule Validation

**What to validate:** KG-derived intermediate features used by the surrogate (e.g., zone temperatures `T_cold`, `T_hot`, `T_return`). This step prevents "garbage-in" features.

### Example: `validate_kg.py`

```python
pred = kg.infer_intermediates(site_data)     # e.g., T_cold, T_hot, T_return
ref  = load_reference(site_data)             # sensors or small CFD subset

report = {
    "MAE_T_cold":   mae(pred["T_cold"], ref["T_cold"]),
    "MAE_T_hot":    mae(pred["T_hot"], ref["T_hot"]),
    "MAE_T_return": mae(pred["T_return"], ref["T_return"]),
}
assert report["MAE_T_cold"] < thresholds["T_cold"]
save_json("kg_validation_report.json", report)
```

### Acceptance Gates

- Intermediate-feature MAE within predefined tolerances
- Monotonicity/sanity checks (e.g., higher IT load should increase `T_hot`)

---

## Minimal Porting Pipeline

### Step 1: Prepare Configuration Files

Update `normalization.yaml` and `layout_map.json` for the new site.

### Step 2: Configure Airflow Mode

Select `airflow_mode` and initial coefficients in `kg_rules.yaml`.

### Step 3: Build & Calibrate KG

```bash
python build_kg.py --layout layout_map.json --out kg_site.json
python calibrate_kg_rules.py --kg kg_site.json --data site_72h.parquet --out kg_site_calibrated.json
python validate_kg.py --kg kg_site_calibrated.json --data site_24h.parquet --report kg_validation_report.json
```

### Step 4 (Optional): Fine-tune Surrogate

Run short fine-tuning (small-sample) using the same network architecture:

```bash
python finetune_surrogate.py --ckpt pretrained.ckpt --kg kg_site_calibrated.json --data site_train.parquet --epochs 1k-5k
```
