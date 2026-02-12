# Product Requirements Document: academicr Package

## Overview
An R package for parsing, manipulating, and formatting academic period data following lubridate design principles. Addresses the challenge of working with academic calendars that span calendar years, vary across institutions, and exist in multiple format conventions.

## Problem Statement
Academic institutions use inconsistent period notation formats (fa26, 20268, Fall 2026, etc.) and have diverse calendar systems (semesters, quarters, trimesters, custom terms) making data analysis difficult. Academic years cross calendar years (Fall 2026 → Spring 2027 → Summer 2027 = AY 2026-27), breaking natural sort order and complicating time-based analyses.

## Target Users
- Higher education data analysts
- Institutional researchers
- Academic schedulers
- R users working with academic calendar data across diverse institutional contexts

## Core Functionality

### 1. Academic Period Object (S3 Class)
**Internal representation:**
```r
structure(
  list(
    name = "Fall",        # Period name from configured calendar
    code = "fa",          # Short code from configured calendar
    year = 2026,          # numeric year
    start_date = as.Date("2026-08-23"),  # Actual start date
    ay_start = 2026,      # Academic year start year
    ay_end = 2027,        # Academic year end year
    calendar_id = "default"  # Reference to calendar configuration
  ),
  class = "academic_period"
)
```

**Display format:**
```r
print(period("fa26"))
#> <academic_period[1]> Fall 2026 (AY 2026-27)
#> Starts: 2026-08-23
```

### 2. Calendar Configuration

#### `set_academic_calendar(periods, ay_start_period, calendar_id = "default", yyyym_strict = TRUE)`
**Define institutional calendar:**
```r
set_academic_calendar(
  periods = list(
    list(name = "Fall", code = "fa", start_date = "08-23"),
    list(name = "Spring", code = "sp", start_date = "01-15"),
    list(name = "Summer", code = "su", start_date = "06-01")
  ),
  ay_start_period = "Fall",
  calendar_id = "default",
  yyyym_strict = TRUE
)
```

**Parameters:**
- `periods`: List of period definitions, each with:
  - `name`: Full period name (e.g., "Fall", "Winter", "Michaelmas")
  - `code`: 2-letter short code (e.g., "fa", "wi", "mi")
  - `start_date`: MM-DD format (e.g., "08-23", "01-15")
- `ay_start_period`: Which period starts the academic year (by name)
- `calendar_id`: Identifier for this calendar (supports multiple calendars)
- `yyyym_strict`: If TRUE, error on ambiguous YYYYM formats; if FALSE, warn and use first match

**Examples for different systems:**
```r
# Quarters
set_academic_calendar(
  periods = list(
    list(name = "Fall", code = "fa", start_date = "09-20"),
    list(name = "Winter", code = "wi", start_date = "01-05"),
    list(name = "Spring", code = "sp", start_date = "03-28"),
    list(name = "Summer", code = "su", start_date = "06-15")
  ),
  ay_start_period = "Fall"
)

# UK Terms
set_academic_calendar(
  periods = list(
    list(name = "Michaelmas", code = "mi", start_date = "10-10"),
    list(name = "Lent", code = "le", start_date = "01-15"),
    list(name = "Easter", code = "ea", start_date = "04-25")
  ),
  ay_start_period = "Michaelmas"
)

# Custom with mid-month starts
set_academic_calendar(
  periods = list(
    list(name = "Fall", code = "fa", start_date = "08-23"),
    list(name = "J-Term", code = "jt", start_date = "01-03"),
    list(name = "Spring", code = "sp", start_date = "01-25"),
    list(name = "Summer", code = "su", start_date = "05-15")
  ),
  ay_start_period = "Fall",
  yyyym_strict = FALSE  # January has two periods
)
```

#### `get_academic_calendar(calendar_id = "default")`
Retrieve current calendar configuration

#### `validate_academic_calendar(calendar_id = "default")`
Validate calendar configuration and warn about potential issues:
- Check for duplicate codes
- Identify ambiguous YYYYM mappings (multiple periods in same month)
- Verify ay_start_period exists
- Validate date formats

**Output:**
```r
validate_academic_calendar()
#> ✓ Calendar 'default' is valid
#> ⚠ Warning: Months 1, 8 have multiple periods. YYYYM parsing may be ambiguous.
#>   Consider setting yyyym_strict = FALSE or use set_yyyym_mapping().
#> 
#> Period start months:
#>   January: J-Term, Spring
#>   May: Summer
#>   August: Fall
```

