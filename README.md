# Agentic Customer Support Task

A multi-component AI-powered customer support system using Gemini, RAG (Retrieval-Augmented Generation), and intelligent query routing to automatically triage and respond to customer inquiries.

---

## System Overview

This system intelligently routes customer queries to the appropriate handler based on the query type, then retrieves or looks up relevant information to provide accurate, sourced responses with a consistent decision-making framework.

---

## Implementation Steps

### Step 1: Query Routing – Implement All Five Route Types

**Task:** Route incoming queries to the correct handler based on content classification.

**Route Types:**
- `KNOWLEDGE_BASE`: Documentation or policy questions
- `TICKET_LOOKUP`: Ticket ID or ticket status questions (pattern: T-XXXX)
- `ACCOUNT_LOOKUP`: Customer plan or renewal questions
- `AMBIGUOUS`: Unclear requests requiring clarification
- `UNSUPPORTED`: Requests for data not present in the knowledge base

**Implementation:**
- Uses Gemini to classify queries via prompt-based routing
- Returns only the category name to ensure deterministic behavior
- Routes are checked in order: triage → knowledge base → ticket → account → ambiguous → unsupported

**Notes:**
- Routing is deterministic and rule-based to avoid hallucination
- Each route type triggers a specific handler function
- Special case: queries containing "handle first" trigger triage logic

---

### Step 2: Knowledge Retrieval – Basic RAG Pipeline; Cite Sources

**Task:** Implement Retrieval-Augmented Generation to answer documentation questions accurately.

**Implementation:**
- **Embeddings:** SentenceTransformer (`all-MiniLM-L6-v2`) generates vector embeddings for all knowledge base documents
- **Vector Database:** FAISS IndexFlatL2 stores embeddings for fast semantic similarity search
- **Retrieval:** Query embeddings are searched against the index to retrieve top-k (default: 2) most relevant documents
- **Generation:** Retrieved documents are passed as context to Gemini to generate an informed answer

**Knowledge Base Sources:**
- `data/refund_policy.md`
- `data/account_upgrade.md`
- `data/api_rate_limits.md`
- `data/security_practices.md`
- `data/integration_setup.md`

**Notes:**
- Only ranked documents are included in the context; prevents hallucination
- Source files are tracked and returned with each response
- K=2 provides sufficient context without overwhelming the model

---

### Step 3: Ticket Lookup – Structured Output

**Task:** Retrieve ticket information by ticket ID with consistent structured formatting.

**Implementation:**
- Pattern matching extracts ticket IDs from queries (regex: `T-\d+`)
- Linear search through tickets JSON to find matching ticket
- Returns structured output with ticket metadata:
  - Ticket ID
  - Status
  - Assigned Agent (with fallback handling for field naming variations)
  - Priority
  - Customer Name

**Notes:**
- Handles missing fields gracefully using `.get()` with fallback values
- Returns "Ticket not found" rather than querying external systems
- Assigned agent field uses fallback logic: `agent` → `assigned_agent` → "Not assigned"

---

### Step 4: Triage Logic – Weighted Ranking with Explanation

**Task:** Prioritize tickets based on priority level and account health score.

**Implementation:**
- **Priority Scoring:**
  - High priority: +3 points
  - Medium priority: +2 points
  - Low priority: +1 point

- **Health Score Adjustment:**
  - Health score < 50: +2 points (at-risk accounts get higher priority)
  - Health score 50-80: +1 point (moderate risk)
  - Health score ≥ 80: no adjustment (healthy accounts)

- **Output:** Sorted list of tickets with computed scores, customer name, and summary; lowest scores first

**Notes:**
- Combines multiple signal types (priority + account health) for intelligent ranking
- Explanation is embedded in the scoring logic for transparency
- Helps support teams handle the most critical cases first

---

### Step 5: Account Lookup – JSON Lookup with Filtering

**Task:** Retrieve customer account information by name matching.

**Implementation:**
- Case-insensitive name matching against accounts JSON
- Returns structured account information:
  - Customer Name
  - Plan type
  - Renewal Date
  - Health Score
  - Open Ticket Count

- Filtering: Searches for customer name in query string using substring matching

**Notes:**
- Simple but effective name-based lookup avoids external API calls
- Health score provides context for issue severity
- Open ticket count surfaces customer support load

---

### Step 6: Consistent Decision Object Per Response

**Task:** Return uniform response structure across all query types for consistent client-side handling.

