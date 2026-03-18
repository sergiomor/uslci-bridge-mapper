---
name: "uslci-bridge-mapper"
description: "Automate mapping of USLCI elementary flows to biosphere3 in UBW's working_bridge.csv. Use when building the USLCI Brightway database, after a new USLCI quarterly release, or when biosphere exchanges are missing from the built database."
---

# USLCI Bridge Mapper

## What This Skill Does

Analyzes `uslci.csv` for biosphere flows not mapped in UBW's `working_bridge.csv`, matches them against biosphere3 by name and compartment, and appends the new mappings to `working_bridge.csv`. This fixes the gap identified in UBW's future work: "Automate mapping of USLCI -> biosphere3 flows using NLP/AI tools."

## Prerequisites

- Any Brightway project with biosphere3 installed (4,709 flows)
- `uslci.csv` in the UBW directory (the pipeline input file)
- `working_bridge.csv` in the UBW directory (the existing mapping file)
- Python with `bw2data`, `pandas`

## Quick Start

### Step 1: Analyze unmapped flows

```python
import bw2data as bd
import pandas as pd

# Find a project that has biosphere3
for proj in bd.projects:
    bd.projects.set_current(proj.name)
    if "biosphere3" in bd.databases:
        break

# Load USLCI CSV and bridge (paths relative to UBW directory)
df = pd.read_csv("uslci.csv")
bio = df[df["exchange_ecoinvent_type"] == "biosphere"].copy()
bridge = pd.read_csv("working_bridge.csv")
mapped_ids = set(bridge["uslci_id"].astype(str))

# Find unmapped flows
bio["mapped"] = bio["exchange_flow_id"].astype(str).isin(mapped_ids)
unmapped = bio[~bio["mapped"]]
unmapped_flows = unmapped[["exchange_flow_name", "exchange_flow_id", "exchange_category"]].drop_duplicates()
print(f"Unmapped unique flows: {len(unmapped_flows)}")
print(f"Unmapped exchanges: {len(unmapped)}")
```

### Step 2: Build biosphere3 lookup and match

```python
biosphere = bd.Database("biosphere3")

# Build lookup: flow name -> list of candidates
bio_lookup = {}
for flow in biosphere:
    name = flow.get("name", "")
    if name not in bio_lookup:
        bio_lookup[name] = []
    bio_lookup[name].append({
        "code": flow.get("code"),
        "categories": flow.get("categories", ()),
        "unit": flow.get("unit", ""),
    })

# USLCI compartment -> biosphere3 category mapping
# USLCI uses openLCA-style paths: "Elementary flows/emission/air/troposphere/rural"
# biosphere3 uses tuples: ("air", "non-urban air or from high stacks")
COMPARTMENT_MAP = {
    "emission/air/troposphere/rural": ("air", "non-urban air or from high stacks"),
    "emission/air/troposphere/urban": ("air", "urban air close to ground"),
    "emission/air": ("air",),
    "emission/air/stratosphere": ("air", "low population density, long-term"),
    "emission/air/troposphere/high": ("air", "lower stratosphere + upper troposphere"),
    "emission/air/troposphere/very high": ("air", "low population density, long-term"),
    "emission/water/fresh water body": ("water", "surface water"),
    "emission/water/fresh water body/river": ("water", "surface water"),
    "emission/water/fresh water body/lake": ("water", "surface water"),
    "emission/water": ("water",),
    "emission/water/subterranean": ("water", "ground-"),
    "emission/ground": ("soil", "industrial"),
    "emission/ground/human-dominated/agricultural": ("soil", "agricultural"),
    "emission/ground/human-dominated/industrial": ("soil", "industrial"),
    "resource/air": ("natural resource", "in air"),
    "resource/air/subterranean": ("natural resource", "in ground"),
    "resource/ground": ("natural resource", "in ground"),
    "resource/water": ("natural resource", "in water"),
    "resource/water/fresh water body": ("natural resource", "in water"),
    "resource/water/fresh water body/river": ("natural resource", "in water"),
}

def parse_uslci_compartment(category_str):
    """Extract compartment from USLCI category string."""
    if not category_str or not isinstance(category_str, str):
        return None
    return category_str.replace("Elementary flows/", "")

def find_biosphere3_match(flow_name, uslci_category):
    """Find best biosphere3 match for a USLCI flow by name + compartment."""
    if flow_name not in bio_lookup:
        return None

    candidates = bio_lookup[flow_name]
    comp = parse_uslci_compartment(uslci_category)

    # Try mapped compartment match
    if comp and comp in COMPARTMENT_MAP:
        target_cats = COMPARTMENT_MAP[comp]
        # Exact compartment match
        for c in candidates:
            if tuple(c["categories"][:len(target_cats)]) == target_cats:
                return c["code"]
        # Top-level compartment match
        for c in candidates:
            if c["categories"] and c["categories"][0] == target_cats[0]:
                return c["code"]

    # Fallback: match by emission/resource top-level
    if comp:
        top = comp.split("/")[0]
        bio3_top = {"emission": None, "resource": "natural resource"}
        if top in bio3_top and bio3_top[top]:
            for c in candidates:
                if c["categories"] and c["categories"][0] == bio3_top[top]:
                    return c["code"]
        # For emissions, try any candidate in air/water/soil
        if top == "emission":
            for c in candidates:
                if c["categories"] and c["categories"][0] in ("air", "water", "soil"):
                    return c["code"]

    # Last resort: first candidate
    if candidates:
        return candidates[0]["code"]
    return None
```

