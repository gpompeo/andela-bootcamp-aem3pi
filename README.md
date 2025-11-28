# Multi-Agent RAG System for SaaS Support

An intelligent multi-agent system that routes user queries to specialized AI agents (HR, Tech, Commercial, Finance) using RAG (Retrieval Augmented Generation), orchestration, and automatic evaluation.

## Features

- **Multi-Agent Architecture**: Four specialized agents handling different support domains
- **Intelligent Routing**: Orchestrator classifies intent and routes queries to appropriate agents
- **Query Decomposition**: Automatically splits complex multi-domain queries into focused sub-queries
- **RAG Implementation**: Vector-based retrieval using Qdrant with OpenAI embeddings
- **Response Consolidation**: Merges multiple agent responses into coherent answers
- **Automated Evaluation**: LLM-as-judge system scoring responses on relevance, completeness, accuracy, and groundedness
- **Langfuse Integration**: Full observability and tracing of agent interactions
- **Workflow Visualization**: LangGraph-based state management with visual workflow

## Architecture

### Agent Domains

- **HR Agent**: Payroll, vacation days, sick leave, benefits, employment policies
- **Tech Agent**: Password resets, VPN, connectivity issues, software licenses, IT tickets
- **Commercial Agent**: Pricing, plans, upgrades, downgrades, cancellations, billing
- **Finance Agent**: Payment methods, invoice status, reimbursements, financial policies

### Workflow

```
User Query → Orchestrator → Intent Classification
                ↓
         Needs Clarification?
         ↙            ↘
    Yes: Return        No: Route to Agents
    clarification            ↓
                      Query Decomposition (if multi-agent)
                             ↓
                      Parallel Agent Execution
                             ↓
                      Response Consolidation
                             ↓
                      Evaluation & Scoring
                             ↓
                      Final Response
```

## Setup

### Prerequisites

- **Python**: 3.12.x
- **API Keys**:
  - OpenAI API key (required)
  - Langfuse API keys (optional, for monitoring)

### Installation

1. **Clone the repository** or navigate to project directory

2. **Create virtual environment**:

```bash
python -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
```

Or with uv:

```bash
uv venv --python 3.12
source .venv/bin/activate
```

3. **Install dependencies**:

```bash
pip install -r requirements.txt
```

Or with uv:

```bash
uv pip install -r requirements.txt
```

### Environment Variables

Create a `.env` file in the project root with the following keys:

```env
# Required
OPENAI_API_KEY=your_openai_api_key_here

LANGFUSE_PUBLIC_KEY=your_langfuse_public_key
LANGFUSE_SECRET_KEY=your_langfuse_secret_key
LANGFUSE_HOST=https://cloud.langfuse.com
```

**Note**: Without Langfuse keys, the system will still work but monitoring/tracing will be disabled.

## Project Structure

```
project/
├── data/
│   ├── hr_agent_docs.md           # HR knowledge base
│   ├── tech_agent_docs.md         # Tech knowledge base
│   ├── commercial_agent_docs.md   # Commercial knowledge base
│   ├── finance_agent_docs.md      # Finance knowledge base
│   └── qdrant_db/                 # Vector database (auto-created)
├── src/
│   └── multi_agent_system.ipynb   # Main implementation notebook
├── test_queries/
│   └── test_queries.json          # JSON file with queries and expected intent
├── .env                           # API keys (create from .env.example)
├── requirements.txt               # Python dependencies
└── README.md                      # This file
```

## Usage

### Running the Notebook

Open `multi_agent_system.ipynb` in Jupyter and execute cells in order:

#### 1. Setup & Imports (Section 1)

Run cells to load dependencies and environment variables.

#### 2. Document Loading (Section 2)

Run cells to create/load vector stores:

- Creates Qdrant vector databases for each agent
- Loads markdown documents and chunks them
- Generates embeddings using OpenAI
- Persists to `data/qdrant_db/`

**First time only**: This creates the vector stores. Subsequent runs will load existing stores.

```python
# This cell creates or loads all agent vector stores
vectorstores = load_all_agents(agent_dic)
```

Expected output:

```
✓ Loaded existing vector store 'hr_agent' with 52 chunks
✓ Loaded existing vector store 'tech_agent' with 56 chunks
...
```

#### 3. Agent Definition (Section 3)

Run cells to initialize LLMs, prompts, and RAG chains:

- Sets up deterministic LLM (temperature=0) for orchestration
- Sets up creative LLM (temperature=0.3) for agent responses
- Creates specialized RAG chains for each agent

#### 4. Orchestration & Routing (Section 4)

Run cells to build the LangGraph workflow:

- Defines state management
- Creates orchestrator, routing, consolidation, and evaluator nodes
- Compiles the workflow with memory

#### 5. Visualization (Section 4.5)

Optionally run to see the workflow diagram:

```python
visualize_workflow()
```

#### 6. Testing & Examples (Section 5)

Run test cases to see the system in action:

**Simple Query (Single Agent)**:

```python
result = process_query("What is the vacation policy?")
```

**Multi-Domain Query**:

```python
result = process_query("I need to reset my password and schedule my vacations.")
```

**Query Requiring Clarification**:

```python
result = process_query("I have a problem")
```

### Query Response Structure

Each query returns a dictionary with:

