# F280_COMPLIANCE_SPEC.md
## CSA F280:12 — Compliance Specification for Residential Heating and Cooling Load Calculations

**Source:** CSA F280:12, *Determining the Required Capacity of Residential Space Heating and Cooling
Appliances*, National Standard of Canada, published March 2012, reaffirmed 2025.
**OCR source:** `F280_OCR_RAW.md` (commit 3742a2c), licensed to ben@stationcraft.ca, Station Craft Inc.
**Purpose of this document:** Plain-language digest of F280's normative requirements for use as a
contract by the Collabinator main track. Clause citations in parentheses trace every requirement back
to the standard. This document describes what F280 requires — it does not prescribe how Collabinator's
model sources or stores the data. Resolution of implementation choices belongs in the main planning chat.

---

## 1. SCOPE

### What F280:12 Covers (Cl. 1.1–1.3)

F280 provides the calculation method for determining the **design heating load** and **design cooling
load** of a residential building, for the sole purpose of selecting correctly sized space heating and
cooling appliances. It applies to:

- **Building types:** Housing and small buildings of residential occupancy governed by **Part 9 of the
  National Building Code of Canada** (NBC Part 9). This covers single-family homes, semi-detached,
  townhouses, and small multi-unit residential buildings within NBC Part 9's scope.
- **Appliance types:** All permanently installed space heating appliances (gas, electric, heat pump,
  hydronic, etc.) and cooling appliances (central air, heat pump, etc.) within the dwelling they serve.
- **Output:** A room-by-room heat loss (W) and heat gain (W) calculation, summed to a whole-building
  design load, which sets the minimum required capacity of the installed heating and cooling systems.
- **Software systems (Cl. 1.3A, 1.3B, Cl. 8):** F280 also provides a verification procedure for
  software systems that claim to follow the F280 calculation method.
- **Units (Cl. 1.5):** Requirements are expressed in SI (metric). Clause 8 and Annexes G–K express
  requirements in both imperial and SI.

### What F280:12 Does NOT Cover (Cl. 1.4)

- Detailed design or installation of heating/cooling distribution systems (duct sizing, pipe sizing, etc.)
- Equipment beyond the Part 9 NBC residential scope
- Commercial, industrial, or large multi-unit residential buildings
- Mechanical ventilation system design (that is governed by CAN/CSA-F326)

---

## 2. CALCULATION MODEL

### 2.1 Design Conditions (Cl. 4)

**Outdoor conditions (Cl. 4.1):** Use local building code design tables for the building's location. In
the absence of local code tables, use the values in Table 5 of F280 (a sample; the full dataset is in the
F280_Weather.xls spreadsheet). Key weather parameters per location:
- Outdoor design dry bulb temperature, heating: **Toh** (°C)
- Outdoor design dry bulb temperature, cooling: **Toc** (°C)
- Summer mean daily temperature range: **STrange** (°C) — used for solar correction (Table 3)
- Deep ground temperature: **DGTEMP** (°C) — required by BASESIMP
- Monthly average dry bulb temperatures — required by BASESIMP
- Design humidity difference: **DHD** (g/kg) — for latent load multiplier
- January and July average wind speeds — required by AIM-2
- Heating degree days — required by BASESIMP

**Indoor conditions (Cl. 4.2):** Use values specified by the local building code or NBC. In the absence
of a code specification:
- Heating: **Tin** = 21 °C for most spaces (15 °C for heated crawl spaces). Use of 22 °C is acceptable.
- Cooling: **Tic** = 24 °C (typical; per NBC or local regulation)

**Unheated enclosed spaces (Cl. 4.3):** Assume at the outdoor design temperature.

### 2.2 Heating Design Temperature Difference (Cl. 5.1)

```
DTDh = Tin − Toh
```
- **DTDh** = design temperature difference for heating, °C
- **Tin** = indoor design temperature for heating, °C
- **Toh** = outdoor design temperature for heating, °C

*When radiant heating is in the above-grade walls or ceilings, substitute the circulating fluid
temperature at steady state for Tin.*