#### `set_yyyym_mapping(..., calendar_id = "default")`
**Explicit month-to-period mapping for ambiguous cases:**
```r
set_yyyym_mapping(
  "1" = "Spring",  # Map month 1 to Spring (not J-Term)
  "8" = "Fall",
  calendar_id = "default"
)
```

#### `list_academic_calendars()`
List all configured calendars (supports multi-institution datasets)

### 3. Parsing Functions

#### `period(x, format = "auto", calendar_id = "default")`
**Auto-detect input format and parse:**
- `"fa26"`, `"sp27"`, `"su25"` → code + 2-digit year
- `"20268"`, `"202611"`, `"20272"` → YYYYM format (variable length: 5-6 digits)
- `"Fall 2026"`, `"Spring 2027"` → name + 4-digit year
- `"2026 Fall"`, `"2027 Spring"` → 4-digit year + name
- Vectorized (handles character vectors)

**YYYYM Format Details:**
- 5 digits: YYYY + M (months 1-9) → `20268` = August 2026
- 6 digits: YYYY + MM (months 10-12) → `202611` = November 2026
- Maps to period that starts in that month
- Uses configured calendar to determine which period
- Respects `yyyym_strict` setting

**Auto-detection logic:**
1. Check length and pattern:
   - If 2 letters + 2 digits (`^[a-z]{2}\d{2}$`) → `period_code()`
   - If 5-6 digits (`^\d{5,6}$`) → `period_numeric()`
   - If contains period name from calendar → `period_text()`
2. Otherwise error with helpful message showing expected formats

#### `period_code(x, calendar_id = "default")`
Parse period codes: `"fa26"`, `"sp27"`, `"su25"`, `"wi26"`
- Case insensitive
- Validates code exists in configured calendar
- Assumes 20xx for years (2000-2099)

#### `period_numeric(x, calendar_id = "default")`
Parse YYYYM format: `20268`, `202611`, `20272`
- Handles 5 digits (months 1-9) or 6 digits (months 10-12)
- Extracts year from first 4 digits
- Extracts month from remaining 1-2 digits
- Maps month to period using calendar configuration
- Handles ambiguity based on `yyyym_strict` setting:
  - `TRUE`: Error if multiple periods in same month
  - `FALSE`: Warning + uses first configured period or explicit mapping

**Examples:**
```r
# Clear mapping
period_numeric("20268")   # August -> Fall 2026

# Ambiguous month (January has J-Term and Spring)
period_numeric("20271")   # Error if yyyym_strict=TRUE
                          # Warning + uses mapping if yyyym_strict=FALSE
```

#### `period_text(x, calendar_id = "default")`
Parse text formats: `"Fall 2026"`, `"2026 Fall"`, `"Michaelmas 2026"`
- Case insensitive period names
- Flexible separator (space, comma, hyphen, underscore)
- Support both "NAME YEAR" and "YEAR NAME" order
- Validates period name exists in calendar

### 4. Accessor Functions

#### `period_name(x)`
Returns: Full period name (e.g., `"Fall"`, `"Michaelmas"`)

#### `period_code(x)`
Returns: Short code (e.g., `"fa"`, `"mi"`)

#### `period_year(x)`
Returns: Numeric year (2026, 2027)

#### `period_start_date(x)`
Returns: Date object for period start (e.g., `as.Date("2026-08-23")`)

#### `period_ay(x, format = "short")`
Returns academic year string:
- `format = "short"`: `"2026-27"` (default)
- `format = "long"`: `"2026-2027"`
- `format = "start"`: `2026` (start year only)
- `format = "end"`: `2027` (end year only)

#### `period_term(x)`
Returns: Term number within academic year (1, 2, 3, ...) based on calendar order

#### `period_month(x)`
Returns: Numeric month when period starts (1-12)

#### `period_calendar(x)`
Returns: Calendar ID this period belongs to

### 5. Formatting Functions