```python
{
    "query": "original user question",
    "intent": {"action": "route", "agents": ["HR", "Tech"]},
    "agent_queries": {"HR": "specific HR question", "Tech": "specific Tech question"},
    "agent_responses": {"HR": "HR answer", "Tech": "Tech answer"},
    "final_response": "consolidated response",
    "agents_used": ["HR", "Tech"],
    "evaluation_scores": {"HR": {...}, "Tech": {...}},
    "needs_clarification": false,
    "conversation_id": "unique_id"
}
```

### Evaluation Metrics

The system automatically evaluates each agent response on:

- **Relevance** (1-10): Does the answer address the query?
- **Completeness** (1-10): Is all necessary information included?
- **Accuracy** (1-10): Is the answer factually correct in the given context?
- **Groundedness** (1-10): Is the answer based solely on provided context?
- **Overall**: Average of all scores

### Example Usage

```python
# Process a query
result = process_query(
    query="How do I request vacation time?",
    conversation_id="chat_123",  # Optional: for conversation tracking
    verbose=True                  # Optional: print detailed agent responses
)

# Access results
print(result['final_response'])
print(f"Agents used: {result['agents_used']}")
print(f"Scores: {result['evaluation_scores']}")
```

## Configuration

### Agent Customization

To modify agent behavior, edit the prompts in Section 3.1:

- `INTENT_SYSTEM_PROMPT`: Controls intent classification
- `QUERY_DECOMPOSITION_PROMPT`: Controls multi-agent query splitting
- `CONSOLIDATION_PROMPT`: Controls response merging
- `create_rag_prompt()`: Controls individual agent responses

### LLM Settings

Adjust temperature and model in Section 3:

```python
llm_deterministic = ChatOpenAI(model="gpt-4o-mini", temperature=0)  # Orchestration, consolidation, and evaluation
llm_rag = ChatOpenAI(model="gpt-4o-mini", temperature=0.3)     # RAG Agents
```

### Vector Store Settings

Modify retrieval parameters in Section 3.2:

```python
    retriever = vectorstore.as_retriever(
        search_type="similarity_score_threshold",
        search_kwargs={"k": 5,   # Number of chunks
        "score_threshold": SIMILARITY_THRESHOLD}   # minimum similarity score to return chunks
    )
```

### Knowledge Base Updates

To update agent knowledge:

1. Edit the corresponding markdown file in `data/`
2. Delete the `data/qdrant_db/` folder
3. Re-run Section 2 cells to rebuild vector stores

## Test Cases Included

The notebook includes 14 test cases covering:

1. Simple single-agent queries (HR, Tech, Finance, Commercial)
2. Multi-domain queries requiring multiple agents
3. Unclear queries requiring clarification
4. Queries with escalation requirements
5. Complex multi-agent scenarios
6. Queries outside all domains
7. Broad queries needing disambiguation

## Known Limitations

### 1. **No Real-time Vector Store Updates**

- Vector stores must be rebuilt manually when knowledge bases change
- No incremental updates supported
- **Workaround**: Delete `data/qdrant_db/` and re-run Section 2

### 2. **Memory Limitations**

- MemorySaver only persists within notebook session
- Conversation history lost on kernel restart
- **Future**: Implement persistent storage (SQLite, PostgreSQL)

### 3. **Single Language Support**

- System prompts and knowledge bases are English-only
- No multi-language support currently implemented

### 4. **Evaluation Latency**

- Each query incurs additional LLM call for evaluation
- Adds ~1-2 seconds per query
- **Note**: Can be disabled by removing evaluator node

### 5. **Cost Considerations**

- Each query may call multiple LLMs:
  - Intent classification (1 call)
  - Query decomposition (1 call if multi-agent)
  - Agent responses (1-4 calls)
  - Consolidation (1 call if multi-agent)
  - Evaluation (1-4 calls)
- **Typical cost**: $0.0001-0.0005 per query

### 6. **Fixed Agent Domains**

- Agents are hardcoded to specific domains
- Adding new agents requires code changes
- **Future**: Dynamic agent registration

## Troubleshooting

### "Missing API keys"

**Solution**: Ensure `.env` file exists with valid `OPENAI_API_KEY`

### "Vector store not found"

**Solution**: Run Section 2 cells to create vector stores

### "ModuleNotFoundError"

**Solution**: Install all dependencies: `pip install -r requirements.txt`

### "Qdrant client connection error"

**Solution**: The system uses local file-based Qdrant (no server needed). Ensure `data/qdrant_db/` folder exists.

### High latency

**Solution**:

- Reduce `k` value in retriever (fewer chunks)
- Use smaller embedding model (requires recreation of vector database)
- Disable evaluation node

### Poor evaluation scores

**Solution**:

- Update knowledge base documents with more detailed information
- Increase `k` value or reduce threshold to retrieve more context
- Adjust system prompts for better agent responses

## Langfuse Integration

The system logs all interactions to Langfuse for observability:

- Intent classification traces
- Query decomposition traces
- Agent response traces
- Consolidation traces
- Evaluation scores with metadata

**View traces**: Log into Langfuse dashboard at https://cloud.langfuse.com

## Shutdown

When finished, properly shutdown resources:

```python
shutdown_resources()
```

This closes Qdrant client and flushes Langfuse data.

## Future Enhancements

- [ ] Persistent conversation memory
- [ ] Dynamic agent registration
- [ ] Web interface (Streamlit)
- [ ] Incremental vector store updates
- [ ] Custom evaluation metrics
- [ ] Continuous improvement via Langfuse evaluation (golden datasets, a/b testing)
