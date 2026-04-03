# Product Catalog Data Preprocessing

## Overview

This folder contains a data pipeline for extracting, structuring, and indexing LED product specifications from PDF datasheets. The pipeline consists of two main notebooks that work sequentially to convert unstructured PDF documents into a queryable database and semantic search index.

## Setup

### 1. Create Virtual Environment

```bash
cd src/02.product-catalog-data-preprocessing
python -m venv .venv
```

### 2. Activate Virtual Environment

On macOS/Linux:
```bash
source .venv/bin/activate
```

On Windows:
```bash
.venv\Scripts\activate
```

### 3. Install Requirements

```bash
pip install -r requrements.txt
```

The main dependencies include:
- pymupdf4llm: PDF to markdown conversion
- faiss-cpu: Vector similarity search
- sentence-transformers: Text embeddings
- sqlite3: Database storage (built-in with Python)

## Data Pipeline

### Input

Place PDF datasheets in the `data/` folder. These should be technical specification documents for LED products.

### Pipeline Overview

```
data/*.pdf
    |
    v
[data_preprocessing.ipynb]
    |
    v
data_processed/pages_with_tables.json
    |
    v
[data_ingestion.ipynb]
    |
    v
data_processed/products.db
data_processed/faiss_index/
```

## Notebook 1: data_preprocessing.ipynb

### Purpose

Extracts structured data from PDF documents and creates an intermediate JSON representation with tables and metadata.

### What It Does

#### Step 1: PDF to Markdown Conversion
- Reads all PDF files from the `data/` folder
- Uses pymupdf4llm to convert each PDF page to markdown format
- Preserves page numbers and document metadata

#### Step 2: Table Extraction
- Parses markdown text to identify tables (header row, separator row, data rows)
- Normalizes table headers (lowercase, underscore-separated, alphanumeric)
- Extracts rows as dictionaries with header-to-value mappings

#### Step 3: Product Association
- Identifies target products from tables containing "commercial_product_name" column
- Stores these as document-level metadata (`targeted_products` field)

#### Step 4: Context-Based Table Linking
- For each table, examines the preceding 20 lines of text
- Searches for mentions of specific product names in that context
- Associates tables with related products when explicit mentions are found
- Falls back to document-level products when no specific mention exists
- Adds `related_products` column to tables (except those already containing product names)

#### Step 5: Persistence
- Saves the complete structure to `data_processed/pages_with_tables.json`
- Each document contains pages, each page contains tables, each table contains headers and rows

### Output Structure

```json
[
  {
    "document_name": "product_spec.pdf",
    "targeted_products": "Fortimo LED HV5, Xitanium Driver",
    "pages": [
      {
        "page_num": 1,
        "text": "full markdown text...",
        "tables": [
          {
            "headers": ["parameter", "unit", "min", "typ", "max"],
            "rows": [
              {
                "parameter": "Input Voltage",
                "unit": "V",
                "min": "220",
                "typ": "230",
                "max": "240",
                "related_products": "Fortimo LED HV5"
              }
            ]
          }
        ]
      }
    ]
  }
]
```

### Key Functions

- `parse_tables_from_markdown()`: Extracts tables from markdown text
- `split_md_row()`: Parses markdown table rows
- `normalize_header()`: Cleans and standardizes column names
- `find_products_in_context()`: Matches product names in text context
- `get_pre_table_contexts()`: Extracts N lines before each table

## Notebook 2: data_ingestion.ipynb

### Purpose

Transforms the intermediate JSON structure into a queryable SQLite database and creates a FAISS vector index for semantic search.

### What It Does

#### Step 1: Load Intermediate Data
- Reads `data_processed/pages_with_tables.json`
- Flattens structure into pages and tables for processing

#### Step 2: Header Anomaly Detection and Fixing
- Identifies tables with generic column names (`col1`, `col2`, etc.)
- Applies intelligent renaming: if `col1` follows "voltage", renames to `voltage_1`
- Updates both headers and row dictionaries to maintain consistency