**Implementation:**
- Every assistant response returns a tuple: `(answer, decision)`
- Decision object contains:
  ```json
  {
    "route": "STRING",                 // Routing category used
    "confidence": "FLOAT",              // Confidence score (0.0-1.0)
    "used_sources": ["STRING"],         // Files/data accessed (KB or JSON)
    "used_tools": ["STRING"],           // Tools invoked (rag_retriever, ticket_lookup, etc.)
    "needs_clarification": "BOOLEAN",   // True if follow-up needed
    "final_answer": "STRING"            // The response text or structured data
  }
  ```

**Notes:**
- Enables logging, auditing, and debugging of system decisions
- Confidence flag allows client to request clarification or escalation
- Used tools and sources provide transparency and traceability

---

### Step 7: Ambiguity Handling – Ask Clarification; Do Not Guess

**Task:** Detect unclear queries and request clarification rather than assume intent.

**Implementation:**
- Ambiguous queries (classified by Gemini as `AMBIGUOUS`) return a clarification request
- Clarification request: `"Could you clarify your request?"`
- `needs_clarification` flag set to `True` in decision object
- No attempt to retrieve or generate answers from incomplete information

**Notes:**
- Prevents incorrect or misleading responses
- Guides users toward providing more specific queries
- Better for user experience and support accuracy

---

### Step 8: Unsupported Refusal – No Hallucination Under Any Input

**Task:** Refuse requests for information outside the system's data without inventing answers.

**Implementation:**
- Queries that don't match any supported route type are classified as `UNSUPPORTED`
- Response: `"Sorry, that information is not available."`
- `needs_clarification` flag set to `False`
- No model-generated content for out-of-scope queries

**Example Unsupported Queries:**
- Requests for data not in knowledge base
- Requests for external APIs or third-party data
- Requests outside the support domain

**Notes:**
- Prevents model hallucination and factual errors
- Clear user feedback about system limitations
- Maintains system reliability and trustworthiness

---

## Usage

### Interactive Mode
```python
query = input("Ask a question (or type 'exit' to quit): ")
answer, decision = assistant(query)
print(answer)
print(json.dumps(decision, indent=2))
```

### Example Queries

| Query | Route | Output |
|-------|-------|--------|
| "What is your refund policy?" | KNOWLEDGE_BASE | RAG-retrieved policy with sources |
| "What's the status of T-12345?" | TICKET_LOOKUP | Ticket details (status, agent, priority) |
| "Tell me about Acme Corp's account" | ACCOUNT_LOOKUP | Account details (plan, health, open tickets) |
| "Which tickets should we handle first?" | TRIAGE | Sorted list of tickets by priority score |
| "How do I...?" | AMBIGUOUS | "Could you clarify your request?" |
| "Show me your stock prices" | UNSUPPORTED | "Sorry, that information is not available." |

---

## Architecture

```
User Query
    ↓
Route Query (Gemini Classification)
    ↓
┌─────────────────────────────────────┐
│  Routing Decision                   │
├─────────────────────────────────────┤
│ • TRIAGE → Triage Logic            │
│ • KNOWLEDGE_BASE → RAG Pipeline    │
│ • TICKET_LOOKUP → Ticket Search    │
│ • ACCOUNT_LOOKUP → Account Search  │
│ • AMBIGUOUS → Ask Clarification    │
│ • UNSUPPORTED → Refuse             │
└─────────────────────────────────────┘
    ↓
Retrieve/Generate Answer
    ↓
Return (Answer, Decision Object)
```

---

## Key Design Principles

1. **No Hallucination:** All information comes from provided knowledge base or JSON data
2. **Transparency:** Every response includes sources and tools used
3. **Consistency:** Uniform decision object structure across all route types
4. **Graceful Degradation:** Missing fields handled with fallback values
5. **User Clarity:** Ambiguous queries prompt clarification; out-of-scope requests are refused
6. **Intelligent Prioritization:** Triage combines multiple signals for better ticket ranking

---

## Dependencies

- `google-generativeai` – Gemini API integration
- `sentence-transformers` – SentenceTransformer embeddings
- `faiss-cpu` – Vector database
- `python-dotenv` – Environment variable management
- `numpy` – Numerical operations

---

## Data Files

- `data/tickets.json` – Ticket records
- `data/accounts.json` – Customer account records
- `data/refund_policy.md` – Refund policy documentation
- `data/account_upgrade.md` – Account upgrade documentation
- `data/api_rate_limits.md` – API rate limits documentation
- `data/security_practices.md` – Security practices documentation
- `data/integration_setup.md` – Integration setup documentation

---

## Configuration

Set the following environment variable in `.env`:
```
GEMINI_API_KEY=your_api_key_here
```

---

## Future Improvements (Optional)

- Multi-turn conversation memory
- Fine-tuned routing classifier
- Caching of embeddings and FAISS index
- Webhook integration for ticket updates
- Analytics and logging dashboard
- More sophisticated NLP for entity extraction