### Step 3: Fuzzy matching for flows with no exact name match

For flows where the exact name doesn't exist in biosphere3, apply AI-assisted fuzzy matching. Common patterns:

```python
# Known name mappings: USLCI name -> biosphere3 name
# These handle spelling variants, US/UK English, synonyms, and naming conventions
FUZZY_NAME_MAP = {
    # US vs UK spelling
    "Aluminum": "Aluminium",
    "Aluminum(III)": "Aluminium",
    "Cesium": "Caesium",
    "Cesium-134": "Caesium-134",
    "Cesium-137": "Caesium-137",
    # Speciation needed
    "Methane": "Methane, fossil",
    "Carbon dioxide": "Carbon dioxide, fossil",
    "Carbon monoxide": "Carbon monoxide, fossil",
    # Naming conventions (USLCI -> biosphere3)
    "Particulate matter, \u2264 10\u03bcm": "Particulates, < 10 um",
    "Particulate matter": "Particulates, < 2.5 um",
    "Particulate matter, > 10\u03bcm": "Particulates, > 10 um",
    "Crude oil": "Oil, crude, in ground",
    "Natural gas": "Gas, natural/m3",
    "Water, fresh": "Water, unspecified natural origin/kg",
    "Water, saline": "Water, salt, ocean",
    # Common vs IUPAC names
    "Propylene": "Propene",
    "Propionaldehyde": "Propanal",
    "Methylene chloride": "Dichloromethane",
    "Ethylene glycol": "Ethylene glycol",
    "Ethylene dibromide": "Ethane, 1,2-dibromo-",
    "Isopropanol": "2-Propanol",
    "Isobutanol": "Isobutanol",
    "Methyl tert-butyl ether": "Methyl tert-butyl ether",
    "Vinyl acetate": "Vinyl acetate",
    # Aggregate -> specific
    "Volatile organic compounds": "NMVOC, non-methane volatile organic compounds, unspecified origin",
    "Polycyclic aromatic hydrocarbons": "PAH, polycyclic aromatic hydrocarbons",
    "Hydrocarbons": "Hydrocarbons, unspecified",
    "Aromatic hydrocarbons": "Hydrocarbons, aromatic",
    "Orthophosphate": "Phosphate",
    "Dioxins": "Dioxin, 2,3,7,8 Tetrachlorodibenzo-p-",
    # Refrigerants / CFCs
    "CFC-11": "Methane, trichlorofluoro-, CFC-11",
    "CFC-12": "Methane, dichlorodifluoro-, CFC-12",
    "HFC-245fa": "Ethane, 1,1,1,3,3-pentafluoro-, HFC-245fa",
    # Bracket vs parenthesis
    "Benzo[ghi]perylene": "Benzo(ghi)perylene",
    "Benz[a]anthracene": "Benz(a)anthracene",
    "Benzo[k]fluoranthene": "Benzo(k)fluoranthene",
    "Benzo[b]fluoranthene": "Benzo(b)fluoranthene",
    "Benzo[a]pyrene": "Benzo(a)pyrene",
    "Indeno[1,2,3-cd]pyrene": "Indeno(1,2,3-cd)pyrene",
    "Dibenz[a,h]anthracene": "Dibenz(a,h)anthracene",
    "Benzofluoranthene": "Benzo(b)fluoranthene",
    # Chemical synonyms
    "Hydrofluoric acid": "Hydrogen fluoride",
    "Hydrogen cyanide": "Cyanide",
    "Cresol": "m-Cresol",
    "Chlorobenzene": "Chlorobenzene",
    "Pentachlorophenol": "Pentachlorophenol",
    "Caprolactam": "Caprolactam",
    "Ethanolamine": "Ethanol amine",
    "Biphenyl": "Diphenyl",
    "Dibenzofuran": "Dibenzofuran",
    "Tritium": "Hydrogen-3, Tritium",
    "Hydrogen bromide": "Hydrogen bromide",
    "Ethylbenzene": "Ethylbenzene",
    # Parentheses/oxidation state removal
    "Chromium(III)": "Chromium III",
    "Chromium(VI)": "Chromium VI",
    "Iron(III)": "Iron",
    "Manganese(II)": "Manganese",
    "Cobalt(II)": "Cobalt",
    "Arsenic(V)": "Arsenic",
    "Arsenic(III) trioxide": "Arsenic",
    "Copper(I)": "Copper",
    "Copper(II)": "Copper",
}

def find_fuzzy_match(flow_name, uslci_category):
    """Try fuzzy matching for flows not found by exact name."""
    # 1. Check known name mappings
    if flow_name in FUZZY_NAME_MAP:
        mapped_name = FUZZY_NAME_MAP[flow_name]
        if mapped_name in bio_lookup:
            return find_biosphere3_match(mapped_name, uslci_category)

    # 2. Try removing parentheses: "Chromium(III)" -> "Chromium III"
    cleaned = flow_name.replace("(", " ").replace(")", "").strip()
    cleaned = " ".join(cleaned.split())  # normalize whitespace
    if cleaned in bio_lookup:
        return find_biosphere3_match(cleaned, uslci_category)

    # 3. Try replacing brackets with parentheses: "Benzo[a]pyrene" -> "Benzo(a)pyrene"
    bracket_fixed = flow_name.replace("[", "(").replace("]", ")")
    if bracket_fixed in bio_lookup:
        return find_biosphere3_match(bracket_fixed, uslci_category)

    # 4. Try adding common suffixes: "Methane" -> "Methane, fossil"
    for suffix in [", fossil", ", biogenic", ", in ground", ", ion"]:
        candidate = flow_name + suffix
        if candidate in bio_lookup:
            return find_biosphere3_match(candidate, uslci_category)

    # 5. Try Greek letter prefixes: ".gamma.-Butyrolactone" -> "Butyrolactone"
    if flow_name.startswith("."):
        # Remove ".greek.-" prefix
        parts = flow_name.split("-", 1)
        if len(parts) > 1:
            stripped = parts[1]
            if stripped in bio_lookup:
                return find_biosphere3_match(stripped, uslci_category)

    return None
```

