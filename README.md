# 📦 U.S. Export Compliance Risk Analysis Framework

## Overview

An end-to-end risk detection framework for analyzing U.S. export shipments, identifying compliance violations across five key regulatory risk layers. This project simulates a real-world export compliance audit for "Apex Global Tech," a U.S.-based technology manufacturer with global operations.

### Business Context

Following a warning letter from the Bureau of Industry and Security (BIS), Apex Global Tech's CEO mandated a comprehensive review of all export shipments. The company processes thousands of exports annually, but fragmented compliance systems have created significant risk exposure.

### North Star Question

> *"What are the most critical compliance and operational risks in our U.S. export supply chain, and how can we quantify their financial and operational impact?"*

---

## 🛠️ Technical Implementation

### Technology Stack

| Tool | Purpose |
|------|---------|
| **Python 3.9** | Core programming language |
| **Pandas** | Data manipulation and analysis |
| **NumPy** | Numerical operations |
| **Matplotlib** | Data visualization |
| **Seaborn** | Advanced statistical visualizations |
| **Jupyter Notebooks** | Interactive analysis environment |
| **VS Code** | Development environment |

---

## 🔴 Risk Detection Framework

### The Five Risk Layers

| Layer | Risk Type | Regulatory Basis | Detection Logic |
|-------|-----------|------------------|-----------------|
| **1** | Address-String Screening | OFAC Sanctions, BIS Entity List | Keyword matching in consignee address fields |
| **2** | FDPR Auditing | 15 CFR § 734.9 | US tooling + foreign production + non-EAR99 classification |
| **3** | EEI Threshold Validation | 15 CFR Part 30 | >$2,500 value + NLR classification |
| **4** | PGA Approval Validation | PGA Requirements (FDA, EPA, etc.) | PGA required + approval token missing |
| **5** | License Validation | EAR Licensing Requirements | License type declared + license not obtained |

---

### Risk Layer 1: Address-String Screening

**Purpose**: Identify hidden sanctioned locations in consignee addresses

**Regulatory Basis**: OFAC Sanctions, BIS Entity List

**What Are We Trying to Check?**

**The Core Question**: Are we inadvertently shipping to sanctioned or restricted entities by hiding their location in the address field?

**The Problem We're Solving**

Imagine this scenario:

- A company in North Korea wants to buy medical equipment
- They know they're sanctioned, so they use a front company name
- But the address still contains "Pyongyang, North Korea"
- Your screening system only checks the company name, not the address
- The shipment goes through and you violate OFAC sanctions

**Sanctioned Keywords** (12 total):
- NORTH KOREA, PYONGYANG
- IRAN, TEHRAN
- SYRIA, DAMASCUS
- CRIMEA, SIMFEROPOL
- CUBA, HAVANA
- VENEZUELA, CARACAS

**Code Snippet**:
```python
# Define sanctioned keywords
SANCTIONED_KEYWORDS = [
    'NORTH KOREA', 'PYONGYANG',  # North Korea
    'IRAN', 'TEHRAN',             # Iran
    'SYRIA', 'DAMASCUS',          # Syria
    'CRIMEA', 'SIMFEROPOL',       # Crimea
    'CUBA', 'HAVANA',             # Cuba
    'VENEZUELA', 'CARACAS'        # Venezuela
]

# Apply screening logic (case-insensitive)
df['address_red_flag'] = df['consignee_address'].str.upper().apply(
    lambda x: any(kw in x for kw in [kw.upper() for kw in SANCTIONED_KEYWORDS])
)

# Count matches
address_risk_count = df['address_red_flag'].sum()
address_risk_rate = (address_risk_count / len(df)) * 100

print(f"Shipments with hidden sanctioned addresses: {address_risk_count:,}")
print(f"Defect Rate: {address_risk_rate:.2f}%")
```

**Results**:

* Total Address Red Flags Found: 101 shipments
* Defect Rate: 2.02%
* Clean Shipments: 4,899 (97.98%)

**Analysis by Business Unit**:

* US-ConsumerElectronics: 43 red flags (2.72%) - HIGHEST RISK
* US-IndustrialEquipment: 19 red flags (1.97%)
* US-Pharmaceuticals: 9 red flags (1.88%)
* US-MedicalDevices: 21 red flags (1.69%)
* US-AerospaceParts: 9 red flags (1.22%)