#### `format.academic_period(x, format = "key", ...)`
Output formats:
- `"key"`: `"2026-27_20268_Fall"` (sortable composite key)
- `"code"`: `"fa26"` (compact code)
- `"numeric"`: `"20268"` (YYYYM format, variable length)
- `"text"`: `"Fall 2026"` (human readable)
- `"ay_term"`: `"2026-27_1"` (academic year + term number)
- `"iso_date"`: `"2026-08-23"` (ISO 8601 start date)
- `"year_month"`: `"2026-08"` (start year-month)
- `"custom"`: user-defined template using placeholders

**Custom format templates:**
```r
# Available placeholders:
# {ay} or {ay_short}    -> "2026-27"
# {ay_long}             -> "2026-2027"
# {ay_start}            -> "2026"
# {ay_end}              -> "2027"
# {name}                -> "Fall"
# {code}                -> "fa"
# {year}                -> "2026"
# {term}                -> "1"
# {month}               -> "8"
# {month_pad}           -> "08"
# {date}                -> "2026-08-23"
# {year_month}          -> "2026-08"

# Examples:
format(period("fa26"), format = "{ay}_{term}_{month_pad}_{name}")
#> "2026-27_1_08_Fall"

format(period("fa26"), format = "{code}{year}_{date}")
#> "fa26_2026-08-23"
```

#### `as.character.academic_period(x, format = "text")`
Default character coercion (calls `format()`)

### 6. Arithmetic Operations

#### `+.academic_period(x, n)`
Add periods (wraps through academic year and calendar):
```r
period("fa26") + 1  #> Spring 2027 (or next configured period)
period("su27") + 1  #> Fall 2027
period("fa26") + 10 #> Handles multi-year sequences
```

#### `-.academic_period(x, n)`
Subtract periods or compute difference:
```r
period("sp27") - 1                    #> Fall 2026
period("sp27") - period("fa26")       #> 1 (periods apart)
period("fa27") - period("fa26")       #> 3 (if 3 periods/year)
```

#### `seq.academic_period(from, to, by = 1)`
Generate period sequences:
```r
seq(period("fa26"), period("fa27"), by = 1)
#> <academic_period[4]>
#> [1] Fall 2026   Spring 2027 Summer 2027 Fall 2027

# Works across different calendars if periods configured
seq(period("mi26"), period("ea27"), by = 1)
#> <academic_period[4]>
#> [1] Michaelmas 2026  Lent 2027  Easter 2027  Michaelmas 2027
```

### 7. Comparison and Sorting

Implement S3 comparison methods:
- `==.academic_period`, `!=.academic_period`
- `<.academic_period`, `>.academic_period`, `<=.academic_period`, `>=.academic_period`
- `sort.academic_period`, `order.academic_period`

**Comparison logic:** 
- Primary: Compare actual start dates (`period_start_date()`)
- This ensures chronological sorting regardless of calendar system
- Periods from different calendars can be compared if needed

**Examples:**
```r
period("fa26") < period("sp27")   # TRUE (Aug 2026 < Jan 2027)
period("sp27") > period("fa26")   # TRUE

# Sorting mixed periods
sort(c(period("su27"), period("fa26"), period("sp27")))
#> [1] Fall 2026   Spring 2027 Summer 2027

# Even works across calendars (if both configured)
period("mi26", calendar = "uk") < period("fa26", calendar = "us")
```

### 8. Coercion Functions

#### `as.Date.academic_period(x, day = "start")`
Convert to Date object:
- `day = "start"`: Start date from calendar configuration (default)
- `day = "mid"`: Computed midpoint (requires end date logic)
- `day = "end"`: End date (requires end date specification in calendar)

**Note:** Midpoint and end date calculations require additional calendar configuration specifying period durations or end dates.

#### `as_period(x, calendar_id = "default")`
Generic coercion function (calls appropriate parser based on input type)

#### `as.POSIXct.academic_period(x, ...)`
Coerce to POSIXct (via `as.Date()` with time set to 00:00:00)

### 9. Validation

#### `is_period(x)`
Check if object is of class `"academic_period"`

#### `validate_period(x)`
Internal function to check period object integrity:
- Verify all required fields present
- Check dates are valid
- Confirm period exists in referenced calendar
- Validate ay_start <= ay_end

### 10. Utility Functions

#### `current_period(calendar_id = "default")`
Get current academic period based on system date and configured calendar

#### `periods_in_year(year = NULL, calendar_id = "default")`
List all periods in a given academic year:
```r
periods_in_year(2026)
#> <academic_period[3]>
#> [1] Fall 2026   Spring 2027 Summer 2027
```