### Step 4: Generate new mappings and append to working_bridge.csv

```python
new_mappings = []
unresolved = []

for _, row in unmapped_flows.iterrows():
    # Try exact match first
    bio3_code = find_biosphere3_match(row["exchange_flow_name"], row["exchange_category"])

    # Try fuzzy match if exact fails
    if bio3_code is None:
        bio3_code = find_fuzzy_match(row["exchange_flow_name"], row["exchange_category"])

    if bio3_code:
        new_mappings.append({
            "uslci_id": row["exchange_flow_id"],
            "biosphere_id": bio3_code,
        })
    else:
        unresolved.append({
            "flow_name": row["exchange_flow_name"],
            "flow_id": row["exchange_flow_id"],
            "category": row["exchange_category"],
        })

print(f"Resolved: {len(new_mappings)} new mappings")
print(f"Unresolved: {len(unresolved)} flows (no biosphere3 match by name)")

# Append to working_bridge.csv (preserving existing mappings)
if new_mappings:
    new_df = pd.DataFrame(new_mappings)
    # Deduplicate against existing bridge
    existing_ids = set(bridge["uslci_id"].astype(str))
    new_df = new_df[~new_df["uslci_id"].astype(str).isin(existing_ids)]
    print(f"After dedup: {len(new_df)} truly new mappings to add")

    if len(new_df) > 0:
        updated_bridge = pd.concat([bridge, new_df], ignore_index=True)
        updated_bridge.to_csv("working_bridge.csv", index=False)
        print(f"Updated working_bridge.csv: {len(bridge)} -> {len(updated_bridge)} mappings")

# Report unresolved for manual review
if unresolved:
    unresolved_df = pd.DataFrame(unresolved)
    unresolved_df.to_csv("unresolved_flows.csv", index=False)
    print(f"Wrote unresolved_flows.csv for manual review ({len(unresolved)} flows)")
```

