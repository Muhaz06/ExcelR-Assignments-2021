# Healthcare Provider Affiliation Mapping Analysis

## Overview
This SQL script performs a comprehensive analysis to map Healthcare Providers (HCPs) to Healthcare Organizations (HCOs) using fuzzy matching techniques and iterative processes. The script processes approximately 49,756 NPIs (National Provider Identifiers) to identify organizational affiliations.

## Key Components

### 1. Data Sources
- **Primary NPI List**: `SAND_SCIPHER_DS_DB.ATS.TENPOINT_HCP_LIST_2025_07_18` (49,756 NPIs)
- **Base Tables**:
  - `PROD_RWD_PRESENTATION_DB.VEEVA_OPEN_DATA.HCP` (Healthcare Providers)
  - `PROD_RWD_PRESENTATION_DB.VEEVA_OPEN_DATA.HCO` (Healthcare Organizations)
  - `PROD_RWD_PRESENTATION_DB.VEEVA_OPEN_DATA.PARENTHCO` (Parent Organizations)
  - `PROD_RWD_PRESENTATION_DB.VEEVA_OPEN_DATA.ADDRESS` (Address Information)
  - `PROD_RWD_PRESENTATION_DB.VEEVA_OPEN_DATA.REFERENCE` (Reference Data)

### 2. Variable Configuration
```sql
SET NPI_TABLE = 'SAND_SCIPHER_DS_DB.ATS.TENPOINT_HCP_LIST_2025_07_18';
SET YOUR_CREDENTIALS='I_RD';
SET DATE_RUN = '20250718'||$YOUR_CREDENTIALS;
```

### 3. Processing Workflow

#### Phase 1: Data Preparation
- **CODE 01**: Creates master table linking HCPs to all their HCO affiliations with address details
- **CODE 02**: Adds fuzzy matching scores using JAROWINKLER_SIMILARITY for:
  - Address lines
  - Suite numbers
  - City, State, ZIP
  - Latitude/Longitude
  - Phone numbers
- **CODE 03**: Defines 17 matching criteria conditions with varying strictness levels

#### Phase 2: Iterative Affiliation Mapping

**Process 1 - Physician Group (PG) Affiliations**
- Filters for HCOs with `HCO_TYPE='Group Practice'`
- Results: 32,490 HCPs mapped to Physician Groups

**Process 3 - Clinic Affiliations**
- Targets remaining HCPs affiliated with clinics
- Filters: `HCO_TYPE_DETAIL in ('Organization, Dept at Hospital', 'Walk-In Clinic')`
- Results: 85 HCPs mapped to Clinics

**Process 5 - Hospital Affiliations**
- Maps remaining HCPs to hospitals
- Filter: `HCO_TYPE ='Hospital'`
- Results: 1,920 HCPs mapped to Hospitals

**Process 7 - Other Affiliations**
- Handles remaining non-PG, non-clinic, non-hospital affiliations
- Results: 1,373 HCPs mapped to "Others"

### 4. Fuzzy Matching Criteria
The system uses 17 hierarchical conditions, from most strict to more lenient:

**Strictest Conditions (1-3)**:
- 100% address, state, ZIP, latitude, longitude matches
- Eye care specialty name patterns required
- Phone number matching

**Moderate Conditions (4-13)**:
- 90-100% address similarity
- Geographic matching requirements
- Specialty-specific filtering

**Most Lenient (14-17)**:
- 80% address similarity
- Basic geographic matching

### 5. Data Quality Filters
Excludes HCOs with names containing:
- Orthopedic, Dental, Oncology, Neurology
- Dermatology, Cancer, Pathology, Radiology
- Psychology, Cardiology, Urology, Psychiatry
- And 20+ other specialty exclusions

### 6. Final Results Summary

| HCO Type Group | HCP Count |
|----------------|-----------|
| Physician Group | 32,490 |
| Solo Practice | 13,888 |
| Hospital | 1,920 |
| Others | 1,373 |
| Clinic | 85 |
| **Total** | **49,756** |

### 7. Key Features

#### Fuzzy Matching Algorithm
- Uses JAROWINKLER_SIMILARITY for string comparison
- Handles address normalization and standardization
- Accounts for geographic proximity via lat/long

#### Address Prioritization
- Primary addresses preferred over calculated addresses
- Current addresses prioritized over historical

#### Conflict Resolution
For 1:Many mappings, prioritizes based on:
1. Phone number exact matches
2. Specialty relevance (Gynecology/Urology > General Practice)
3. Geographic proximity scores

#### Solo Practice Identification
HCPs are classified as "Solo Practice" if:
- No organizational affiliations found through fuzzy matching
- HCP and HCO ZIP codes don't match (indicating poor affiliation quality)

### 8. Output Tables
- **Master Data**: Complete HCP-HCO mapping with all attributes
- **Final Target List**: Clean dataset with opt-out flags from Symphony database
- **Iteration Tables**: Intermediate results for each processing phase

### 9. Data Validation
- Maintains referential integrity between HCPs and addresses
- Validates record states ('VALID', 'UNDER_REVIEW')
- Ensures primary address flags are properly set

## Technical Implementation Notes

### Performance Optimizations
- Uses CTEs for complex joins
- Implements cursor-based iteration for condition processing
- Leverages window functions for ranking and deduplication

### Data Quality Measures
- Excludes invalid/inactive records
- Filters out non-relevant medical specialties
- Implements geographic validation checks

### Scalability Considerations
- Modular design allows for easy condition modifications
- Variable-based table naming supports multiple runs
- Stored procedures enable automated processing

This script represents a sophisticated approach to healthcare provider network analysis, combining fuzzy matching, geographic validation, and business rule implementation to create accurate provider-organization mappings.