# Product Catalog Agent System

## Overview

This folder contains an intelligent multi-agent system for querying LED product specifications using a hybrid approach combining structured SQL databases and semantic vector search. The system uses Microsoft Agent Framework to orchestrate between a specialized SQL agent and a FAISS-based semantic search engine.

## Architecture

The system consists of two primary agents that work together:

```
User Query
    |
    v
Orchestrator Agent
    |
    +---> tokenize_search_text_with_llm (LLM query normalization)
    |
    +---> search_local_faiss_index (semantic search for purpose/design)
    |
    +---> query_product_database
              |
              v
          SQL Agent
              |
              +---> get_database_schema_string (enriched schema)
              +---> tokenize_search_term (simple tokenization)
              +---> build_like_clause (SQL LIKE generation)
              +---> execute_database_query (SQL execution)
```

## Prerequisites

Before running this notebook, you must complete the data preprocessing pipeline:

1. Run `../02.product-catalog-data-preprocessing/data_preprocessing.ipynb`
2. Run `../02.product-catalog-data-preprocessing/data_ingestion.ipynb`

This creates:
- `data/products.db` (SQLite database)
- `data/faiss_index/` (FAISS vector index)

## Setup

### 1. Create Virtual Environment

```bash
cd src/03.product-catalog-agent-thought-process
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

Key dependencies:
- agent-framework: Microsoft Agent Framework for agent orchestration
- openai: Azure OpenAI client
- faiss-cpu: Vector similarity search
- sentence-transformers: Text embeddings
- python-dotenv: Environment variable management

### 4. Configure Environment Variables

Create a `.env` file in this directory:

```bash
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
AZURE_OPENAI_API_KEY=your-api-key-here
```

## System Components

### Tools

#### 1. Database Schema Tools

**get_database_schema()**
- Dynamically retrieves schema from products.db
- Shows table names, columns, data types
- Returns random sample rows for each table
- Used for runtime schema inspection

**get_database_schema_string()**
- Returns enriched schema documentation
- Includes semantic interpretation of spec_name field
- Provides normalization categories (thermal, optical, electrical, etc.)
- Preferred by SQL agent for understanding data semantics

#### 2. Query Tools

**execute_database_query(query: str)**
- Executes SQL queries against products.db
- Returns results as formatted string table
- Handles errors and empty results gracefully
- Core execution tool for SQL agent

#### 3. Simple Tokenization Tools

**tokenize_search_term(search_term: str)**
- Splits text by whitespace into lowercase tokens
- Example: "Fortimo HV5" → ["fortimo", "hv5"]
- Used by SQL agent for basic text processing

**build_like_clause(tokens: list[str])**
- Constructs SQL LIKE clause from token list
- Example: ["fortimo", "hv5"] → "LOWER(product_name) LIKE '%fortimo%' AND LOWER(product_name) LIKE '%hv5%'"
- Handles SQL injection with quote escaping
- Used for fuzzy product name matching

#### 4. LLM-Powered Tokenization

**tokenize_search_text_with_llm(user_text: str)**
- Uses Azure OpenAI (GPT-4) to extract structured search terms
- Parses free-form text into product name and attribute
- Returns normalized query format: "{product} {attribute}"
- Deterministic with temperature=0.0 and seed=42
- Examples:
  - "what's the power of fortimo hv5?" → "fortimo hv5 power"
  - "show me the light for reading room" → "reading room light"

System prompt guides LLM to:
- Identify product names (e.g., fortimo hv5, xitanium, certadrive)
- Extract attributes (e.g., power, luminous flux, CCT)
- Normalize to lowercase
- Return valid JSON: `{"product": "...", "attribute": "..."}`

#### 5. Semantic Search

**search_local_faiss_index(search_term: str, top_n: int)**
- Searches FAISS vector index for semantically similar documents
- Uses sentence-transformers (all-MiniLM-L6-v2) for embeddings
- Returns purpose/design descriptions with product context
- Deterministic behavior (fixed seeds, evaluation mode)
- L2 distance metric (lower score = better match)

Configuration for determinism:
- Random seeds set (Python, NumPy, PyTorch)
- Model in evaluation mode
- Consistent encoding parameters (normalize_embeddings=False, batch_size=1)
- Same settings as index creation

### SQL Agent

**Purpose**: Specialized agent for querying the SQLite database with intelligent query generation.

**Configuration**:
- Model: GPT-4 (deployment name: gpt-4.1)
- Temperature: 0.0 (deterministic)
- Tool choice: required (must use tools)

**Instructions Summary**:

1. Always retrieve schema first (prefers enriched schema)
2. Never assumes table/column names
3. Generates valid SQLite SQL only

Search Strategy:
- For text matching: tokenize → build LIKE clause → search
- For specifications: use product IDs after initial match
- Prefers structured queries over repeated fuzzy matching

Query Rules:
- Always includes LIMIT clause (unless aggregation)
- Prefers simple over complex queries
- Uses explicit conditions
- Revises on errors without repeating mistakes

Output Rules:
- Generates SQL query first
- Executes via execute_database_query
- Never returns SQL without execution

**Tools Available**:
- get_database_schema_string
- tokenize_search_term
- build_like_clause
- execute_database_query

**Example Workflow**:
```
User: "What is the CCT of Fortimo LED HV5?"
  → SQL Agent gets schema
  → Tokenizes "fortimo led hv5"
  → Builds LIKE clause
  → Queries product table for product_id
  → Queries product_spec for CCT specification
  → Returns results
