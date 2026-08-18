# Telecom RAG Customer Support Assistant

A Retrieval-Augmented Generation (RAG) system that answers Egyptian telecom customer support tickets (written in Egyptian Arabic) using a company's internal technical support knowledge base — built as an improvement on a baseline RAG demo, with structure-aware chunking, smarter retrieval, and a hardened system prompt.

## Overview

Customer service agents at a telecom provider need instant, accurate answers to customer complaints (billing issues, router troubleshooting, outage escalation, SLAs) without manually searching a large internal knowledge base. This project builds a RAG pipeline that:

1. Ingests the internal KB (router specs, troubleshooting guides, billing rules, escalation matrix)
2. Retrieves the most relevant sections for a given customer complaint
3. Generates a natural, policy-compliant Egyptian Arabic response using Gemini

## Key Improvements Over the Baseline

The original baseline chunked the knowledge base purely by character count (`RecursiveCharacterTextSplitter`, `chunk_size=500`, `chunk_overlap=100`), ignoring the document's actual structure. This project improves on that baseline in five ways:

### 1. Structure-Aware Markdown Chunking
The knowledge base is written in Markdown with a clear header hierarchy (`#`, `##`, `###`, `####`) — e.g., each router model's specs live under their own header, and individual error codes live under their own `####` sub-header. Instead of splitting blindly by character count, this project:
- Splits first by Markdown headers (`MarkdownHeaderTextSplitter`), keeping each section (e.g., a single router model's full spec, or a single error code's description + resolution) intact and tagged with its header path as metadata
- Falls back to `RecursiveCharacterTextSplitter` only for sections still too large

This prevents a router's model name from being separated from its own specifications — a failure mode the character-based baseline was prone to.

### 2. Metadata-Aware Retrieval with MMR
- Retrieval switched from plain top-k similarity search to **MMR (Maximal Marginal Relevance)**, fetching a wider candidate pool (`fetch_k=20`) and returning a smaller, diverse, non-redundant set (`k=6`) — since the new chunks are denser and more complete, fewer of them are needed
- Each retrieved chunk is tagged with its source section (e.g., `[Section: Router Model: VDF-ZTE-2023X1]`), using the most specific header available (`Header 4` → `Header 3` → `Header 2`) before being passed to the LLM, reducing confusion between similar router models
- Chunks are embedded and ingested into FAISS in batches (`batch_size=50`, with a `tqdm` progress bar) instead of all at once, to keep embedding/indexing resource usage manageable on larger knowledge bases

### 3. Hardened System Prompt
The prompt enforces explicit negative constraints so the agent persona stays safe and consistent:
- Never reveals a real telecom company name
- Never discloses it's an AI/bot
- Never fabricates a phone number, email, or link not present in the retrieved context
- Escalates to a specialist team instead of inventing an answer when information is missing
- Avoids overly formal Arabic phrasing, keeps responses to ~4–5 sentences, and avoids repetitive openings
- Presents multi-step resolutions as a numbered list, and always opens with the error code if the customer mentioned one

### 4. Deterministic Exact-Match Layer for Error Codes
Semantic retrieval alone can't guarantee it surfaces the exact right error code when a customer's ticket literally states one (e.g., `E-204`). To close that gap:
- The knowledge base's `#### Error Code E-XXX` entries are parsed once (via regex) into a lookup dictionary mapping each code to its description and resolution protocol
- If the incoming ticket mentions an error code by name, that code's info is pulled directly from the dictionary — guaranteed correct — and injected into the context alongside whatever the semantic retriever returns
- This runs as an addition to, not a replacement for, the MMR-based semantic search

### 5. Always-Injected SLA Policy Context
The "General SLA & Dispatch Policies" section is short and relevant to nearly every outage-related complaint, so instead of leaving it to chance whether semantic retrieval surfaces it:
- It's extracted once by section heading and injected into the context on every single request, in full, regardless of what the retriever returns

## Tech Stack

| Component | Choice |
|---|---|
| Orchestration | LangChain |
| Embeddings | `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2` (multilingual — matches Arabic queries to English/Arabic-mixed docs) |
| Vector Store | FAISS (local, persisted to disk) |
| LLM | Google Gemini (`gemini-3.6-flash`) via `langchain-google-genai` |
| Chunking | `MarkdownHeaderTextSplitter` (`#`–`####`) + `RecursiveCharacterTextSplitter` (fallback) |
| Retrieval | MMR (`k=6`, `fetch_k=20`) |
| Exact-Match Layer | Regex-based parser + lookup dict for guaranteed-correct `E-XXX` error code resolutions |
| Always-On Context | Full "General SLA & Dispatch Policies" section injected on every request |

## Project Structure

```
telecom-rag-customer-support/
├── data/
│   ├── Telecom_Internal_KB.pdf     # Source knowledge base (original)
│   └── Telecom_Internal_KB.txt     # Source knowledge base (Markdown-structured text)
├── rag_demo.ipynb                  # Full pipeline: ingestion → chunking → retrieval → generation
└── README.md
```

## Setup

1. Clone the repo and install dependencies:
   ```bash
   pip install -U langchain langchain-community langchain-core langchain-google-genai langchain-huggingface sentence-transformers faiss-cpu python-dotenv tqdm
   ```
2. Get a free Gemini API key from [Google AI Studio](https://aistudio.google.com/apikey)
3. Create a `.env` file in the project root:
   ```
   GOOGLE_API_KEY=your_key_here
   ```
4. Open `rag_demo.ipynb` and run the cells in order

## Example

**Customer ticket (Egyptian Arabic):**
> أنا دافع الفاتورة من يومين أونلاين والفلوس اتخصمت من الفيزا، لكن النت لسه مرجعش لحد دلوقتي ومكتوبلي إن الخدمة موقوفة!

**Agent response:**
> مساء الخير يا فندم، بعتذر لحضرتك جدًا عن الإزعاج ده... بيتم تحويلها فورًا لقسم الحسابات على رقم 111...

The response stays grounded in the retrieved KB sections, uses a natural Egyptian Arabic support tone, and never invents information outside the provided context.

## Acknowledgments

Built as an extension of a RAG demo from the HumaVolve AI Engineering Bootcamp (instructor: Ziad Ashraf).