### 2.3 Heating Heat-Loss Components

#### 2.3.1 Above-Grade Conductive Heat Loss (Cl. 5.2.1)

For each room, for each above-grade building assembly (wall, roof, window, door, exposed floor):

```
HLage = A / RSI × DTDh
```
- **HLage** = above-grade conductive heat loss for one assembly, W
- **A** = net area of the building assembly, m² (rough opening dimensions for windows and doors)
- **RSI** = thermal resistance of the assembly, m²·°C/W (from Tables 6A–6H, 7A, 7B, or by test/manufacturer)
- **DTDh** = design temperature difference for heating, °C

Room total:
```
HLager = Σ HLage (all above-grade assemblies bounding the room)
```

#### 2.3.2 Below-Grade Conductive Heat Loss — BASESIMP Sub-Calculator (Cl. 5.2.2)

For any room with envelope elements in contact with soil, below-grade heat loss is not calculated by
the simple A/RSI formula. It is calculated by the **BASESIMP** algorithm, implemented in two
spreadsheets linked to F280:

- **Basement walls and slab below grade → BasementHLR.xls** (Cl. 5.2.2.1)
- **Slab on grade → SlabOnGradeHLR.xls** (Cl. 5.2.2.2)

BASESIMP is a simulation-derived polynomial algorithm (Beausoleil-Morrison & Mitalas, 1997) covering
145 foundation configurations. Its output is the below-grade conductive heat loss in January:

```
HLbgcr = output of BASESIMP spreadsheet for the foundation element, W
```

Required inputs are listed in Section 3 (REQUIRED DATA FIELDS) below.

*Note: Below-grade floor and wall elements that are NOT in contact with soil (e.g., basement walls
above grade, exposed floor over open crawlspace) use the above-grade formula (Cl. 5.2.1).*

#### 2.3.3 Air Change Heat Loss (Cl. 5.2.3)

Air change = air leakage + continuous ventilation. Calculated in two sub-parts:

**Part A — Air leakage: AIM-2 Sub-Calculator (Cl. 5.2.3.1)**

Air leakage is calculated by the **AIM-2 model** (Walker & Wilson, 1990), implemented in the
spreadsheet **AIM2.xls** linked to F280. AIM-2 accounts for building height and exposure, wind speed,
stack effect, and ventilation system pressurization. Its output:

```
LRairh = envelope air leakage rate for heating, AC/H
```

The building-level air leakage heat loss:
```
HLairb = LRairh × Vb × DTDh / 3.6
```
- **Vb** = volume of the building, measured on inside dimensions, m³
- **DTDh** = design temperature difference for heating, °C

(The /3.6 conversion factor converts AC/H × m³ × °C into W, using the volumetric heat capacity
of air ≈ 1.2 kJ/m³·°C.)

*In the absence of a blower door test result, AIM-2 provides prescriptive air tightness defaults.
Where large concentrated air barrier flaws exist (e.g., rooms above garages), see Annex C.*

**Part B — Continuous mechanical ventilation (Cl. 5.2.3.2)**

```
HLvairb = PVC × DTDh × 1.2 × (1 − E)
```
- **PVC** = principal continuous ventilation rate, L/s (40–60% of MVC per CAN/CSA-F326, or per local code)
- **E** = apparent sensible effectiveness of the HRV at design temperature and PVC rate (per CAN/CSA-C439);
  use E = 0 if no heat recovery ventilator
- **HLvairb** = heat loss from continuous ventilation for the whole building, W

**Part C — Room allocation (Cl. 5.2.3.3)**

```
HLairr = Level_factor × HLairbv × (HLager + HLbgcr) / (HLagclevel + HLbgclevel)
```
- **HLairbv** = HLairb + HLairve (where HLairve = unbalanced exhaust rate × DTDh × 1.2; = 0 for balanced systems)
- **Level_factor** = factor from the level-factor table in Cl. 5.2.3.3 (varies by number of building levels
  and whether the room is at slab level, main level, or upper level)
- Subscript *level* denotes the sum across all rooms on the same floor level