#### Step 3: Table Classification
- Analyzes each table's headers and content to determine its semantic type
- Classification categories:
  - `ordering_data`: Tables with commercial_product_name, EOC codes, 12NC
  - `photometric_by_cct`: Performance data by color temperature (CCT codes like 830, 840)
  - `temperature_tuning`: Performance across different case temperatures
  - `current_tuning`: Performance at different input currents
  - `lumen_maintenance`: Lifetime data (L70, L80, L90 metrics)
  - `parameter_*`: Various specification formats (min/typ/max, nominal, value-only)
  - `wiring_spec`: Installation specifications
  - `application`: Use case descriptions

#### Step 4: Database Schema Creation
Creates SQLite database (`data_processed/products.db`) with four main tables:

**product**
- Core product identity: name, EOC, 12NC, EPREL registration
- Box quantity, source document, page number

**product_spec**
- Semi-structured specifications extracted from parameter tables
- Flexible schema accommodating min/max/nominal/value/condition fields
- Stores raw JSON for audit trail
- Handles diverse spec types: electrical, optical, thermal, mechanical

**product_performance**
- Structured photometric and electrical performance data
- Luminous flux, efficacy, input current
- Operating point, temperature, color code

**product_lifetime**
- Lumen maintenance metrics: L70/L80/L90 with B10/B20/B50 variants
- Operating conditions and temperature data

#### Step 5: Data Ingestion
- Iterates through all classified tables
- Routes data to appropriate database table based on classification
- Creates products on-the-fly (with caching to avoid duplicates)
- Links specifications, performance, and lifetime data to products via foreign keys

#### Step 6: FAISS Vector Index Creation
- Loads purpose/design text extractions from `purpose_design_extractions.json`
- Generates embeddings using sentence-transformers (all-MiniLM-L6-v2 model)
- Creates FAISS L2 distance index for semantic similarity search
- Stores index and metadata in `data_processed/faiss_index/`
- Enables deterministic behavior with fixed seeds and evaluation mode

### Output Artifacts

#### 1. SQLite Database: `data_processed/products.db`
- Queryable with standard SQL
- Indexed on product names and foreign keys
- Contains structured and semi-structured data

#### 2. FAISS Index: `data_processed/faiss_index/`
- `faiss.index`: Vector similarity search index
- `metadata.pkl`: Python pickle with documents and metadata
- `metadata.json`: Human-readable version

### Key Classification Logic

```python
# Example: Photometric table detection
if any_cct_code_in_headers and "lm" in headers:
    return "photometric_by_cct"

# Example: Lifetime table detection
if headers_contain("l70") or headers_contain("l80"):
    return "lumen_maintenance"
```

### Key Helper Functions

- `classify_table()`: Determines table type from structure
- `get_or_create_product()`: Product deduplication and creation
- `to_real()`: Safe numeric conversion with null handling
- `split_products()`: Parses comma-separated product lists

## Running the Pipeline

### Step-by-Step Execution

1. Place PDF files in `data/` folder
2. Open and run all cells in `data_preprocessing.ipynb`
   - Verify output: `data_processed/pages_with_tables.json` created
3. Open and run all cells in `data_ingestion.ipynb`
   - Verify outputs:
     - `data_processed/products.db` created
     - `data_processed/faiss_index/` folder populated

### Verification Queries

After running data_ingestion.ipynb, you can verify the database:

```python
import sqlite3

conn = sqlite3.connect("data_processed/products.db")
cur = conn.cursor()

# Count products
cur.execute("SELECT COUNT(*) FROM product")
print(f"Products: {cur.fetchone()[0]}")

# Count specifications
cur.execute("SELECT COUNT(*) FROM product_spec")
print(f"Specs: {cur.fetchone()[0]}")

# Sample query
cur.execute("""
    SELECT commercial_name, eoc 
    FROM product 
    LIMIT 5
""")
for row in cur.fetchall():
    print(row)

conn.close()
```

## Data Quality Considerations

### Handled Automatically

- Duplicate product detection and merging
- Generic column name disambiguation
- Multi-format specification handling
- Numeric value parsing with error handling
- Context-based table-product association

### Requires Domain Knowledge

- Table classification rules are based on LED product domain patterns
- May need adjustment for other product categories
- Column name normalization assumes specific naming conventions

## Notes

- The pipeline is deterministic with fixed random seeds for embeddings
- FAISS uses L2 distance (not cosine similarity)
- Product names are cached to avoid duplicate database entries
- All tables retain source document and page metadata for traceability