### Step 5: Verify improvement

After updating `working_bridge.csv`, rebuild the database with `backup_plan.py` (or `prepare_uslci.py`) and check the biosphere exchange count:

```
Before: Biosphere: 34,366 exchanges added, 8,204 missing/failed
After:  Biosphere: [should be higher], [should be lower]
```

---

## How It Works

The UBW pipeline (`backup_plan.py`) uses `working_bridge.csv` to map USLCI elementary flow IDs to biosphere3 UUIDs. Flows not in the bridge are dropped during database construction. The main cause of missing mappings is **compartment variants** — the same substance (e.g., Methane) exists under multiple USLCI sub-compartments (`air/troposphere/rural`, `air`, `air/stratosphere`) but only some variants are mapped in the bridge.

This skill:
1. Reads `uslci.csv` to find all biosphere exchanges
2. Identifies which USLCI flow IDs are missing from `working_bridge.csv`
3. Searches biosphere3 for matching flows by name + compartment
4. Appends new mappings to `working_bridge.csv`
5. Reports unresolved flows for manual review

The process is **idempotent** — it deduplicates against existing bridge entries before appending. Running it multiple times or after UBW updates is safe.

## When to Run

- After downloading a new USLCI quarterly release (new flows may be introduced)
- After cloning or updating the UBW repository (bridge may have new mappings, reducing what needs patching)
- When LCIA results from USLCI processes seem low (missing biosphere flows underestimate impacts)

## Compartment Mapping Reference

| USLCI Compartment | biosphere3 Category |
|---|---|
| `emission/air/troposphere/rural` | `("air", "non-urban air or from high stacks")` |
| `emission/air/troposphere/urban` | `("air", "urban air close to ground")` |
| `emission/air` | `("air",)` |
| `emission/air/stratosphere` | `("air", "low population density, long-term")` |
| `emission/water/fresh water body` | `("water", "surface water")` |
| `emission/water` | `("water",)` |
| `emission/water/subterranean` | `("water", "ground-")` |
| `emission/ground` | `("soil", "industrial")` |
| `resource/air` | `("natural resource", "in air")` |
| `resource/ground` | `("natural resource", "in ground")` |
| `resource/water` | `("natural resource", "in water")` |

## Troubleshooting

### No biosphere3 match found
The flow name in USLCI doesn't match any biosphere3 flow name exactly. Common causes:
- USLCI-specific naming (e.g., "Trihalomethanes (four), total" has no biosphere3 equivalent)
- Spelling differences or chemical name variants
- These flows are written to `unresolved_flows.csv` for manual review

### Compartment not in COMPARTMENT_MAP
A USLCI category path not covered by the mapping table. Add the new compartment to `COMPARTMENT_MAP` with the appropriate biosphere3 category tuple and re-run.

### Bridge already up to date
If all flows are already mapped, the skill reports "0 truly new mappings to add" and does not modify `working_bridge.csv`.

## Unfixable Flows

Some USLCI flows have no biosphere3 equivalent and cannot be mapped. These are aggregate flows (Metals, Radionuclides, Surfactants), non-characterized flows (Air, Rock, Energy heat), and USLCI-specific inventory items. Their absence does not affect LCIA scores because no LCIA method assigns characterization factors to them.

See [README.md](README.md) for the complete list with explanations.