When a separate ventilation duct serves known rooms at known flow rates:
```
HLvairr = 1.2 × Qvr × DTDh × (1 − E)
```

#### 2.3.4 Duct Heat Loss — Air Systems Only (Cl. 5.2.4)

For each room served by ducts through unconditioned space or adjacent to slab perimeter:
```
HLdr = DLMh × (HLager + HLbgcr + HLairr)
```
- **DLMh** = duct loss multiplier from Table 1 (based on duct location and duct insulation RSI)
- Key values from Table 1: attic/open crawlspace uninsulated → DLMh = 0.25; RSI ≥ 3.50 → 0.05;
  slab perimeter (all insulation values) → 0.10

#### 2.3.5 Pipe Heat Loss — Hydronic Systems Only (Cl. 5.2.5)

For each room served by heat distribution piping through unconditioned space:
```
HLpr = PLMh × (HLager + HLbgcr + HLairr)
```
- **PLMh** = pipe loss multiplier from Table 2 (based on pipe type and the ratio of total heat loss to floor area)

#### 2.3.6 Total Room Heat Loss (Cl. 5.2.6)

```
HLr = HLager + HLbgcr + HLairr + [HLdr OR HLpr] + HLvairr
```
Apply either the duct term (air system) or pipe term (hydronic system), not both. HLvairr applies only
where ventilation is distributed room-by-room by a separate duct system.

#### 2.3.7 Total Building Heat Loss (Cl. 5.2.7)

```
HLb = Σ HLr (all rooms inside the building envelope)
```
*If ventilation heat loss was not added per-room, add HLvairb to HLb.*

### 2.4 Heating System Sizing Limits (Cl. 5.3)

- **Building capacity:** Total installed heating output ≥ **100% of HLb** (Cl. 5.3.1)
- **Room delivery:** Combined heating delivery to each room ≥ **100% of HLr** (Cl. 5.3.2)

### 2.5 Cooling Design Temperature Difference (Cl. 6.1.1)

```
DTDc = Toc − Tic
```
- **DTDc** = design temperature difference for cooling, °C
- **Toc** = outdoor design temperature for cooling, °C
- **Tic** = indoor design temperature for cooling, °C

*Only rooms served by ducts connected to the cooling system (and not intended to be shut off from it)
are included in cooling calculations (Cl. 6.1.2).*

### 2.6 Cooling Sensible Heat Gain Components

#### 2.6.1 Opaque Building Assemblies (Cl. 6.2.1)

```
HGcop = A / RSI × (DTDc + SC)
```
- **HGcop** = conductive heat gain through opaque assembly, W
- **SC** = solar correction (°C) from Table 3, based on assembly orientation and summer daily temperature range

Table 3 key values (SC in °C):
- Dark-coloured roofs / east or west walls: +6.7 (STrange ≤ 14 °C) / +11.1 (STrange > 14 °C)
- Light-coloured roofs: +4.4 / +7.8
- Medium-colour walls: +3.3 / +6.1
- North-facing or fully shaded walls/doors: −3.3 / −6.1
- Floors/ceilings over/under non-conditioned rooms: −3.3 / −6.1

*Heat gains through floors or walls at or below grade shall NOT be considered in cooling (Cl. 6.2.1).*

#### 2.6.2 Windows and Translucent Assemblies (Cl. 6.2.2)

```
HGet = A × (SHGC × Solar + DTDc / RSIw)
```
- **HGet** = combined conductive and solar heat gain through window/glazing, W
- **A** = area, m² (rough opening per Table notes)
- **SHGC** = solar heat gain coefficient (from Tables 6E–6H or manufacturer; same source as RSIw)
- **Solar** = solar radiation incident on the window, W/m² — see below
- **RSIw** = whole-window thermal resistance, m²·°C/W (from Tables 6E–6H or manufacturer)
- **DTDc** = cooling design temperature difference, °C