#### `academic_year_range(ay, calendar_id = "default")`
Get start and end dates for an academic year:
```r
academic_year_range("2026-27")
#> $start
#> [1] "2026-08-23"
#> $end
#> [1] "2027-08-22"
```

### 11. Built-in Calendar Presets

Include common calendar configurations as presets:

```r
# US Semester (Fall/Spring/Summer)
use_calendar_preset("us_semester")

# US Quarter (Fall/Winter/Spring/Summer)
use_calendar_preset("us_quarter")

# UK Terms (Michaelmas/Lent/Easter)
use_calendar_preset("uk_terms")

# Trimester (Fall/Winter/Spring)
use_calendar_preset("trimester")
```

Users can view and modify presets:
```r
show_calendar_preset("us_semester")
#> periods:
#>   Fall: fa, starts 08-25
#>   Spring: sp, starts 01-15
#>   Summer: su, starts 06-01
#> ay_start_period: Fall
```

## Technical Requirements

### Dependencies
- **Imports:** None (base R only for core functionality)
- **Suggests:** lubridate (for testing/examples), testthat, knitr, rmarkdown

### Code Structure
```
academicr/
├── R/
│   ├── period-class.R         # S3 class definition
│   ├── period-config.R        # Calendar configuration & management
│   ├── period-parse.R         # Parsing functions
│   ├── period-accessors.R     # Getter functions
│   ├── period-format.R        # Formatting functions
│   ├── period-arithmetic.R    # Math operations
│   ├── period-compare.R       # Comparison operators
│   ├── period-coerce.R        # Coercion functions
│   ├── period-validate.R      # Validation functions
│   ├── period-utils.R         # Utility functions
│   ├── calendar-presets.R     # Built-in calendar configurations
│   └── zzz.R                  # Package initialization (load default calendar)
├── tests/
│   └── testthat/
│       ├── test-config.R
│       ├── test-parse.R
│       ├── test-parse-yyyym.R
│       ├── test-format.R
│       ├── test-arithmetic.R
│       ├── test-compare.R
│       ├── test-calendars.R
│       └── test-edge-cases.R
├── data/
│   └── calendar_presets.rda   # Built-in calendar configurations
├── man/                       # Documentation
├── vignettes/
│   ├── introduction.Rmd
│   ├── calendar-configuration.Rmd
│   └── advanced-usage.Rmd
├── DESCRIPTION
├── NAMESPACE
└── README.md
```

### Testing Requirements
- Unit tests for all parsing formats (code, numeric, text)
- YYYYM edge cases: 5 vs 6 digits, ambiguous months, invalid months
- Calendar configuration validation
- Multiple calendar support
- Edge cases: year boundaries, invalid inputs, missing calendars
- Vectorization tests (all functions handle vectors)
- Arithmetic edge cases (wrapping years, multi-year sequences)
- Round-trip conversion tests (parse → format → parse)
- Comparison and sorting correctness across calendars
- Configuration persistence
- Preset calendar functionality