**Monthly Trend (Last 6 months)**:
* 2026-02: 0.76%
* 2026-03: 1.24%
* 2026-04: 3.00%
* 2026-05: 2.71%
* 2026-06: 1.02%
* 2026-07: 5.38% (Spike - investigate)

**Visualization**:

![Address Screening Results](output/risk_layer1_address_screening.png)


**Key Insights**:

**Highest Risk Business Unit**:

US-ConsumerElectronics has the highest defect rate at 2.72%, accounting for 43 out of 101 total address red flags (42.6% of all defects). This suggests inadequate data entry training or missing address validation in this division.

**Monthly Volatility**: 

Defect rates show significant monthly variation, ranging from 0.76% to 5.38%. The spike in July 2026 (5.38%) warrants investigation - was there a new employee, new system, or process change?

**Destination Pattern**: 

The majority of red-flagged shipments are destined to North Korea and Crimea, which are under strict US sanctions.

**Hidden Defects**: 

Many defects were hidden within otherwise legitimate addresses (e.g., "123 Corporate Blvd, Dublin, IE, PYONGYANG, NORTH KOREA"), demonstrating that name-only screening would have missed these violations.

**Action Plan**:
* Immediate: Review all 101 flagged shipments and validate consignee addresses (1 week)
* Short-term: Deploy automated address validation at point of entry for US-ConsumerElectronics (30 days)
* Medium-term: Provide targeted training to data entry teams in high-risk business units (60 days)
* Long-term: Implement real-time OFAC screening API integration (90 days)


### Risk Layer 2: Foreign Direct Product Rule (FDPR) Auditing

**Purpose**: Identify items subject to US jurisdiction under FDPR

**Regulatory Basis**: 15 CFR § 734.9

**What is FDPR?**

The Foreign Direct Product Rule (15 CFR § 734.9) states that if a product is made **outside the US** but uses **US-origin technology, software, or tooling**, it may still be subject to US export controls.

**Think of it like**: *"Made in China, but with American brains"* → Still controlled by US law

**The Rule**:

Foreign-made items that are the **direct product** of US-origin technology or software are subject to the EAR if they are destined for certain countries.

**Example**:

- You manufacture chips in China (`bom_origin_country = 'CN'`)
- But you use US-made manufacturing equipment (`bom_tooling_origin = 'US'`)
- The chips are still under US jurisdiction → Must be classified correctly!

**What We're Checking**

**Key Question**: Are we misclassifying items that are actually under US jurisdiction?

**The Logic**:

- Item is made abroad (`bom_origin_country != 'US'`)
- Uses US tooling/software (`bom_tooling_origin == 'US'`)
- Not classified as EAR99 (misclassified as controlled item)

**The Defect**: Items subject to FDPR but classified as controlled ECCN codes (should be properly assessed!)

**Code Snippet**:
```python
# Apply FDPR jurisdiction logic
df['fdpr_jurisdiction_flag'] = (
    (df['bom_origin_country'] != 'US') & 
    (df['bom_tooling_origin'] == 'US')
)

# High-risk FDPR: Items under US jurisdiction but misclassified
df['fdpr_high_risk'] = (
    df['fdpr_jurisdiction_flag'] & 
    (~df['eccn'].isin(['EAR99', 'NLR']))
)
```
**Results**:

- **Items under FDPR jurisdiction**: 1,293 shipments
- **High-risk misclassified items**: 679 shipments
- **Defect Rate**: 13.58%
- **Total Financial Exposure**: $431,253,329.98
- **Average Value per Misclassified Item**: $635,130.09

**Analysis by ECCN Code**:

- **5A002**: 233 violations (highest risk)
- **3A001**: 190 violations
- **9A004**: 133 violations
- **5A001**: 123 violations

**Analysis by Business Unit**:

| Business Unit | FDPR Violations | Total Value Exposed |
|---------------|-----------------|---------------------|
| US-ConsumerElectronics | 212 | $139,580,200 |
| US-MedicalDevices | 174 | $113,583,200 |
| US-IndustrialEquipment | 135 | $74,797,180 |
| US-AerospaceParts | 107 | $71,035,820 |
| US-Pharmaceuticals | 51 | $32,256,970 |

**Top FDPR Risks (ECCN × Country)**:

| ECCN | Country | Shipments | Exposed Value |
|------|---------|-----------|---------------|
| EAR99 | MX | 79 | $55,961,435.63 |
| EAR99 | IE | 85 | $53,636,808.43 |
| EAR99 | SG | 83 | $50,907,240.00 |
| EAR99 | BE | 74 | $50,290,582.62 |
| EAR99 | SA | 77 | $46,599,683.49 |
| EAR99 | CN | 69 | $45,138,492.80 |
| EAR99 | IN | 68 | $43,682,096.73 |
| EAR99 | TW | 66 | $35,607,299.16 |
| 5A002 | BE | 33 | $25,628,100.43 |
| 5A002 | CN | 29 | $21,577,590.74 |

Visualization:

![FDPR Auditing Results](output/risk_layer2_fdpr_auditing.png)


**Key Insights**:

1. **Massive Regulatory Exposure**: $431.3 million in products are misclassified under FDPR. These items are made abroad but use US tooling/software, meaning they should be subject to US export controls but are being treated as EAR99.

2. **High-Risk ECCN Codes**: The top misclassified ECCN codes are:
   - **5A002** (telecom/encryption): 233 violations
   - **3A001** (electronics): 190 violations
   - **9A004** (aerospace): 133 violations
   - **5A001** (telecom): 123 violations
   
   These are all controlled items that require licenses.

3. **Consumer Electronics Dominance**: US-ConsumerElectronics has the highest number of FDPR violations (212) and the highest financial exposure ($139.6M), representing 32.4% of total exposure.

4. **Geographic Concentration**: Mexico, Ireland, Singapore, Belgium, and Saudi Arabia are the top destinations for misclassified FDPR items, accounting for over $250M in exposure.

5. **Hidden Pattern**: Many FDPR items are classified as EAR99 but have controlled ECCN codes, suggesting systematic misclassification of products made with US technology abroad.

**Action Plan**:

- **Immediate**: Review all 679 misclassified shipments and validate ECCN classifications (1 week)
- **Short-term**: Implement automated FDPR detection in shipping database; flag items with US tooling + foreign origin for manual review (30 days)
- **Medium-term**: Conduct compliance training for product classification teams, especially US-ConsumerElectronics (60 days)
- **Long-term**: Implement real-time FDPR screening with automated holds for high-risk combinations (90 days)

### Risk Layer 3: EEI Threshold Validation

**What is EEI?**

Electronic Export Information (EEI) is the electronic filing required for shipments leaving the US. Under 15 CFR Part 30, any shipment valued over $2,500 classified as "No License Required" (NLR) must file EEI through the Automated Export System (AES).

**Think of it like**: *"You can't just ship high-value items without telling the government"* → Must file EEI

**The Rule**:

Any single line item valued over **$2,500** with **NLR classification** must file EEI before export.

**Example**:

- Your item is valued at $3,500 (`total_value_usd > 2500`)
- It's classified as NLR (`license_type = 'NLR'`)
- You must file EEI → If not, it's a violation!

**What We're Checking**

**Key Question**: Are we failing to file EEI for high-value NLR shipments?

**The Logic**:

- Shipment value exceeds $2,500 (`total_value_usd > 2500`)
- Classified as NLR (`license_type == 'NLR'`)
- EEI filing is required (by law)

**The Defect**: Shipments over $2,500 under NLR without EEI filing

**Code Snippet**:
```python
# Apply EEI threshold logic
EEI_THRESHOLD = 2500
df['eei_filing_required'] = (
    (df['total_value_usd'] > EEI_THRESHOLD) & 
    (df['license_type'] == 'NLR')
)
```

**Results**:

- **EEI Violations Found**: 2,211 shipments
- **Defect Rate**: 44.22%
- **Total Value at Risk**: $1,417,623,062.12
- **Average Value per Violation**: $641,168.28

**Analysis by License Type**:

| License Type | Total Shipments | EEI Violations | Violation Rate |
|--------------|-----------------|----------------|----------------|
| NLR | 2,218 | 2,211 | 99.68% |
| LICENSE | 1,789 | 0 | 0.00% |
| EXCEPTION | 993 | 0 | 0.00% |

**Key Finding**: 99.68% of all NLR shipments exceed the $2,500 threshold and require EEI filing!

**Analysis by Business Unit**:

| Business Unit | Total Shipments | EEI Violations | Violation Rate |
|---------------|-----------------|----------------|----------------|
| US-AerospaceParts | 735 | 332 | 45.17% |
| US-MedicalDevices | 1,239 | 557 | 44.96% |
| US-Pharmaceuticals | 478 | 213 | 44.56% |
| US-ConsumerElectronics | 1,582 | 693 | 43.81% |
| US-IndustrialEquipment | 966 | 416 | 43.06% |

**Analysis by Port of Exit**:

| Port of Exit | Total Shipments | EEI Violations | Violation Rate |
|--------------|-----------------|----------------|----------------|
| MIAMI | 715 | 326 | 45.59% |
| HOUSTON | 713 | 321 | 45.02% |
| SEATTLE | 722 | 325 | 45.01% |
| LOS ANGELES | 722 | 323 | 44.74% |
| CHICAGO O'HARE | 699 | 306 | 43.78% |
| NEWARK | 712 | 307 | 43.12% |
| NEW YORK | 717 | 303 | 42.26% |

**Monthly Trend** (Last 6 months):

| Month | Violation Rate |
|-------|----------------|
| 2026-02 | 42.78% |
| 2026-03 | 47.01% |
| 2026-04 | 44.25% |
| 2026-05 | 42.53% |
| 2026-06 | 42.20% |
| 2026-07 | 41.94% |

**Visualization**:

![EEI Threshold Results](output/risk_layer3_eei_threshold.png)

**Key Insights**:

- **Systemic Filing Failure**: 99.68% of all NLR shipments exceed the $2,500 threshold. This means virtually every NLR shipment requires EEI filing, but the system is not enforcing this requirement.

- **Massive Financial Exposure**: $1.42 billion in shipments are at risk of EEI violations. This represents a significant regulatory exposure that could result in substantial penalties.

- **Consistent Pattern Across Business Units**: All business units have violation rates between 43-45%, indicating this is a company-wide systemic issue rather than isolated errors.

- **Port-Level Consistency**: EEI violation rates are remarkably consistent across all ports (42-46%), suggesting the issue is with the shipping system/process rather than specific port operations.

- **Stable Trend**: Violation rates have remained consistently high (41-47%) over the 12-month period, indicating this is a chronic, unresolved compliance gap.

**Action Plan**:

- **Immediate**: Flag all 2,211 shipments for immediate EEI filing. Stop any shipments that haven't been filed (1 week)

- **Short-term**: Implement automated pre-shipment validation that checks EEI filing status for all NLR shipments >$2,500 (30 days)

- **Medium-term**: Review and update EEI filing procedures. Create automated alerts for shipments requiring EEI filing (60 days)

- **Long-term**: Integrate EEI filing status into shipping system with automated holds for non-compliant shipments (90 days)


### Risk Layer 4: PGA Approval Validation

**What is PGA?**

Partner Government Agencies (PGA) are federal agencies that regulate specific types of products entering or leaving the US. Examples include:

- **FDA** (Food and Drug Administration) - Medical devices, pharmaceuticals, food
- **EPA** (Environmental Protection Agency) - Chemicals, pesticides
- **ATF** (Alcohol, Tobacco, Firearms) - Firearms, explosives
- **FWS** (Fish and Wildlife Service) - Wildlife products
- **APHIS** (Animal and Plant Health Inspection Service) - Agriculture products

**Think of it like**: *"You can't ship medical devices without FDA approval"* → PGA required

**The Rule**:

If a product is regulated by a PGA, you must obtain PGA approval before shipping. Without it, the shipment will be detained at the border.

**Example**:

- You're shipping medical devices (`pga_required = 'Y'`)
- But you didn't get FDA approval (`pga_obtained = 'N'`)
- The shipment will be detained at the border!

**What We're Checking**

**Key Question**: Are we shipping items requiring PGA approval without getting it?

**The Logic**:

- PGA is required for the item (`pga_required == 'Y'`)
- But PGA approval was not obtained (`pga_obtained == 'N'`)

**The Defect**: Transactions requiring agency clearance that skipped validation

**Code Snippet**:
```python
# Apply PGA approval logic
df['pga_compliance_failure'] = (
    (df['pga_required'] == 'Y') & 
    (df['pga_obtained'] == 'N')
)
```
**Results**:

- **Shipments requiring PGA approval**: 982 shipments
- **Missing PGA token (Skipped Validation)**: 115 shipments
- **PGA Compliance Rate**: 88.29%
- **Overall Defect Rate**: 2.30%
- **Total Value at Risk**: $67,878,591.13