**Solar radiation selection:**
Base values from the table in Cl. 6.2.2 by orientation (N, S, E/W, NE/NW, SE/SW, Horizontal).
Latitude correction factor:
```
LFactor = 1 + (Latitude − 40) × 0.0375     [clamped to range 0.7–1.3]
```
LFactor applies **only to South and SE/SW windows**; all other orientations use LFactor = 1.
```
Solar = Solaro × LFactor
```

**Interior shading (Cl. 6.2.2.2):**
```
Solar = Solaro × LFactor × SFactor
```
SFactor from Table 4 (e.g., interior reflective blinds ≈ 0.35–0.37 depending on glazing type).
Shaded window areas shall be treated as north-facing.

#### 2.6.3 Internal Gains — People (Cl. 6.2.4)

```
HGsp = 70 W × number_of_occupants
```
Assign to rooms normally occupied at peak load (living, family, dining). If occupant count unknown,
use (number of bedrooms + 1).

#### 2.6.4 Internal Gains — Appliances, Lights, Plug Loads (Cl. 6.2.5)

```
HGiae = MAX(4 W/m² × Afloorb, 800 W)
```
- **Afloorb** = gross floor area of the building, m²
Assign to rooms normally occupied at peak load (family, living, kitchen).

#### 2.6.5 Air Leakage — AIM-2 Sub-Calculator (Cl. 6.2.6)

```
LRairc = cooling air leakage rate, AC/H  (from AIM-2 spreadsheet)
HGsaib = LRairc × Vb × DTDc / 3.6
```
Room allocation:
```
HGsair = HGsaib × (HGcr / HGcb)
```
where HGcb is the total conductive heat gain for the whole building.

#### 2.6.6 Mechanical Ventilation (Cl. 6.2.7)

```
HGsvb = PVC × DTDc × 1.2 × (1 − ATRE)
```
- **ATRE** = adjusted total recovery efficiency of the HRV/ERV (per CAN/CSA-C439); 0 if none
Room allocation (when room flow rates are known):
```
HGsvr = 1.2 × Qvr × DTDc
```
Otherwise allocate proportionally to conductive heat gain ratio.

#### 2.6.7 Duct Gain (Cl. 6.2.8)

For each room served by ducts through unconditioned space:
```
HGdr = DGMc × (HGcr + HGsp* + HGiae* × Afloorb + HGsair + HGsvr*)
```
- **DGMc** = duct gain multiplier from Table 1 (* if applicable to the room)

#### 2.6.8 Total Room Sensible Heat Gain (Cl. 6.2.9)

```
HGsr = HGcr + HGsp* + HGiae* × Afloorb + HGsair + HGdr + HGsvr
```

### 2.7 Cooling System Sizing Limits (Cl. 6.3)

```
CSCn = Lu × Σ HGsr (all rooms to be cooled)
```
- **Lu** = latent load multiplier = 1.3 in most applications (higher for humid climates per Annex B)
- **CSCn** = nominal cooling system capacity, W

**Minimum installed capacity:** ≥ max(0.80 × CSCn, CSCn − 1800 W) (Cl. 6.3.2)
**Maximum installed capacity:** ≤ min(1.25 × CSCn, CSCn + 1750 W if CSCn < 6000 W) (Cl. 6.3.4–6.3.5)
*Ground/water-source heat pumps used for cooling are exempt from the 125% upper limit (Cl. 6.3.4).*

---

## 3. REQUIRED DATA FIELDS, SURFACE BY SURFACE

This section is the checklist against which the main model's surface data shall be verified.
Every item is required by F280 to compute that surface's contribution to the load.

### 3.1 Above-Grade Opaque Wall

| Field | Unit | Source |
|---|---|---|
| Net area (after deducting rough openings) | m² | Drawing take-off |
| RSI value (total assembly, adjusted for framing) | m²·°C/W | Tables 6A–6D + 7A, or test data |
| Orientation (N/S/E/W/NE/NW/SE/SW or interior) | — | Drawing / compass |

*RSI calculated as: RSI_uninsulated (Table 6A) + RSI_cavity (Table 6B) + RSI_exterior/interior (Table 6D).
Framing losses shall be included using parallel path method (Table 7A).*