```

### Orchestrator Agent

**Purpose**: Top-level agent that coordinates between semantic search and SQL database queries to answer user questions.

**Configuration**:
- Model: GPT-4 (deployment name: gpt-4.1)
- Temperature: 0.0 (deterministic)
- Tool choice: required (must use tools)

**Instructions Summary**:

Has access to three complementary tools:
1. tokenize_search_text_with_llm - Query normalization
2. search_local_faiss_index - Semantic search for use cases
3. query_product_database - Structured data queries

Decision Strategy:

**A. Discovery/Recommendation Queries**
- Pattern: "What product is good for X?"
- Strategy:
  1. Search FAISS index (top 5-10 results)
  2. Extract product names from results
  3. Query database for detailed specs

**B. Specification Queries**
- Pattern: "What is the X of product Y?"
- Strategy:
  1. Query database directly with product name

**C. Complex Queries**
- Pattern: "Best product for X with specification Y"
- Strategy:
  1. Search FAISS for application match (X)
  2. Query database to filter by specification (Y)
  3. Combine and recommend

Response Rules:
- Explains search strategy briefly
- Shows intermediate results (product names, values)
- Combines results intelligently
- Tries alternative approaches if one fails
- Conversational but precise

**Tools Available**:
- tokenize_search_text_with_llm
- search_local_faiss_index
- query_product_database (wraps SQL Agent)

**Example Workflows**:

Use Case 1: Application-based search
```
User: "Which light is good for living room?"
  → Orchestrator searches FAISS for "living room residential lighting"
  → Returns product recommendations with design context
```

Use Case 2: Specification lookup
```
User: "What is the CCT of Fortimo LED HV5?"
  → Orchestrator delegates to query_product_database
  → SQL Agent executes query
  → Returns CCT specification value
```

Use Case 3: Complex filtering
```
User: "I need a bright light for retail, at least 2500 lumens"
  → Orchestrator searches FAISS for "retail lighting"
  → Extracts candidate products
  → Queries database: luminous_flux >= 2500
  → Combines results and recommends matching products
```

## Running the Notebook

### Cell Execution Order

The notebook is designed to be run sequentially from top to bottom:

#### Cells 1-2: Setup and Sample Tool
- Import dependencies (agent_framework, openai, dotenv)
- Load environment variables
- Sample weather tool (for demonstration)

#### Cells 3-4: Database Schema Tools
- get_database_schema() - dynamic schema retrieval
- get_database_schema_string() - enriched schema documentation
- Test output to verify database connection

#### Cell 5: Query Execution Tool
- execute_database_query() implementation
- Handles SQL execution and formatting

#### Cell 6: Simple Tokenization Tools
- tokenize_search_term() - whitespace splitting
- build_like_clause() - SQL LIKE generation

#### Cell 7: LLM-Powered Tokenization
- tokenize_search_text_with_llm() implementation
- Azure OpenAI client initialization
- Test cases with various query patterns

#### Cell 8: Test Query
- Sample query execution to verify tools work

#### Cell 9: SQL Agent Creation
- Agent configuration with instructions
- Tool registration
- Temperature and tool choice settings

#### Cell 10: FAISS Setup Header
- Documentation for deterministic configuration

#### Cell 11: FAISS Search Implementation
- search_local_faiss_index() tool
- Embedding model caching
- Deterministic seed configuration

#### Cells 12-13: FAISS Test
- Test query execution
- Result inspection

#### Cell 14: SQL Agent Wrapper
- query_product_database() tool
- Wraps SQL agent for orchestrator use

#### Cell 15: Orchestrator Agent Creation
- Agent configuration with instructions
- Registers three main tools
- Strategy documentation

#### Cell 16: Orchestrator Test
- Single test query execution
- Verifies orchestration logic

#### Cell 17: Interactive Chat Loop
- Conversational interface
- Maintains chat history
- Context preservation across turns

### Running Specific Sections

**To test just the SQL Agent**:
```python
# Run cells 1-9
sql_session = agent.create_session()
response = await agent.run("What is the efficacy of Fortimo products?", session=sql_session)
print(response)
```

**To test just FAISS search**:
```python
# Run cells 1-2, 10-11
result = search_local_faiss_index("kitchen workbench lighting", top_n=5)
print(result)
```

**To test the full orchestrator**:
```python
# Run all cells up to cell 16
test_session = orchestrator_agent.create_session()
test_response = await orchestrator_agent.run(
    "Which products are suitable for retail with high efficacy?",
    session=test_session
)
print(test_response)
```

**To run interactive chat**:
```python
# Run all cells including cell 17
# Interactive prompt will start automatically
# Type queries and get responses
# Type 'exit', 'quit', or 'q' to end
```

## Example Queries

### Application/Purpose Queries

```python
"Which products are good for retail environments?"
"Show me lights suitable for office workspaces"
"What products are designed for outdoor applications?"
"I need lighting for a kitchen workbench"
```

### Specification Queries

```python
"What is the CCT of Fortimo LED HV5?"
"Show me the power consumption of Xitanium drivers"
"What is the luminous flux of LED Strip products?"
"What are the dimensions of Certadrive modules?"
```

### Filtering Queries

```python
"Find products with luminous flux greater than 2000 lumens"
"Which products have CCT of 830?"
"Show products with efficacy above 120 lm/W"
"List products with nominal case temperature below 70°C"
```

### Complex Queries

```python
"I need a bright light for retail, at least 2500 lumens"
"Which office products have high efficacy and 840 CCT?"
"Show me outdoor products with long lifetime (L70 > 50000 hours)"
"What are the most efficient products for residential applications?"
```

## Architecture Patterns

### Hybrid Retrieval Strategy

The system implements a hybrid retrieval approach combining:

1. **Semantic Search (FAISS)**
   - Use case: Purpose, design intent, application matching
   - Strength: Understands natural language intent
   - Limitation: No precise filtering

2. **Structured Query (SQL)**
   - Use case: Specifications, filtering, exact values
   - Strength: Precise numerical/categorical filtering
   - Limitation: Requires knowing product names

3. **Combined Approach**
   - Step 1: Semantic search identifies candidates
   - Step 2: SQL filters by precise criteria
   - Result: Best of both worlds

### Agent Delegation Pattern

```
Orchestrator (Coordinator)
    ├── Handles high-level strategy
    ├── Understands user intent
    └── Delegates to specialized agents
            |
            └── SQL Agent (Specialist)
                    ├── Schema-aware query generation
                    ├── Error recovery
                    └── Result formatting
