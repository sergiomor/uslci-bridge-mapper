# USLCI Bridge Mapper

A Claude Code skill that automates mapping of USLCI elementary flows to ecoinvent's biosphere3 flow list. Addresses the UBW project's future work item: "Automate mapping of USLCI -> biosphere3 flows using NLP/AI tools."

## Problem

The [UBW pipeline](https://github.com/NatLabRockies/UBW) converts the USLCI database into Brightway format. During this conversion, biosphere exchanges (emissions, resources) are linked to biosphere3 using a manual mapping file (`working_bridge.csv`). Flows not in the bridge are dropped, leading to:

- Incomplete LCI inventories
- Underestimated LCIA scores (missing GHG emissions, particulate matter, etc.)
- ~1,495 unmapped unique flows affecting ~14,582 exchanges in a typical USLCI build

## What the Skill Does

1. **Exact name + compartment matching**: Finds USLCI flows that exist in biosphere3 under the same name but different compartment paths (e.g., Methane in `air/stratosphere` vs `air/troposphere/rural`)
2. **Fuzzy matching**: Resolves naming differences between USLCI and biosphere3:
   - US vs UK spelling (Aluminum -> Aluminium, Cesium -> Caesium)
   - Common vs IUPAC names (Propylene -> Propene, Methylene chloride -> Dichloromethane)
   - Fossil/biogenic speciation (Methane -> Methane, fossil)
   - Bracket/parenthesis variants (Benzo[a]pyrene -> Benzo(a)pyrene)
   - Oxidation state notation (Chromium(III) -> Chromium III)
   - CFC trade names (CFC-11 -> Methane, trichlorofluoro-, CFC-11)
   - Aggregate -> specific (VOC -> NMVOC, PAH -> PAH polycyclic aromatic hydrocarbons)
3. **Reports unresolved flows** to `unresolved_flows.csv` for manual review

## Results

Tested against USLCI 2025 Q1 (1,164 processes, 55,852 biosphere exchanges):

| Stage | Mappings added | Total in bridge | Unresolved flows |
|-------|---------------|-----------------|------------------|
| Original `working_bridge.csv` | - | 2,424 | 1,495 |
| + Exact matching | +708 | 3,132 | 787 |
| + Fuzzy matching | +118 | 3,250 | 669 |
| **Total improvement** | **+826** | **3,250** | **669** |

Coverage improved from 62% to 76% of unique biosphere flows. The 669 remaining unresolved flows are truly unfixable (see below).

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed
- A Brightway project with biosphere3 (4,709 flows) — created by running `backup_plan.py` or `bw2setup()`
- `uslci.csv` in the UBW directory
- `working_bridge.csv` in the UBW directory
- Python with `bw2data`, `pandas`

## Usage

### 1. Install the skill

Copy the `uslci-bridge-mapper` directory to your Claude Code skills location:

```bash
# Project-level (recommended for UBW repos)
cp -r uslci-bridge-mapper .claude/skills/

# Or personal (available across all projects)
cp -r uslci-bridge-mapper ~/.claude/skills/
```

### 2. Run the UBW pipeline first

The skill needs biosphere3 to exist. Run the pipeline at least once:

```bash
python backup_plan.py
```

### 3. Invoke the skill

In Claude Code, run:

```
/uslci-bridge-mapper
```

Claude will execute the matching steps and update `working_bridge.csv` with new mappings.

### 4. Rebuild with updated bridge

Re-run the pipeline to build the database with the improved bridge:

```bash
python backup_plan.py
```

The biosphere exchange count should increase and missing/failed count should decrease.

### 5. Review unresolved flows

Check `unresolved_flows.csv` for flows that could not be mapped. Most are truly unfixable (see below), but some may be resolvable by adding entries to the `FUZZY_NAME_MAP` in `SKILL.md`.

## Idempotent and Safe

- The skill deduplicates against existing bridge entries before appending
- Running it multiple times produces the same result
- If UBW updates `working_bridge.csv` with new mappings, the skill skips those flows automatically
- `working_bridge.csv` is only appended to, never overwritten

## Unfixable Flows

669 USLCI flows have no biosphere3 equivalent. Their absence does **not** affect LCIA scores because no standard LCIA method (IPCC, ReCiPe, EF, TRACI) assigns characterization factors to them.

### Aggregate Flows

Flows representing groups of substances. biosphere3 requires individual speciation.

| USLCI Flow | Exchanges | Why |
|---|---|---|
| Metals | ~127 | biosphere3 tracks individual metals (Chromium, Lead, etc.) |
| Radionuclides | ~124 | biosphere3 tracks individual isotopes (Caesium-137, etc.) |
| Volatile organic compounds | ~725 | biosphere3 uses NMVOC (already mapped) plus individual compounds |
| Polycyclic aromatic hydrocarbons | ~160 | biosphere3 tracks individual PAH compounds |
| Surfactants | ~38 | biosphere3 tracks specific surfactant compounds |
| Chlorinated organic compounds | ~29 | biosphere3 tracks individual chlorinated compounds |
| Organic compounds | ~95 | biosphere3 tracks individual organics |
| Total Organic Carbon | ~152 | Aggregate water quality parameter |

### Non-Characterized Flows

Flows that biosphere3 does not classify because no LCIA method uses them.

| USLCI Flow | Exchanges | Why |
|---|---|---|
| Land use | ~443 | USLCI uses generic "Land use"; biosphere3 uses specific "Occupation, ..." and "Transformation, ..." by land type |
| Organic carbon, PM2.5 LC | ~232 | USLCI-specific particulate fraction |
| Energy, heat | ~109 | Waste heat, not in biosphere3 |
| Biomass | ~57 | Generic biotic resource, biosphere3 uses specific types |
| Air | ~45 | Not an emission or resource in LCA terms |
| Rock | ~35 | Generic geological material |
| Oil sand | ~30 | USLCI-specific resource category |
| Limestone | ~26 | Construction material as resource |
| Oils, biogenic | ~28 | Uses non-FEDEFL compartment path |

### Notes

- Exchange counts are approximate and vary by USLCI release
- "Land use" is the largest unfixable category (443 exchanges) but could potentially be mapped with domain expertise per land type sub-compartment
- The aggregate flows would require disaggregation data that USLCI does not provide

## Compartment Mapping Reference

The skill maps USLCI openLCA-style compartment paths to biosphere3 category tuples:

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

## License

MIT
