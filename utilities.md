# DAP Utilities — Reference

Thin wrappers around Haystack components for common pre/post-processing tasks in DAP workflows. Drop them into any workflow phase or call them directly from tool handlers.

---

## Reranking

After a vector search returns top-N candidates, a cross-encoder reranker scores each (query, document) pair more accurately than cosine similarity alone.

```python
from haystack.components.rankers import TransformersSimilarityRanker

class DAPReranker:
    def __init__(self, model: str = "cross-encoder/ms-marco-MiniLM-L-6-v2", top_k: int = 5):
        self.ranker = TransformersSimilarityRanker(model=model, top_k=top_k)
        self.ranker.warm_up()

    def rerank(self, query: str, documents: list[dict]) -> list[dict]:
        from haystack.dataclasses import Document
        docs = [Document(content=d["content"], meta=d.get("meta", {})) for d in documents]
        result = self.ranker.run(query=query, documents=docs)
        return [{"content": d.content, "score": d.score, "meta": d.meta}
                for d in result["documents"]]
```

**Usage in RAG phase:**

```python
chunks = surreal_hnsw_search(query_vec, limit=20)   # broad retrieval
ranked = reranker.rerank(query_text, chunks)[:5]    # precision rerank → top 5
```

**Alternatives:**

| Model | Speed | Quality | Notes |
|---|---|---|---|
| `cross-encoder/ms-marco-MiniLM-L-6-v2` | Fast | Good | Default, runs locally |
| `cross-encoder/ms-marco-electra-base` | Medium | Better | Larger, more accurate |
| `BAAI/bge-reranker-base` | Fast | Good | Multilingual-friendly |
| Cohere Rerank API | Fast | Excellent | External API, paid |

---

## PDF Ingestion

```python
from haystack.components.converters import PyPDFToDocument
from haystack.components.preprocessors import DocumentCleaner, DocumentSplitter

class DAPPDFIngestor:
    def __init__(self, chunk_size: int = 500, chunk_overlap: int = 50):
        self.converter  = PyPDFToDocument()
        self.cleaner    = DocumentCleaner(remove_empty_lines=True, remove_extra_whitespaces=True)
        self.splitter   = DocumentSplitter(
            split_by="word", split_length=chunk_size, split_overlap=chunk_overlap
        )

    def ingest(self, pdf_path: str, meta: dict = {}) -> list[dict]:
        raw   = self.converter.run(sources=[pdf_path])
        clean = self.cleaner.run(documents=raw["documents"])
        split = self.splitter.run(documents=clean["documents"])
        return [
            {
                "content": d.content,
                "meta": {**d.meta, **meta,
                         "source": pdf_path,
                         "page": d.meta.get("page_number")}
            }
            for d in split["documents"]
        ]
```

**Then embed + store in SurrealDB:**

```python
chunks = ingestor.ingest("report.pdf", meta={"doc_type": "research", "team": "quant_desk"})
for chunk in chunks:
    vec = embed(chunk["content"])
    await db.create("document_chunk", {**chunk, "embedding": vec})
```

---

## Metadata Extraction

Extract structured metadata from documents — useful before storing in SurrealDB or building a knowledge graph.

```python
from haystack.components.extractors import NamedEntityExtractor

class DAPMetadataExtractor:
    """Extracts entities (ORG, DATE, MONEY, PERSON) from text."""

    def __init__(self):
        self.extractor = NamedEntityExtractor(
            backend="hugging_face",
            model="dslim/bert-base-NER"
        )
        self.extractor.warm_up()

    def extract(self, text: str) -> dict:
        from haystack.dataclasses import Document
        result = self.extractor.run(documents=[Document(content=text)])
        entities = {}
        for annotation in result["documents"][0].meta.get("named_entities", []):
            label = annotation["label"]
            entities.setdefault(label, []).append(annotation["word"])
        return entities
```

**Output example:**

```python
extract("Acme Corp reported $4.2M revenue in Q3 2024.")
# → {"ORG": ["Acme Corp"], "MONEY": ["$4.2M"], "DATE": ["Q3 2024"]}
```

---

## Document Converters

| Format | Haystack Component | Notes |
|---|---|---|
| PDF | `PyPDFToDocument` | Text extraction per page |
| HTML | `HTMLToDocument` | Strips tags, keeps text |
| Markdown | `MarkdownToDocument` | Preserves structure |
| CSV | `CSVToDocument` | Row-per-document |
| DOCX | `DOCXToDocument` | Word documents |
| Plain text | `TextFileToDocument` | UTF-8 |

```python
from haystack.components.converters import HTMLToDocument, MarkdownToDocument

html_converter = HTMLToDocument()
md_converter   = MarkdownToDocument()

html_docs = html_converter.run(sources=["page.html"])["documents"]
md_docs   = md_converter.run(sources=["README.md"])["documents"]
```

---

## Text Splitting Strategies

```python
from haystack.components.preprocessors import DocumentSplitter

# By word count (default for dense prose)
word_splitter = DocumentSplitter(split_by="word", split_length=300, split_overlap=30)

# By sentence (better for Q&A)
sent_splitter = DocumentSplitter(split_by="sentence", split_length=5, split_overlap=1)

# By passage (markdown sections)
pass_splitter = DocumentSplitter(split_by="passage", split_length=2, split_overlap=0)
```

| Strategy | Best for |
|---|---|
| `word` | Dense prose, reports, PDFs |
| `sentence` | Q&A retrieval, factual docs |
| `passage` | Structured docs (markdown, wikis) |

---

## Token Counter

Fast token counting before sending to LLM — stays within the workflow token budget.

```python
import tiktoken

_enc = tiktoken.get_encoding("cl100k_base")

def count_tokens(text: str) -> int:
    return len(_enc.encode(text))

def trim_to_budget(chunks: list[str], budget: int) -> list[str]:
    kept, total = [], 0
    for chunk in chunks:
        n = count_tokens(chunk)
        if total + n > budget:
            break
        kept.append(chunk)
        total += n
    return kept
```

Used in the RAG phase to enforce `token_budget` from the workflow YAML:

```yaml
phases:
  - type: rag
    token_budget: 1200   # trim_to_budget enforced here
    collections: [web_content, document_chunk]
```

---

*See also: [rag.md](rag.md) · [workflows.md](workflows.md) · [observability.md](observability.md)*
