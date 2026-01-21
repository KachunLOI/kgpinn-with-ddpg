
1. Normalization Configuration
Objective. Update feature-wise normalization parameters (min/max or mean/std) for temperature, airflow/actuation, and load variables such that the input scaling reflects the target site’s operating envelope.
1.1 Example: normalization.yaml
temperature:
  output_room_T_TH:      {min: 12.0, max: 45.0}   # Room temperature (26 sensors)
  output_room_T_cr:      {min: 12.0, max: 28.0}   # CRAH supply temperature (8 sensors)
  output_room_T_return:  {min: 22.0, max: 45.0}   # CRAH return temperature (8 sensors)

airflow:
  input_crah_u_cr:       {min: 10.0, max: 90.0}   # CRAH fan speed (8 sensors)

load:
  IT_Load:               {min: 0.0,  max: 3.6e4}  # IT load (144 sensors)
1.2 Normalization Hook (Illustrative)
norm = load_yaml("normalization.yaml")

def normalize(x, key):
    r = norm[key["group"]][key["name"]]
    return (x - r["min"]) / (r["max"] - r["min"] + 1e-8)

2. Layout and Zoning Instantiation
Objective. Represent the site topology (zones, aisles, racks, CRAH placement, and sensor-to-entity mappings) using configuration files and instantiate these entities and relations in the KG (i.e., site-specific KG schema instantiation).
2.1 Example: layout_map.json
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
2.2 KG Update Hook (Illustrative)
kg = KG.load("kg_base_schema.json")
layout = json.load(open("layout_map.json"))

kg.upsert_entities_from_layout(layout)
kg.upsert_zone_membership(layout["zones"])
kg.upsert_crah_links(layout["crah_units"])
kg.save("kg_site_GPU_HALL_2.json")

3. Airflow Configuration and Rule-Coefficient Re-calibration
Objective. Select the site-appropriate airflow rule pack (e.g., single-sided vs. double-sided) and initialize and re-calibrate key physical/empirical coefficients.
3.1 Example: kg_rules.yaml
airflow_mode: "single_sided"   # or "double_sided"

coefficients:
  # Zone-1 (Cold Aisle)
  hA:           520.0    # effective heat transfer coefficient (W/K)
  eta:          0.08     # leakage factor for non-recoverable heat loss
  c_p:          1005.0   # specific heat capacity of air (J/(kg·K))
  
  # Zone-2 (Hot Aisle)
  n_zones:      4        # number of cold aisle zones for averaging
  
  # Zone-3 (Return)
  delta_t:      60.0     # sampling interval (s)
  T_return_set: 35.0     # return temperature setpoint (°C)
3.2 Rulepack Activation and Coefficient Injection (Illustrative)
rules = load_yaml("kg_rules.yaml")

if rules["airflow_mode"] == "single_sided":
    kg.activate_rulepack("RULEPACK_SINGLE_SIDE")
else:
    kg.activate_rulepack("RULEPACK_DOUBLE_SIDE")

kg.set_rule_coefficients(rules["coefficients"])
kg.save("kg_site_calibrated.json")
3.3 Lightweight Coefficient Calibration
Input: 24–72 hours of site data.
Output: Best-fit coefficients that minimize errors on intermediate targets (e.g., zone temperatures inferred by the KG relative to sensor-derived proxies.
def calibrate_coeffs(data, init_coeffs):
    # minimize error between KG-inferred intermediates and measurements/CFD references
    # e.g., optimize [hA, eta, c_p, n_zones, delta_t, T_return_set] with bounded search
    return best_coeffs

coeffs = calibrate_coeffs(site_data, init_coeffs=rules["coefficients"])
kg.set_rule_coefficients(coeffs)

4. Rule Validation (Intermediate-Feature Quality Control)
Objective. Verify that KG-derived intermediate features are reliable before feeding them into the surrogate model, thereby preventing “garbage-in” physics features.
4.1 Validation Targets
● KG-derived intermediates used by the surrogate: 
● Sanity and monotonicity properties (e.g., increasing IT_load should increase T_hot, all else equal)
4.2 Example: validate_kg.py
pred = kg.infer_intermediates(site_data)     # e.g., T_cold, T_hot, T_return
ref  = load_reference(site_data)             # sensors or small CFD subset

report = {
    "MAE_T_cold":   mae(pred["T_cold"], ref["T_cold"]),
    "MAE_T_hot":    mae(pred["T_hot"], ref["T_hot"]),
    "MAE_T_return": mae(pred["T_return"], ref["T_return"]),
}
assert report["MAE_T_cold"] < thresholds["T_cold"]
save_json("kg_validation_report.json", report)
4.3 Acceptance Criteria
Porting is considered acceptable if:
● intermediate-feature MAE falls within predefined tolerances; and
● sanity/monotonicity checks pass consistently; and
● (optional) sampled alignment against a limited CFD subset or critical measurement points is satisfactory.

5. Minimal Porting Pipeline (Recommended)
Step 1 — Prepare configuration files
● Update normalization.yaml
● Update layout_map.json
Step 2 — Configure airflow mode
● Select airflow_mode
● Provide initial coefficients in kg_rules.yaml
Step 3 — Build, calibrate, and validate the KG
python build_kg.py --layout layout_map.json --out kg_site.json
python calibrate_kg_rules.py --kg kg_site.json --data site_72h.parquet --out kg_site_calibrated.json
python validate_kg.py --kg kg_site_calibrated.json --data site_24h.parquet --report kg_validation_report.json
Step 4 (Optional) — Small-sample surrogate fine-tuning
A short fine-tuning phase may be performed while keeping the network architecture unchanged:
python finetune_surrogate.py --ckpt pretrained.ckpt --kg kg_site_calibrated.json --data site_train.parquet --epochs 1k-5k