### Performance Considerations
- Calendar configurations stored in package options (fast lookup)
- Vectorized operations for parsing/formatting large datasets
- Lazy validation (don't re-validate on every operation)

## Documentation Requirements

### Package-level Documentation
- README with installation, quick start, multiple calendar examples
- Vignette: "Introduction to academicr"
- Vignette: "Configuring Academic Calendars"
- Vignette: "Advanced Usage and Custom Formats"
- Vignette: "Working with Multiple Institutions"

### Function Documentation
- All exported functions must have roxygen2 docs
- Include `@examples` for all user-facing functions
- Cross-reference related functions
- Document S3 methods explicitly
- Include examples using different calendar configurations

### Examples
```r
# README quick start example
library(academicr)

# Use default US semester calendar
# (or configure your institution's calendar first)
set_academic_calendar(
  periods = list(
    list(name = "Fall", code = "fa", start_date = "08-23"),
    list(name = "Spring", code = "sp", start_date = "01-15"),
    list(name = "Summer", code = "su", start_date = "06-01")
  ),
  ay_start_period = "Fall"
)

# Parse various formats
periods <- period(c("fa26", "20272", "Spring 2027", "202611"))

# Extract components
period_name(periods)
#> [1] "Fall"   "Spring" "Spring" "Fall"

period_ay(periods)
#> [1] "2026-27" "2026-27" "2026-27" "2026-27"

# Format for analysis
format(periods, format = "key")
#> [1] "2026-27_20268_Fall"     "2026-27_20272_Spring"  
#> [3] "2026-27_20272_Spring"   "2026-27_202611_Fall"

# Arithmetic
period("fa26") + 3  
#> <academic_period[1]> Fall 2027

seq(period("fa26"), period("sp28"), by = 1)
#> <academic_period[7]>
#> [1] Fall 2026   Spring 2027 Summer 2027 Fall 2027  
#> [5] Spring 2028 Summer 2028 Fall 2028

# Sorting (chronological within academic years)
sort(c(period("sp27"), period("fa26"), period("su26")))
#> <academic_period[3]>
#> [1] Fall 2026   Summer 2026 Spring 2027

# Work with data
library(dplyr)
df <- tibble(
  semester = c("fa26", "sp27", "su27", "fa27"),
  enrollment = c(1200, 1150, 450, 1300)
)

df %>%
  mutate(
    period_obj = period(semester),
    ay = period_ay(period_obj),
    sort_key = format(period_obj, "key")
  ) %>%
  arrange(sort_key)
```

## Success Criteria
1. Parses all common period notation formats correctly
2. Supports diverse calendar systems through configuration
3. Handles YYYYM format with variable length (5-6 digits)
4. Validates calendar configurations and warns about ambiguities
5. Sorts chronologically within and across academic years
6. Zero external dependencies for core functionality
7. >90% test coverage
8. Clear, comprehensive documentation with multi-calendar examples
9. CRAN-ready package structure
10. Handles edge cases gracefully (ambiguous YYYYM, missing calendars, etc.)

## Migration Path for Existing Users
For institutions with existing YYYYM codes:
1. Configure calendar to match existing system
2. Use `validate_academic_calendar()` to check for issues
3. Use `set_yyyym_mapping()` if needed for ambiguous months
4. Batch convert existing data: `period(old_codes)`

## Future Considerations (Out of Scope for v1.0)
- Period duration/end date configuration for midpoint calculations
- Integration with academic calendar APIs (Ellucian, Workday, etc.)
- Support for irregular periods (e.g., 7-week intensive courses)
- Fiscal year alignment (when different from academic year)
- Multi-institution datasets with mixed calendars
- Time zone handling for global institutions
- Localization of period names (i18n)

## Open Questions for v1.0
1. ✅ RESOLVED: Use variable-length YYYYM (5-6 digits) instead of YYYYM[M]
2. ✅ RESOLVED: Generalize to `academic_period` instead of `semester`
3. ✅ RESOLVED: Use specific start dates (MM-DD) not just months
4. Should default calendar be US semester or require explicit configuration?
5. How to handle Southern Hemisphere (academic year starts in January/February)?
6. Should we include academic year end dates in configuration or compute from next period's start?
7. Default behavior when no calendar configured: error or load US semester preset?

## Design Decisions Log

### Variable-Length YYYYM Format
**Decision:** Use 5 digits for months 1-9 (e.g., `20268`) and 6 digits for months 10-12 (e.g., `202611`)
**Rationale:** Self-describing format, no special syntax needed, aligns with natural numeric representation
**Alternative considered:** YYYYM[M] notation with brackets - rejected as less intuitive

### Generalized Period vs. Semester
**Decision:** Use `academic_period` as primary abstraction, not `semester`
**Rationale:** Supports diverse institutional calendars (quarters, trimesters, terms, custom)
**Trade-off:** Slightly more verbose but significantly more useful across institutions

### Specific Start Dates
**Decision:** Store MM-DD format (not just month) in calendar configuration
**Rationale:** Periods often start mid-month (e.g., January 15, August 23)
**Implication:** YYYYM format maps to "period starting in that month"

### YYYYM Ambiguity Handling
**Decision:** Configurable `yyyym_strict` flag + explicit mapping option
**Rationale:** Balances strictness for data integrity with flexibility for real-world calendars
**Implementation:** Error if strict, warn + use mapping if lenient

---

**Version:** 2.0 (Updated with generalized calendar support)
**Last Updated:** 2026-02-12
**Author:** Product requirements for Claude Code implementation