```

Benefits:
- Separation of concerns
- Specialized instructions per agent
- Easier debugging and improvement
- Reusable SQL agent for other tasks

### Deterministic Query Processing

Both the SQL agent and FAISS search are configured for deterministic behavior:

**SQL Agent**:
- Temperature: 0.0
- Seed: implicit via Azure OpenAI
- Strict instruction following

**FAISS Search**:
- Fixed random seeds (Python, NumPy, PyTorch)
- Model evaluation mode (no dropout)
- Consistent encoding parameters
- Same seed used during index creation

This ensures:
- Reproducible results for testing
- Consistent user experience
- Easier debugging and validation

## Troubleshooting

### Common Issues

**Error: "FAISS index file not found"**
- Cause: data_ingestion.ipynb not run
- Solution: Complete data preprocessing pipeline first

**Error: "no such table: product"**
- Cause: products.db not created or empty
- Solution: Run data_ingestion.ipynb to populate database

**Error: "AZURE_OPENAI_API_KEY not found"**
- Cause: .env file missing or incorrect
- Solution: Create .env with proper Azure OpenAI credentials

**Agent returns "I cannot answer this"**
- Cause: Data not available in database or index
- Solution: Verify data pipeline completed and data exists

**FAISS search returns unexpected results**
- Cause: Index created with different settings
- Solution: Recreate index with matching deterministic settings

## Performance Considerations

### Query Optimization

**SQL Agent**:
- Caches product IDs to avoid repeated lookups
- Uses indexed columns (product_id, commercial_name)
- Includes LIMIT clauses automatically
- Prefers simple queries over complex joins

**FAISS Search**:
- Model caching (first call slower, subsequent fast)
- Configurable top_n parameter (default 5)
- L2 distance computation (fast)
- No re-ranking or post-processing

### Latency Breakdown

Typical query times:
- SQL Agent: 1-3 seconds (includes LLM call + query execution)
- FAISS Search: 0.5-1 second (embedding + search)
- Orchestrator: 2-5 seconds (strategy + delegation + combination)

For production:
- Consider caching common queries
- Pre-compute embeddings for known query patterns
- Use streaming responses for better UX

## Extending the System

### Adding New Tools

```python
@tool(approval_mode="never_require")
def your_new_tool(
    param: Annotated[str, Field(description="Parameter description")]
) -> str:
    """Tool description for the agent."""
    # Implementation
    return result

# Register with agent
agent = Agent(
    name="Agent_Name",
    instructions="...",
    client=client,
    tools=[existing_tools, your_new_tool]
)
```

### Modifying Agent Instructions

Edit the `instruction` or `orchestrator_instructions` strings to:
- Add new decision patterns
- Change tool selection strategy
- Improve response formatting
- Add domain-specific rules

### Adding New Data Sources

To integrate additional data:
1. Create a new tool that queries the source
2. Document the tool's purpose and output format
3. Update orchestrator instructions with decision rules
4. Register tool with orchestrator agent

## Notes

- The system uses asynchronous execution (async/await) for agent runs
- Sessions preserve context between queries in chat mode
- Tools are executed by agents autonomously based on instructions
- Temperature 0.0 provides deterministic but may be less creative
- Tool approval mode is "never_require" for demo (use "always_require" in production)