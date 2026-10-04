# LangChain Document Assistant

An AI assistant for a company's internal documents. Ask it about a policy and it retrieves the relevant passages and answers from them; it can also summarise, translate with enforced terminology, link related documents, search the web when the documents don't cover something, and generate images on request. Everything runs on GPT-4o through LangChain with a Chroma vector store.

It came out of a one-day hackathon in a generative-AI bootcamp, built by a team of five. The brief was an "intelligent document assistant" over a set of HR and IT policy documents for a fictional company. My part was the document-linking and live-sync features, plus merging everyone's components into the single agent in `02_double_agent.ipynb` and writing the documentation.

## Notebooks

### `01_document_assistant.ipynb`: the step-by-step build

Each capability is built and tested on its own here:

- **Loading and chunking**: text, PDF, Word and HTML loaders, split into 1,000-character chunks with 200 characters of overlap
- **Vector store**: `text-embedding-3-large` embeddings persisted in Chroma, with a similarity-search test
- **RAG Q&A**: a retriever → prompt → GPT-4o chain restricted to the retrieved context
- **Summarisation** tuned for four audiences: general, executive, technical and new employees
- **Translation** with configurable source language, target language and tone
- **A tool-using agent** that combines search, Q&A, summarisation and translation
- **Automatic document linking**, which finds related documents and has the LLM explain the connection
- **Faceted search** that filters by document type, department and policy type (written but not run in the saved notebook)

### `02_double_agent.ipynb`: the combined agent

This is the final version: one OpenAI tools agent with conversation memory and seven tools.

| Tool | What it does |
|---|---|
| `search` | Top-2 similarity search over the document store |
| `qa` | RAG answer, followed by links to the source documents |
| `summarize` | Plain-language summary, plus related documents |
| `translate` | Multi-step translation pipeline (below) |
| `link_documents` | The four most related documents with a short preview of each |
| `generate_image` | DALL·E 3 image from a text description |
| DuckDuckGo | Web search for anything the documents don't cover |

Two details are more interesting than the tool list.

**The system prompt is generated, not hand-written.** A first LLM call acts as a prompt writer and produces the agent's system prompt from a specification: retrieve from internal documents, cite them, say clearly when the documents don't contain the answer and fall back to web search, and call the image tool whenever an image is requested instead of describing one.

**Translation is a small pipeline of its own.** A translator step produces a formal translation, a cultural-adaptation step rewrites it for the target locale (es-ES, fr-FR or de-DE), and a glossary pass forces strict terms such as *AI*, *policy* and *compliance* into their approved translations. A quality check then scores the output and flags any glossary term that went missing.

## Running it

```bash
python -m venv .venv
# Windows:      .venv\Scripts\activate
# macOS/Linux:  source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env    # then put your OpenAI API key in .env
jupyter notebook
```

Put the documents you want to query into `documents/` as `.txt`, `.pdf`, `.docx` or `.html` files. The hackathon's original documents came with the course and aren't redistributed here, so any set of policy-style documents will do. Re-running the notebooks calls the OpenAI API, so it costs a small amount of credit, and the image tool charges per image.

The code targets LangChain 0.3 (`AgentExecutor`, `ConversationBufferMemory` and `langchain.chains`), which LangChain 1.0 removed, so `requirements.txt` pins the 0.3 line.

## Known limitations

- The translation "quality score" compares character n-grams between the English source and the translated text. Across two languages that mostly measures surface overlap, so treat it as a rough sanity check. A real check would back-translate, or compare against a reference translation.
- In the saved run, one Q&A test (the vacation-policy question) failed with a Pydantic validation error, a `Document` with empty `page_content`, during the agent's search step. The other tests passed.
- The real-time sync cell (watching `documents/` with `watchdog` and re-indexing changed files) runs until you interrupt it.