### 3.2 Roof / Ceiling

| Field | Unit | Source |
|---|---|---|
| Net area | m² | Drawing take-off |
| RSI value (adjusted for framing) | m²·°C/W | Tables 6A + 6C + 6D, or test data |
| Slope / orientation for cooling SC | — | Drawing (horizontal for flat; pitched for sloped) |

*Note: Vented attic → RSI_uninsulated = 0.23; unvented/cathedral → 0.58 (Table 6A).*

### 3.3 Exposed Floor (Above Unconditioned Space)

Applies to floors over open or vented crawlspaces, unheated garages, cantilevers, etc.
(Cl. 5.2.1; treated as above-grade, not below-grade.)

| Field | Unit | Source |
|---|---|---|
| Net area | m² | Drawing take-off |
| RSI value (adjusted for framing) | m²·°C/W | Tables 6A + 6C + 6D, or test data |

*Not included in cooling heat gain per Cl. 6.2.1.*

### 3.4 Rim Joist (Floor Header)

| Field | Unit | Source |
|---|---|---|
| Area (length × height of header band) | m² | Drawing take-off |
| RSI value | m²·°C/W | Table 7A.2 (adjusted joist header values) |

### 3.5 Windows

| Field | Unit | Source | Notes |
|---|---|---|---|
| Area (rough opening) | m² | Drawing take-off | Per Table 6E–6H note: rough opening, not frame |
| RSI_W (whole-window thermal resistance) | m²·°C/W | Tables 6E–6H **or** manufacturer CAN/CSA-A440 / A440.2 / CGSB 82.1 test data | See Section 4 |
| SHGC (solar heat gain coefficient) | — | Same source as RSI_W | See Section 4 |
| Orientation | N/S/E/W/NE/NW/SE/SW or H | Drawing / compass | Required for solar calc |
| Interior shading type | — | Spec / assumption | From Table 4; none = SFactor not applied |
| Shaded vs unshaded area split | m² | Drawing / shadow calc | Shaded area → north-facing per Annex A |

### 3.6 Opaque Doors

| Field | Unit | Source |
|---|---|---|
| Area (rough opening) | m² | Drawing take-off |
| RSI value | m²·°C/W | Table 6I **or** manufacturer CAN/CGSB-82.5 test data |
| Orientation | N/S/E/W/NE/NW/SE/SW | Drawing / compass |
| Storm door present? | Yes/No | Spec |

*Table 6I provides two categories: panel-type wood (32 mm thick) and insulated fiberglass/polystyrene
core, each with and without a storm door. Use rough opening dimensions.*

### 3.7 Glazed Doors (Glazed Portions of Doors)

Treated identically to windows per Cl. 6.2.2:

| Field | Unit | Source |
|---|---|---|
| Glazed area (rough opening or measured glazed area) | m² | Drawing take-off |
| RSI_W | m²·°C/W | Tables 6E–6H or manufacturer |
| SHGC | — | Same source as RSI_W |
| Orientation | N/S/E/W/NE/NW/SE/SW | Drawing / compass |

### 3.8 Below-Grade Basement Walls and Slab — BASESIMP Inputs (Cl. 5.2.2.1)

Required inputs to BasementHLR.xls (BASESIMP algorithm):

| Field | Unit | Notes |
|---|---|---|
| Foundation configuration number | — | From 145 configs in FoundationCC.xls; BSMCFG.TXT has descriptions |
| Basement length, width, height | m | Interior dimensions |
| Depth below grade | m | Of wall base |
| Soil conductivity | W/m·°C | As suggested in spreadsheet |
| Water table level | m | Below grade |
| RSI of interior added insulation (vertical) | m²·°C/W | Above and below grade if split |
| RSI of exterior added insulation (vertical) | m²·°C/W | |
| Insulation overlap (extent of exterior over interior) | m | |
| RSI of slab insulation | m²·°C/W | 0 if none |
| Number of corners | count | |
| Heating degree days | °C·days | From local climate data |
| Deep ground temperature | °C | From Table 5 / F280_Weather.xls |
| Monthly average dry bulb temperatures | °C × 12 | From climate data |
| Room temperature | °C | Tin |
| Outdoor design dry bulb temperature, heating | °C | Toh |
| Outdoor design dry bulb temperature, cooling | °C | Toc |
| Slab temperature | °C | Working fluid temp for in-floor hydronic; 22–50 °C for unheated slab |
| Exposed perimeter (attached houses) | m | Length of wall not shared with adjacent unit |
| Areas of windows and doors in above-grade basement walls | m² | Per wall face |

