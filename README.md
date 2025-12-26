# Budget Analyzer

Analyze credit card and bank statements with automatic merchant categorization.

## Features

- **900+ built-in merchant rules** - Common merchants pre-categorized out of the box
- **Pattern-based matching** - Python regex patterns for flexible merchant matching
- **Smart heuristics** - Auto-cleans transaction descriptions, detects locations, identifies travel
- **Multiple format support** - AMEX, Bank of America, or any CSV with custom format strings
- **Agent-friendly** - Designed to work with AI assistants for iterative rule building

## Installation

```bash
uv tool install git+https://github.com/davidfowl/budget-analyzer
```

## Quick Start

1. Initialize a new budget directory:
```bash
budget-analyze init ./my-budget
```

2. Add your statement files to `my-budget/data/`

3. Edit `my-budget/config/settings.yaml` to configure your data sources

4. Run the analyzer:
```bash
cd my-budget
budget-analyze run
```

5. Open `output/spending_summary.html` in your browser.

## Directory Structure

```
my-budget/
├── config/
│   ├── settings.yaml           # Main settings
│   └── merchant_categories.csv # Custom merchant rules (optional)
├── data/
│   ├── amex-2025.csv          # Your AMEX export
│   └── boa-checking.txt       # Your BOA statement
└── output/                     # Generated reports
```

## CLI Commands

```bash
# Show help
budget-analyze

# Initialize a new budget directory
budget-analyze init [dir]              # default: current directory

# Run the analyzer
budget-analyze run [config_dir]        # default: ./config
budget-analyze run --summary           # summary only, no HTML
budget-analyze run --quiet             # minimal output
budget-analyze run -s settings-2024.yaml  # use alternate settings file
budget-analyze run -o report.html      # custom output path

# Inspect a CSV file to determine its format
budget-analyze inspect data/bank.csv   # show structure and suggest format string

# Discover unknown merchants
budget-analyze discover                # human-readable output
budget-analyze discover --format csv   # CSV for copy-paste into rules
budget-analyze discover --format json  # JSON for programmatic use
budget-analyze discover --limit 50     # show top 50 by spend

# Detailed configuration help
budget-analyze --help-config
```

## Configuration

### settings.yaml

```yaml
year: 2025
title: "Spending Analysis"

data_sources:
  - name: AMEX
    file: data/amex-2025.csv
    type: amex
  - name: BOA
    file: data/checking.txt
    type: boa

output_dir: output
html_filename: spending_summary.html

# Optional: specify home locations (auto-detected if not set)
# Transactions from other US states are NOT auto-classified as travel
# International transactions ARE auto-classified as travel
home_locations:
  - WA
  - OR    # Include nearby states you don't want counted as travel

# Optional: pretty names for travel destinations in reports
travel_labels:
  HI: Hawaii
  GB: United Kingdom
```

### Custom CSV Formats

For banks other than AMEX or BOA, use a format string to map columns:

```yaml
data_sources:
  - name: Chase
    file: data/chase.csv
    format: "{date:%m/%d/%Y}, {_}, {description}, {_}, {amount}"
```

Use `budget-analyze inspect data/file.csv` to see the column structure and get a suggested format string.

**Format string tokens:**
- `{date:%m/%d/%Y}` - Date column with strptime format
- `{description}` - Transaction description
- `{amount}` - Amount column
- `{location}` - Optional location/state column
- `{_}` - Skip this column

### merchant_categories.csv

Pattern-based merchant categorization using Python regular expressions:

```csv
Pattern,Merchant,Category,Subcategory
NETFLIX,Netflix,Subscriptions,Streaming
COSTCO,Costco,Food,Grocery
AMAZON,Amazon,Shopping,Online
```

**Pattern syntax:**

| Pattern | Matches | Example |
|---------|---------|---------|
| `NETFLIX` | Contains "NETFLIX" | Simple substring match |
| `^ATT\s` | Starts with "ATT " | Avoid matching "SEATTLE" |
| `UBER\s(?!EATS)` | "UBER " not followed by "EATS" | Exclude Uber Eats |
| `COSTCO(?!\s*GAS)` | "COSTCO" not followed by "GAS" | Exclude Costco Gas |
| `DELTA\|SOUTHWEST` | Either "DELTA" or "SOUTHWEST" | Match multiple variations |
| `CHICK.FIL.A` | "CHICK-FIL-A" or "CHICKFILA" | `.` matches any character |

