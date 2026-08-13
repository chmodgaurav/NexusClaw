# HR AI Agent

An offline, retrieval-augmented HR assistant with conversational memory, built with Streamlit, LangChain, Ollama, and ChromaDB. Upload HR/legal documents (PDF or JSON), index them into a local vector store, ask grounded questions, and complete a multi-turn leave application entirely through chat.

## Features

### Core RAG
- Upload and index **PDF** and **JSON** documents into ChromaDB
- Semantic search using **nomic-embed-text** embeddings
- Question answering using **Gemma 3 (4B)** via Ollama, grounded in retrieved context
- Retrieved source chunks are shown alongside every answer
- Fully local — no external API calls once Ollama and the models are installed

### Conversational memory
- Per-employee session, keyed by an Employee ID entered in the sidebar
- Multi-turn conversations that don't require repeating earlier information
- Session isolation — no data leakage between employees
- Context-aware follow-ups (the assistant resolves pronouns like "them" against prior turns)
- Multi-turn leave applications that collect type, dates, and reason across several messages, then submit automatically
- Sessions expire after 2 hours of inactivity (in-memory TTL, cleared on app restart)

## Tech Stack

- **Frontend:** Streamlit
- **LLM framework:** LangChain (`langchain-community`, `langchain-chroma`, `langchain-ollama`)
- **Vector database:** ChromaDB (persisted at `./database/chroma`)
- **LLM runtime:** Ollama
- **Chat model:** Gemma 3 (4B) — `gemma3:4b`, bound to the `apply_leave` tool for tool-calling
- **Embeddings:** `nomic-embed-text`
- **Leave storage:** SQLite (`./database/hr.db`)
- **Memory system:** in-memory session management with TTL-based cleanup

## Installation & Setup

### 1. Clone the repository

```bash
git clone https://github.com/chmodgaurav/AI-HR-Agent.git
cd AI-HR-Agent
```

### 2. Install Python dependencies

```bash
pip install -r requirements.txt
```

### 3. Install Ollama

Download and install from [ollama.com](https://ollama.com).

### 4. Pull the required models

```bash
ollama pull nomic-embed-text
ollama pull gemma3:4b
```

### 5. Start Ollama

```bash
ollama serve
```

### 6. Run the application

```bash
streamlit run app.py
```

Open `http://localhost:8501`.

## Usage

### Document upload & indexing

1. Upload a PDF or JSON file via the upload UI.
2. The document is loaded (`PyPDFLoader` for PDF; parsed JSON for JSON) into LangChain `Document` objects.
3. Documents are embedded with `nomic-embed-text` and stored in the ChromaDB collection at `./database/chroma`.

### Multi-turn conversation

1. Enter an Employee ID in the sidebar — this creates or resumes a session.
2. Use the leave-application flow to apply for leave conversationally; the assistant asks for missing details (type, dates, reason) one at a time and submits the request to `./database/hr.db` once complete.
3. Ask free-form HR questions elsewhere in the app; the assistant retrieves relevant chunks from ChromaDB and answers using only that context, remembering prior turns in the same session.

### Example: multi-turn leave application

```
You: I want to apply for leave
AI: What type of leave do you need?

You: Casual leave
AI: When do you need the leave?

You: 2024-02-15 to 2024-02-17
AI: Please provide a reason

You: Medical appointment
AI: Your leave request has been submitted successfully!
```

### Example: context-aware Q&A

```
You: What is the policy on medical leave?
AI: [retrieves from knowledge base] "Medical leave policy is..."

You: How many days do I get?
AI: [remembers you asked about medical leave] "You are entitled to..."

You: Can I use them for family emergencies?
AI: [resolves "them" to medical leave] "Yes, medical leave can be used for..."
```

## Project Structure

```text
AI-HR-Agent/
│
├── app.py                          # Main Streamlit application
├── chroma.py                       # Standalone ChromaDB ingestion utility
├── conversation_memory.py          # Session and memory management (TTL-based)
├── memory_chat_engine.py           # LLM integration with memory (MemoryChatEngine, MultiTurnLeaveHandler)
├── tools/
│   └── apply_leave.py              # SQLite-backed leave application tool
├── database/
│   ├── chroma/                     # Persisted vector store
│   └── hr.db                       # SQLite leave-requests database
├── dataset/
│   ├── pdf/                        # Sample PDF documents
│   └── json/                       # Sample JSON documents
├── examples.py                     # Standalone usage examples for the memory system
├── test_conversation_memory.py     # Unit tests for conversation memory
├── test_quick.py                   # Quick smoke test
└── requirements.txt
```

## Testing

```bash
# Quick smoke test
python test_quick.py

# Full unit test suite
pytest test_conversation_memory.py -v

# Standalone usage examples
python examples.py
```

## Session Management

- Each employee gets a session identified by their Employee ID, created on first use in the sidebar.
- Sessions hold conversation history for up to 2 hours of inactivity, then expire automatically.
- Conversations are isolated per employee — no cross-session data leakage.
- All sessions live in memory and are lost when the Streamlit process restarts; there is no persistent session store yet.

## Notes

- Ollama must be running (`ollama serve`) before starting the app — embedding and chat calls fail otherwise.
- The assistant answers only from indexed documents; if the knowledge base doesn't cover a question, it says so rather than guessing.
- `chroma.py` provides a way to (re)index documents outside the Streamlit UI, in addition to the in-app uploader.

## Future Improvements

- Persistent session storage (e.g. Redis or PostgreSQL) instead of in-memory TTL sessions
- Conversation summarization for long chat histories
- Hybrid search (BM25 + vector) and metadata filtering
- Support for DOCX and TXT document uploads
- Document management (delete/re-index) from the UI

## Contributing

Issues and pull requests are welcome.

## License

This project is intended for educational and research purposes. Modify and use it according to your project's licensing requirements.