### 3.9 Slab-on-Grade — BASESIMP Inputs (Cl. 5.2.2.2)

Required inputs to SlabOnGradeHLR.xls:

| Field | Unit | Notes |
|---|---|---|
| Foundation configuration number | — | From FoundationCC.xls |
| Slab length, width | m | |
| Soil conductivity | W/m·°C | |
| Water table level | m below grade | |
| RSI of slab insulation | m²·°C/W | 0 if none |
| Number of corners | count | |
| Heating degree days | °C·days | |
| Deep ground temperature | °C | |
| Monthly average dry bulb temperatures | °C × 12 | |
| Room temperature | °C | Tin |
| Outdoor design dry bulb, heating | °C | Toh |
| Outdoor design dry bulb, cooling | °C | Toc |
| Slab temperature | °C | Working fluid for hydronic; 22–50 °C for unheated |
| Exposed perimeter (attached houses) | m | |

### 3.10 Air Leakage / Infiltration — AIM-2 Inputs (Cl. 5.2.3.1, 6.2.6)

Required inputs to AIM2.xls:

| Field | Unit | Notes |
|---|---|---|
| Building interior volume | m³ | Vb, inside dimensions |
| Building height | m | Stack effect input |
| Site wind exposure category | — | Per AIM-2 categories (sheltered/normal/exposed) |
| ACH50 (air changes per hour at 50 Pa) | AC/H | From blower door test; prescriptive fallback in spreadsheet |
| ELA10 (effective leakage area at 10 Pa) | cm² | From blower door test or prescriptive |
| Ventilation system type | — | Balanced / exhaust-only / supply-only |
| Principal continuous ventilation rate | L/s | PVC — for pressurization adjustment in AIM-2 |
| January average wind speed | km/h | From Table 5 / F280_Weather.xls |
| July average wind speed | km/h | From Table 5 / F280_Weather.xls |

Outputs used in main calculation:
- LRairh = heating mode air leakage rate (AC/H)
- LRairc = cooling mode air leakage rate (AC/H)

### 3.11 Mechanical Ventilation (Cl. 5.2.3.2, 6.2.7)

| Field | Unit | Notes |
|---|---|---|
| Principal continuous ventilation rate (PVC) | L/s | Per CAN/CSA-F326 or local code |
| Minimum ventilation capacity (MVC) | L/s | Per CAN/CSA-F326 (PVC = 40–60% of MVC) |
| Ventilation system type | — | Balanced HRV/ERV / exhaust-only / supply-only |
| HRV apparent sensible effectiveness (E) | fraction | At design temperature and PVC rate per CAN/CSA-C439; 0 if no HRV |
| HRV/ERV adjusted total recovery efficiency (ATRE) | fraction | For cooling; 0 if no unit or no ATRE reported |
| Room ventilation flow rates (Qvr) | L/s each | If separate duct system supplies rooms individually |

### 3.12 Ducts (Cl. 5.2.4, 6.2.8)

| Field | Unit | Notes |
|---|---|---|
| Duct location | — | Attic / open crawlspace / enclosed unconditioned crawlspace / slab perimeter |
| RSI value of duct insulation | m²·°C/W | 0 if bare duct |
| DLMh (heating duct loss multiplier) | — | From Table 1 |
| DGMc (cooling duct gain multiplier) | — | From Table 1 |

### 3.13 Pipes — Hydronic Systems Only (Cl. 5.2.5)