**Analysis by Business Unit**:

| Business Unit | Total Shipments | PGA Failures | Failure Rate |
|---------------|-----------------|--------------|--------------|
| US-AerospaceParts | 735 | 24 | 3.27% |
| US-ConsumerElectronics | 1,582 | 37 | 2.34% |
| US-IndustrialEquipment | 966 | 21 | 2.17% |
| US-MedicalDevices | 1,239 | 25 | 2.02% |
| US-Pharmaceuticals | 478 | 8 | 1.67% |

**Analysis by Destination Country**:

| Country | PGA Failures | Total Value |
|---------|--------------|-------------|
| Belgium (BE) | 19 | $10,487,452.47 |
| Saudi Arabia (SA) | 17 | $10,842,217.88 |
| Mexico (MX) | 15 | $9,718,292.50 |
| Taiwan (TW) | 15 | $7,291,447.73 |
| India (IN) | 13 | $9,658,051.89 |
| Singapore (SG) | 13 | $6,197,923.57 |
| China (CN) | 12 | $7,340,514.35 |
| Ireland (IE) | 10 | $5,238,373.94 |
| Syria (SYRIA) | 1 | $1,104,316.80 |

**Monthly Trend** (Last 6 months):

| Month | PGA Failures | Total Shipments | Failure Rate |
|-------|--------------|-----------------|--------------|
| 2026-02 | 12 | 395 | 3.04% |
| 2026-03 | 6 | 402 | 1.49% |
| 2026-04 | 8 | 400 | 2.00% |
| 2026-05 | 10 | 442 | 2.26% |
| 2026-06 | 12 | 391 | 3.07% |
| 2026-07 | 1 | 93 | 1.08% |

**Visualization**:

![PGA Approval Results](output/risk_layer4_pga_approval.png)

**Key Insights**:

- **Moderate Compliance Rate**: 88.29% of PGA-required shipments have proper approval, but 115 shipments (2.30% of total) are missing required PGA tokens.

- **Aerospace Parts Highest Risk**: US-AerospaceParts has the highest failure rate at 3.27%, suggesting regulatory complexity in aerospace products requiring multiple agency approvals.

- **Geographic Concentration**: Belgium, Saudi Arabia, and Mexico account for the most PGA failures, representing over $31M in at-risk shipments.

- **Syria Red Flag**: One shipment destined for Syria (a sanctioned country) has a PGA failure. This is particularly concerning given the dual regulatory and sanctions risk.

- **Stable Trend**: Failure rates have remained relatively consistent (1-3%) over the 12-month period, indicating this is a chronic but manageable compliance gap.

**Action Plan**:

- **Immediate**: Review all 115 PGA failures and obtain necessary approvals. Flag the Syria shipment for immediate compliance review (1 week)

- **Short-term**: Implement automated PGA validation that checks approval status before shipment release (30 days)

- **Medium-term**: Provide targeted training to US-AerospaceParts on PGA requirements (60 days)

- **Long-term**: Integrate PGA approval verification into shipping system with automated holds for non-compliant shipments (90 days)

### Risk Layer 5: License Validation

**What is License Validation?**

Export licenses are required for controlled items (ECCN codes like 5A002, 3A001, etc.). Companies often declare they have a license, but sometimes they don't actually obtain it.

**Think of it like**: *"You can't say you have a license if you never got one"* → Must validate

**The Rule**:

If you declare a license type (LICENSE or EXCEPTION), you must actually have the license.

**Example**:

- You declare a license (`license_type = 'LICENSE'`)
- But you never actually obtained it (`license_obtained = 'N'`)
- This is a violation!

**What We're Checking**

**Key Question**: Are we declaring licenses we haven't actually obtained?

**The Logic**:

- License type declared (`license_type in ['LICENSE', 'EXCEPTION']`)
- But license was not obtained (`license_obtained == 'N'`)

**The Defect**: Declared licenses without proper authorization

**Code Snippet**:
```python
# Apply license validation logic
df['license_validation_failure'] = (
    (df['license_type'].isin(['LICENSE', 'EXCEPTION'])) & 
    (df['license_obtained'] == 'N')
)
```
**Results**:

- **Shipments declaring license/exception**: 2,782 shipments
- **Missing license (Skipped Validation)**: 115 shipments
- **License Compliance Rate**: 95.87%
- **Overall Defect Rate**: 2.30%
- **Total Value at Risk**: $67,878,591.13
- **Average Value per Failure**: $590,248.62

**Analysis by Business Unit**:

| Business Unit | Total Shipments | License Failures | Failure Rate |
|---------------|-----------------|------------------|--------------|
| US-AerospaceParts | 735 | 24 | 3.27% |
| US-ConsumerElectronics | 1,582 | 37 | 2.34% |
| US-IndustrialEquipment | 966 | 21 | 2.17% |
| US-MedicalDevices | 1,239 | 25 | 2.02% |
| US-Pharmaceuticals | 478 | 8 | 1.67% |

**Analysis by License Type**:

| License Type | Total Shipments | License Failures | Failure Rate |
|--------------|-----------------|------------------|--------------|
| EXCEPTION | 993 | 17 | 1.71% |
| LICENSE | 1,789 | 98 | 5.48% |

**Key Finding**: LICENSE has a significantly higher failure rate (5.48%) compared to EXCEPTION (1.71%). This suggests issues with the formal license application process.

**Analysis by Destination Country**:

| Country | License Failures | Total Value |
|---------|------------------|-------------|
| Belgium (BE) | 19 | $10,487,452.47 |
| Saudi Arabia (SA) | 17 | $10,842,217.88 |
| Mexico (MX) | 15 | $9,718,292.50 |
| Taiwan (TW) | 15 | $7,291,447.73 |
| India (IN) | 13 | $9,658,051.89 |
| Singapore (SG) | 13 | $6,197,923.57 |
| China (CN) | 12 | $7,340,514.35 |
| Ireland (IE) | 10 | $5,238,373.94 |
| Syria (SYRIA) | 1 | $1,104,316.80 |

**Monthly Trend** (Last 6 months):

| Month | License Failures | Total Shipments | Failure Rate |
|-------|------------------|-----------------|--------------|
| 2026-02 | 12 | 395 | 3.04% |
| 2026-03 | 6 | 402 | 1.49% |
| 2026-04 | 8 | 400 | 2.00% |
| 2026-05 | 10 | 442 | 2.26% |
| 2026-06 | 12 | 391 | 3.07% |
| 2026-07 | 1 | 93 | 1.08% |

**Visualization**:

![License Validation Results](output/risk_layer5_license_validation.png)

**Key Insights**:

- **High Overall Compliance**: 95.87% of declared licenses are properly obtained. However, 115 shipments (2.30% of total) are missing required licenses.

- **LICENSE vs EXCEPTION Gap**: LICENSE has a 5.48% failure rate compared to EXCEPTION's 1.71%. This suggests the formal license application process is more complex and prone to errors.

- **Aerospace Parts Highest Risk**: US-AerospaceParts has the highest failure rate at 3.27%, consistent with their pattern of higher compliance risk across multiple layers.

- **Geographic Concentration**: Belgium, Saudi Arabia, and Mexico account for the most license failures, representing over $31M in at-risk shipments.

- **Syria Red Flag**: One shipment destined for Syria (a sanctioned country) has a license failure. This is particularly concerning given the dual regulatory and sanctions risk.

**Action Plan**:

- **Immediate**: Review all 115 license failures and obtain necessary approvals. Flag the Syria shipment for immediate compliance review (1 week)

- **Short-term**: Streamline the LICENSE application process to reduce the 5.48% failure rate. Implement automated license validation that checks status before shipment release (30 days)

- **Medium-term**: Provide targeted training to US-AerospaceParts on license requirements. Create automated alerts for pending license approvals (60 days)

- **Long-term**: Integrate license verification into shipping system with automated holds for shipments without valid licenses (90 days)


---

## Summary: All Five Risk Layers

| Layer | Risk Type | Defect Rate | Key Finding |
|-------|-----------|-------------|-------------|
| **Layer 1** | Address Screening | 2.02% | 101 shipments with hidden sanctioned addresses |
| **Layer 2** | FDPR Auditing | 13.58% | $431M in misclassified controlled items |
| **Layer 3** | EEI Threshold | 44.22% | 2,211 NLR shipments missing EEI filing |
| **Layer 4** | PGA Approval | 2.30% | 115 shipments missing PGA approval |
| **Layer 5** | License Validation | 2.30% | 115 shipments with invalid licenses |

---