**Tips:**
- Lines starting with `#` are comments
- First match wins - put specific rules before general ones
- Test patterns at [regex101.com](https://regex101.com/) (select Python flavor)
- Use `budget-analyze inspect <file>` to see exact transaction descriptions

## Built-in Heuristics

The analyzer applies several automatic transformations:

**Description cleaning:**
- Removes common prefixes: `APLPAY`, `SQ *`, `TST*`, `SP `, `PP*`, `GOOGLE *`
- Strips trailing store IDs, zip codes, and reference numbers
- Normalizes whitespace and casing for matching

**Location detection:**
- Extracts US state codes and country codes from transaction descriptions
- Auto-detects your home location from transaction frequency
- International transactions are automatically flagged as travel

**Travel classification:**
- International locations → Travel (automatic)
- Domestic out-of-state → NOT auto-travel (opt-in via merchant rules)
- Configure `home_locations` to control what counts as "home"

## Supported Statement Formats

- **AMEX**: CSV export with Date, Description, Amount columns
- **BOA**: Text statement with `MM/DD/YYYY Description Amount Balance` format
- **Custom CSV**: Any CSV using format strings (see above)

## Working with AI Agents

Budget Analyzer is designed to work well with AI coding assistants like Claude Code. The iterative workflow of discovering and classifying unknown merchants is ideal for agent assistance.

### Why Agents Excel at This

Classifying merchants requires judgment that AI agents handle well:
- **Context awareness** - "WHOLEFDS MKT" is obviously Whole Foods (grocery)
- **Pattern recognition** - Spotting that "SQ *JOES COFFEE" is a Square payment at a coffee shop
- **Disambiguation** - Knowing "APPLE.COM/BILL" is a subscription, not shopping
- **Local knowledge** - Recognizing that "DICKS DRIVE IN" is a Seattle restaurant, not sporting goods

### Recommended Workflow

1. **Initial setup** - Run `budget-analyze init` and configure your data sources

2. **First analysis** - Run `budget-analyze run --summary` to see categorization coverage

3. **Discover unknown merchants:**
   ```bash
   budget-analyze discover --format json
   ```
   This outputs unknown merchants sorted by total spend, with suggested patterns.

4. **Iterative classification with an agent:**
   ```
   You: "Help me categorize my unknown transactions"

   Agent runs: budget-analyze discover --format json
   Agent analyzes each unknown merchant:
     - Identifies what the merchant is (restaurant, store, service, etc.)
     - Determines appropriate Category and Subcategory
     - Creates a regex pattern that matches the merchant
   Agent adds rules to merchant_categories.csv
   Agent runs: budget-analyze run --summary
   Agent repeats until Unknown < 5%
   ```

5. **Agent-assisted tasks:**
   - Classifying unknown merchants by spending context and name
   - Creating regex patterns for tricky merchant names
   - Identifying transaction patterns (subscriptions, recurring charges)
   - Separating merchants by type (e.g., Costco groceries vs Costco gas)
   - Setting up custom CSV formats for new bank exports
   - Handling P2P payments (Venmo, Zelle) with recipient-specific rules

### The discover Command

The `discover` command is specifically designed for agent workflows:

```bash
# Human-readable for manual review
budget-analyze discover

# JSON for agents - includes suggested patterns and merchant names
budget-analyze discover --format json

# CSV ready to copy into merchant_categories.csv (just add categories)
budget-analyze discover --format csv
```

**JSON output includes:**
- `raw_description` - Original transaction text
- `suggested_pattern` - Regex pattern to match it
- `suggested_merchant` - Clean merchant name
- `count` - Number of transactions
- `total_spend` - Total amount (prioritize high-spend unknowns)

### Files for Agents

When you run `budget-analyze init`, it creates:
- `AGENTS.md` - Detailed instructions for AI agents with pattern examples and category lists
- `CLAUDE.md` - Claude Code specific context

These files help agents understand the project structure, pattern syntax, and common workflows.

### Example Agent Prompts

- "Help me categorize my unknown transactions"
- "Run the budget analyzer and classify any unknowns"
- "Add rules for all my subscription services"
- "Separate my Uber rides from Uber Eats orders"
- "Set up parsing for this new Chase CSV export"
- "Which merchants am I spending the most on that aren't categorized?"

## Development

```bash
# Clone and install in development mode
git clone https://github.com/davidfowl/budget-analyzer
cd budget-analyzer
uv sync

# Run locally
uv run budget-analyze init ./test-budget
uv run budget-analyze run ./test-budget/config
```

## License

MIT