| Field | Unit | Notes |
|---|---|---|
| Pipe location | — | Through unconditioned space or slab perimeter |
| Insulated or non-insulated | — | |
| (HLager + HLbgcr + HLairr) / Afloorb | W/m² | For Table 2 lookup of PLMh |

### 3.14 Internal Gains (Cooling Only)

| Field | Unit | Notes |
|---|---|---|
| Number of occupants | count | Bedrooms + 1 if unknown |
| Gross floor area of building (Afloorb) | m² | For 4 W/m² appliance/lighting gain |

### 3.15 Building-Level Data

| Field | Unit | Notes |
|---|---|---|
| Building volume (Vb) | m³ | Inside dimensions |
| Gross floor area (Afloorb) | m² | Inside dimensions |
| Compass orientation (front-facing direction) | N/S/E/W/… | For solar calculations; if unknown, use worst-case for cooling |
| Latitude | degrees | For LFactor calculation |
| Location (city/weather station) | — | Selects Toh, Toc, climate data |
| Number of building levels | count | For level factor table in Cl. 5.2.3.3 |

---

## 4. OPENINGS — RSI_W AND SHGC: FINDING FROM THE STANDARD

### What F280 Requires (Cl. 5.2.1, 6.2.2)

Every window, glazed door, and translucent assembly requires **two values** for the load calculation:
1. **RSI_W (m²·°C/W):** The whole-window thermal resistance. Used in the heating heat-loss formula
   (Cl. 5.2.1: `HLage = A/RSI × DTDh`) and the conductive term of the cooling gain formula
   (Cl. 6.2.2: `DTDc / RSIw` term in HGet).
2. **SHGC (solar heat gain coefficient):** Dimensionless. Used only in the cooling solar gain term
   (Cl. 6.2.2: `SHGC × Solar` term in HGet). Not used in heating.

Both values must be for the **whole window assembly** (glazing + frame + spacer combined), not
centre-of-glass values alone.

### Source Priority Per the Standard (Cl. 50, Table 6 preamble)

F280 establishes a clear priority order:

**1. Preferred — manufacturer's rated performance data (Cl. 50 / Table 6 preamble):**
> "Designers *should* use the actual performance data for the windows being used. This performance
> data can be found in the window manufacturer's ratings, as determined by testing to the
> specifications of CAN/CSA-A440, CAN/CSA-A440.2, or CAN/CGSB 82.1."

The standard's language is "should" (recommended, not mandatory) for this path, but it is explicitly
the preferred source. Test-rated data from the manufacturer is more accurate than the table defaults.

**2. Fallback — Tables 6E–6H default lookup (Cl. 50 / Table 6 preamble):**
> "If the manufacturer of the windows being used is not known, or if performance data for the windows
> is not available, the designer may use physical descriptors of the windows and match them to the
> characteristics listed in Tables 6E to 6H."

The fallback tables are keyed by:
- Number of glazing layers: single (6E), double (6F), triple (6G), triple TG-2 (6H)
- Frame material: Aluminum / Wood or Vinyl
- Operability: Fixed / Operable
- Storm window: Yes / No (single-glazed only, Table 6E)
- Spacer type: Metal / Insulating (double and triple only)
- Glazing coating: Clear / Low-E (double and triple; TG-2 is Low-E only by definition)
- Gap gas and size: 6 mm Air, 6 mm Argon, 9 mm Krypton, 13 mm Air, 13 mm Argon (double and triple)

The recovered table values are in `F280_OCR_RAW.md`, pages 51–54. **The tables return both RSI_W and
SHGC simultaneously from the same row** — they cannot be mixed across rows.

### The Operator's Decision (to be resolved in the main planning chat)

This spec describes what F280 needs. How Collabinator's model *sources* RSI_W and SHGC for each
opening is an implementation decision that belongs in the main planning chat. The decision involves:
- Whether to carry manufacturer-rated RSI_W/SHGC as first-class fields on the opening object
- Whether and how to expose the Tables 6E–6H lookup as a fallback UI (keyed by the 6 descriptor fields
  above)
- How to handle the case where SHGC is not needed for a heating-only project but RSI_W is

**This is the opening U/SHGC finding flagged for main-chat resolution.**

---

## 5. GAP-ANALYSIS HOOK

The following checklist states what F280 requires per surface type. The operator shall diff this list
against the live main Collabinator model in the main planning chat to identify gaps.

```
SURFACE TYPE                    REQUIRED FIELDS (F280)
─────────────────────────────────────────────────────────────────────────
Above-grade opaque wall         □ Net area (m²)
                                □ RSI (m²·°C/W) — framing-adjusted
                                □ Orientation (for cooling SC)

Roof / ceiling                  □ Net area (m²)
                                □ RSI (m²·°C/W) — framing-adjusted
                                □ Slope/orientation (for cooling SC)

Exposed floor                   □ Net area (m²)
                                □ RSI (m²·°C/W) — framing-adjusted

Rim joist                       □ Area (m²)
                                □ RSI (m²·°C/W) — Table 7A.2 adjusted

Window                          □ Area — rough opening (m²)
                                □ RSI_W — whole window (m²·°C/W)
                                □ SHGC
                                □ Orientation
                                □ Interior shading type (SFactor)
                                □ Shaded / unshaded area split

Opaque door                     □ Area — rough opening (m²)
                                □ RSI — Table 6I or manufacturer (m²·°C/W)
                                □ Storm door present (Yes/No)
                                □ Orientation

Glazed door (glazed portion)    □ Glazed area (m²)
                                □ RSI_W (m²·°C/W)
                                □ SHGC
                                □ Orientation

Basement wall / slab            □ Foundation configuration # (1–145)
(BASESIMP)                      □ L × W × H × depth below grade
                                □ Soil conductivity
                                □ Water table level
                                □ RSI interior insulation (vertical)
                                □ RSI exterior insulation (vertical)
                                □ Insulation overlap
                                □ RSI slab insulation
                                □ Number of corners
                                □ Heating degree days
                                □ Deep ground temperature
                                □ Monthly avg dry bulb temps (× 12)
                                □ Room temperature
                                □ Outdoor design dry bulb heating + cooling
                                □ Slab temperature
                                □ Exposed perimeter (attached houses)
                                □ Above-grade window/door areas

Slab-on-grade                   □ Foundation configuration # (1–145)
(BASESIMP)                      □ Slab L × W
                                □ Soil conductivity, water table
                                □ RSI slab insulation
                                □ Number of corners
                                □ Heating degree days, deep ground temp
                                □ Monthly avg dry bulb temps (× 12)
                                □ Room temperature
                                □ Outdoor design dry bulb heating + cooling
                                □ Slab temperature
                                □ Exposed perimeter

Air leakage                     □ Building volume Vb (m³)
(AIM-2)                         □ Building height
                                □ Site wind exposure category
                                □ ACH50 (or prescriptive default)
                                □ ELA10 (or prescriptive default)
                                □ Ventilation system type
                                □ PVC (L/s)
                                □ January + July avg wind speed (km/h)

Ventilation                     □ PVC (L/s)
                                □ HRV effectiveness E (fraction)
                                □ HRV/ERV ATRE (fraction, for cooling)
                                □ Ventilation system type
                                □ Room-level Qvr (L/s) if distributed

Ducts                           □ Duct location
                                □ Duct insulation RSI
                                → DLMh and DGMc from Table 1

Pipes (hydronic)                □ Pipe location
                                □ Insulated or not
                                → PLMh from Table 2

Internal gains (cooling)        □ Number of occupants (or bedrooms + 1)
                                □ Gross floor area Afloorb (m²)

Building-level                  □ Building volume Vb (m³)
                                □ Gross floor area Afloorb (m²)
                                □ Compass orientation / facing direction
                                □ Latitude
                                □ Location (climate station)
                                □ Number of building levels
```

---

*End of F280_COMPLIANCE_SPEC.md. Source: CSA F280:12 (reaffirmed 2025), OCR via F280_OCR_RAW.md.*
*This document describes standard requirements only. Implementation decisions for the Collabinator
model are resolved separately in the main planning chat, not here.*
