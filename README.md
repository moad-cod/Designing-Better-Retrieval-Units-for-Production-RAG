# Better Retrieval Units for Production RAG

A practical companion repository for the article:

> **Beyond Fixed Chunks: Designing Better Retrieval Units for Production RAG**

This project explores how different document representations affect retrieval quality in Retrieval-Augmented Generation systems.

Instead of treating chunking as a simple preprocessing step, the repository compares multiple retrieval-unit strategies, including fixed-size chunks, paragraphs, sentences, semantic chunks, propositions, late chunking, and hierarchical retrieval.

The central idea is:

> A retriever can only return the knowledge units created during ingestion.

The repository provides a small experimental environment for creating retrieval units, embedding them, indexing them in Qdrant, storing their metadata and relationships in PostgreSQL, and evaluating retrieval performance across different strategies.

---

## Why This Project Exists

Many RAG systems follow the same basic pipeline:

```text
Document
   ↓
Split into chunks
   ↓
Generate embeddings
   ↓
Store vectors
   ↓
Retrieve top-k chunks
   ↓
Send context to an LLM
```

This approach works for demonstrations, but production systems expose several limitations:

* Fixed boundaries may split related information.
* Large chunks may contain multiple unrelated facts.
* Small chunks may lose references and context.
* Tables, code blocks, headings, and document structure may be damaged.
* Sentence-level retrieval may be precise but incomplete.
* Proposition-level retrieval may improve precision but increase ingestion cost.
* Broad questions may require sections or document summaries instead of isolated facts.

This repository makes these trade-offs measurable.

---

## Main Goals

The project is designed to:

1. Represent the same document using different retrieval-unit strategies.
2. Store document structure and parent-child relationships.
3. Index retrieval-unit embeddings in Qdrant.
4. Store source metadata and experiment results in PostgreSQL.
5. Run the same queries against every strategy.
6. Compare retrieval quality, latency, storage size, and context efficiency.
7. Demonstrate the pattern:

```text
Retrieve small, return enough context.
```

---

## Supported Retrieval Strategies

### Fixed-size chunking

Splits text using a predefined number of words or tokens with optional overlap.

```text
Document → Fixed windows
```

Fixed-size chunking is simple, predictable, and useful as a baseline.

---

### Paragraph retrieval

Uses document paragraphs as retrieval units.

```text
Document → Paragraphs
```

Paragraphs usually preserve more natural semantic boundaries than fixed windows.

---

### Sentence retrieval

Indexes individual sentences for precise factual retrieval.

```text
Document → Paragraph → Sentence
```

Each sentence remains connected to its parent paragraph so the system can search with the sentence and return the surrounding paragraph.

---

### Semantic chunking

Creates boundaries when the topic or semantic meaning changes.

```text
Sentences → Similarity comparison → Semantic groups
```

Semantic chunks may preserve topic continuity better than fixed-size windows.

---

### Proposition retrieval

Transforms passages into atomic, self-contained facts.

```text
Paragraph → Sentences → Atomic propositions
```

Proposition retrieval is useful for narrow factual questions where one paragraph contains several independently searchable facts.

---

### Late chunking

Processes a larger document context through an embedding model before pooling token representations into smaller chunk embeddings.

```text
Long document encoding → Contextualized tokens → Chunk pooling
```

This helps small units retain information from their surrounding context.

---

### Hierarchical retrieval

Represents documents at multiple levels.

```text
Document
├── Section
│   ├── Paragraph
│   │   ├── Sentence
│   │   └── Proposition
│   └── Section summary
└── Document summary
```

Hierarchical retrieval supports both detailed factual queries and broad thematic questions.

---

## Architecture

```text
                         ┌─────────────────────┐
                         │   Source Document   │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │  Document Loader    │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │ Retrieval Strategy  │
                         │                     │
                         │ Fixed               │
                         │ Paragraph           │
                         │ Sentence            │
                         │ Semantic            │
                         │ Proposition         │
                         │ Late Chunking       │
                         │ Hierarchical        │
                         └──────────┬──────────┘
                                    │
                         ┌──────────┴──────────┐
                         │                     │
                         ▼                     ▼
              ┌──────────────────┐  ┌──────────────────┐
              │    PostgreSQL    │  │     Embedding    │
              │                  │  │      Model       │
              │ Documents        │  └────────┬─────────┘
              │ Retrieval units  │           │
              │ Parent links     │           ▼
              │ Experiments      │  ┌──────────────────┐
              │ Results          │  │      Qdrant      │
              └────────┬─────────┘  │                  │
                       │            │ Vector index     │
                       │            │ Search payloads  │
                       │            └────────┬─────────┘
                       │                     │
                       └──────────┬──────────┘
                                  │
                                  ▼
                       ┌─────────────────────┐
                       │     Retriever       │
                       └──────────┬──────────┘
                                  │
                                  ▼
                       ┌─────────────────────┐
                       │ Context Expansion   │
                       │                     │
                       │ Sentence → Paragraph│
                       │ Proposition → Source│
                       │ Paragraph → Section │
                       └──────────┬──────────┘
                                  │
                                  ▼
                       ┌─────────────────────┐
                       │    Evaluation       │
                       └─────────────────────┘
```

---

## Technology Stack

| Component            | Technology                     |
| -------------------- | ------------------------------ |
| Programming language | Python                         |
| Vector database      | Qdrant                         |
| Metadata database    | PostgreSQL                     |
| Embedding models     | Sentence Transformers          |
| Database access      | SQLAlchemy and Psycopg         |
| Infrastructure       | Docker Compose                 |
| Evaluation           | NumPy, Pandas and scikit-learn |
| Testing              | Pytest                         |

---

## Why Qdrant and PostgreSQL?

### Qdrant

Qdrant is responsible for:

* Storing retrieval-unit embeddings.
* Performing vector similarity search.
* Filtering by strategy, document, unit type, or metadata.
* Returning the most relevant retrieval-unit IDs.

A Qdrant point may contain:

```json
{
  "id": "retrieval-unit-uuid",
  "vector": [0.021, -0.074, 0.118],
  "payload": {
    "document_id": "document-uuid",
    "parent_id": "paragraph-uuid",
    "strategy": "sentence",
    "unit_type": "sentence",
    "position": 4
  }
}
```

### PostgreSQL

PostgreSQL is the source of truth for:

* Documents.
* Retrieval-unit content.
* Parent-child relationships.
* Chunking strategies.
* Source metadata.
* Evaluation questions.
* Relevance labels.
* Experiment runs.
* Retrieval results.

Qdrant finds the relevant unit. PostgreSQL restores its complete content and original context.

---

## Project Structure

```text
better-retrieval-units-rag/
│
├── README.md
├── requirements.txt
├── docker-compose.yml
├── .env.example
├── .gitignore
│
├── docker/
│   └── postgres/
│       └── init.sql
│
├── data/
│   ├── documents/
│   │   └── sample.txt
│   └── evaluation/
│       ├── questions.jsonl
│       └── relevance.jsonl
│
├── src/
│   └── retrieval_units/
│       ├── __init__.py
│       ├── models.py
│       ├── database.py
│       ├── embeddings.py
│       ├── vector_store.py
│       ├── retriever.py
│       ├── context_expander.py
│       │
│       ├── strategies/
│       │   ├── __init__.py
│       │   ├── base.py
│       │   ├── fixed.py
│       │   ├── paragraph.py
│       │   ├── sentence.py
│       │   ├── semantic.py
│       │   ├── proposition.py
│       │   ├── late_chunking.py
│       │   └── hierarchical.py
│       │
│       └── evaluation/
│           ├── metrics.py
│           └── runner.py
│
├── scripts/
│   ├── ingest.py
│   ├── search.py
│   ├── compare.py
│   └── reset.py
│
├── results/
│   └── .gitkeep
│
└── tests/
    ├── test_strategies.py
    └── test_retrieval.py
```

---

## Getting Started

### Prerequisites

Install:

* Python 3.11 or newer.
* Docker.
* Docker Compose.
* Git.

Verify the installation:

```bash
python --version
docker --version
docker compose version
```

---

## Installation

Clone the repository:

```bash
git clone https://github.com/YOUR_USERNAME/better-retrieval-units-rag.git
cd better-retrieval-units-rag
```

Create a virtual environment:

```bash
python -m venv .venv
```

Activate it on Linux or macOS:

```bash
source .venv/bin/activate
```

Activate it on Windows:

```powershell
.venv\Scripts\activate
```

Install the Python dependencies:

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

---

## Environment Configuration

Copy the environment example:

```bash
cp .env.example .env
```

Windows PowerShell:

```powershell
Copy-Item .env.example .env
```

Example configuration:

```env
# PostgreSQL
POSTGRES_USER=retrieval_units
POSTGRES_PASSWORD=retrieval_units_dev
POSTGRES_DB=retrieval_units
POSTGRES_HOST=localhost
POSTGRES_PORT=5432

# Qdrant
QDRANT_HOST=localhost
QDRANT_HTTP_PORT=6333
QDRANT_GRPC_PORT=6334
QDRANT_COLLECTION=retrieval_units

# Embeddings
EMBEDDING_MODEL=sentence-transformers/all-MiniLM-L6-v2
EMBEDDING_DIMENSION=384
```

Do not commit the local `.env` file.

---

## Start the Infrastructure

Start PostgreSQL and Qdrant:

```bash
docker compose up -d
```

Check their status:

```bash
docker compose ps
```

View service logs:

```bash
docker compose logs -f
```

Qdrant should be available at:

```text
http://localhost:6333
```

Stop the services:

```bash
docker compose down
```

Delete the containers and stored development data:

```bash
docker compose down -v
```

The `-v` option permanently removes the PostgreSQL and Qdrant development volumes.

---

## Basic Workflow

The project follows this workflow:

```text
Document
   ↓
Create retrieval units
   ↓
Store metadata in PostgreSQL
   ↓
Generate embeddings
   ↓
Store vectors in Qdrant
   ↓
Embed the query
   ↓
Retrieve top-k units
   ↓
Expand to parent context
   ↓
Evaluate results
```

---

## Ingest a Document

Place a text document in:

```text
data/documents/
```

Run ingestion with fixed-size chunking:

```bash
python scripts/ingest.py \
  --input data/documents/sample.txt \
  --strategy fixed
```

Ingest it as paragraphs:

```bash
python scripts/ingest.py \
  --input data/documents/sample.txt \
  --strategy paragraph
```

Ingest it as sentences:

```bash
python scripts/ingest.py \
  --input data/documents/sample.txt \
  --strategy sentence
```

Example output:

```text
Document loaded: data/documents/sample.txt
Strategy: sentence
Retrieval units created: 143
Metadata stored in PostgreSQL
Embeddings created: 143
Vectors stored in Qdrant
Ingestion completed successfully
```

---

## Search the Index

Search fixed-size chunks:

```bash
python scripts/search.py \
  --strategy fixed \
  --query "What is late chunking?" \
  --top-k 5
```

Search sentence units and return their parent paragraphs:

```bash
python scripts/search.py \
  --strategy sentence \
  --query "Why can sentence retrieval lose context?" \
  --top-k 5 \
  --expand-parent
```

Example output:

```text
Query:
Why can sentence retrieval lose context?

Strategy:
sentence

Result 1
Score: 0.821

Matched unit:
A sentence is grammatically complete, but it is not always semantically self-contained.

Returned context:
A sentence is grammatically complete, but it is not always semantically
self-contained. Pronouns, dates, definitions, and entity references may depend
on surrounding sentences in the original paragraph.
```

---

## Compare Strategies

Run the same evaluation questions against every available strategy:

```bash
python scripts/compare.py \
  --strategies fixed paragraph sentence \
  --top-k 5
```

Example comparison:

| Strategy  | Hit Rate@5 |  MRR | Indexed Units | Average Unit Tokens | Average Latency |
| --------- | ---------: | ---: | ------------: | ------------------: | --------------: |
| Fixed     |       0.76 | 0.62 |           120 |                 480 |           42 ms |
| Paragraph |       0.80 | 0.68 |            94 |                 360 |           39 ms |
| Sentence  |       0.84 | 0.72 |           310 |                  31 |           45 ms |

Results are written to:

```text
results/
```

---

## Evaluation Dataset

Questions are stored in:

```text
data/evaluation/questions.jsonl
```

Example:

```json
{"id": "q1", "query": "What is late chunking?"}
{"id": "q2", "query": "Why can sentence retrieval lose context?"}
{"id": "q3", "query": "What is proposition retrieval?"}
```

Relevance labels are stored in:

```text
data/evaluation/relevance.jsonl
```

Example:

```json
{
  "question_id": "q1",
  "relevant_document": "sample.txt",
  "relevant_text": "Late chunking processes a longer section through an embedding model before applying chunk-level pooling."
}
```

---

## Evaluation Metrics

The initial implementation focuses on retrieval metrics.

### Hit Rate@K

Measures whether at least one relevant unit appears in the top-k results.

```text
Hit Rate@K = successful queries / total queries
```

### Recall@K

Measures how many relevant units were found in the top-k results.

```text
Recall@K = retrieved relevant units / all relevant units
```

### Mean Reciprocal Rank

Measures how highly the first relevant unit is ranked.

```text
MRR = average(1 / rank of first relevant result)
```

### Context efficiency

Measures how much retrieved content is useful compared with the total number of retrieved tokens.

### Retrieval latency

Measures the time required to embed a query and retrieve the top-k results.

### Index size

Tracks how many searchable units each strategy creates.

This is important because proposition and sentence retrieval may improve precision while significantly increasing the number of indexed units.

---

## Parent-Child Retrieval

One of the main patterns demonstrated by the repository is:

```text
Retrieve the sentence, return the paragraph.
```

For example:

```text
Paragraph
├── Sentence 1
├── Sentence 2
└── Sentence 3
```

Each sentence receives its own vector and can be retrieved independently.

However, the retrieved sentence stores a reference to its parent paragraph.

```python
context = result.parent_text or result.text
```

The sentence provides precise matching. The paragraph restores the context needed for generation.

The same pattern can be applied to:

```text
Proposition → Original sentence
Sentence → Paragraph
Paragraph → Section
Section → Document summary
```

---

## Adding a New Strategy

Every retrieval strategy implements the same interface.

```python
from abc import ABC, abstractmethod

from retrieval_units.models import Document, RetrievalUnit


class RetrievalUnitStrategy(ABC):
    name: str

    @abstractmethod
    def create_units(
        self,
        document: Document,
    ) -> list[RetrievalUnit]:
        """Transform a document into searchable retrieval units."""
        raise NotImplementedError
```

Example implementation:

```python
class CustomStrategy(RetrievalUnitStrategy):
    name = "custom"

    def create_units(
        self,
        document: Document,
    ) -> list[RetrievalUnit]:
        units = []

        # Implement retrieval-unit creation here.

        return units
```

Register the strategy in the ingestion script and add tests under:

```text
tests/test_strategies.py
```

All strategies should produce the same `RetrievalUnit` model so they can use the same embedding, storage, retrieval, and evaluation pipeline.

---

## Retrieval Unit Model

A retrieval unit contains more than text.

```python
from dataclasses import dataclass, field
from typing import Any


@dataclass
class RetrievalUnit:
    id: str
    document_id: str
    text: str
    strategy: str
    unit_type: str

    parent_id: str | None = None
    parent_text: str | None = None

    metadata: dict[str, Any] = field(default_factory=dict)
```

Important fields:

| Field         | Purpose                                                  |
| ------------- | -------------------------------------------------------- |
| `text`        | Content embedded and searched                            |
| `strategy`    | Strategy that created the unit                           |
| `unit_type`   | Sentence, paragraph, proposition, section, or chunk      |
| `parent_id`   | Reference to a larger source unit                        |
| `parent_text` | Context returned after retrieval                         |
| `metadata`    | Position, page, section, token count, and source details |

---

## Current Scope

The first release focuses on:

* Plain-text documents.
* Fixed-size chunking.
* Paragraph retrieval.
* Sentence retrieval.
* Sentence-to-paragraph expansion.
* Dense embeddings.
* Qdrant vector search.
* PostgreSQL metadata storage.
* Basic retrieval evaluation.

The first release intentionally excludes:

* LLM answer generation.
* FastAPI.
* Airflow.
* Redis.
* Kubernetes.
* Authentication.
* Multi-tenancy.
* Hybrid search.
* Production monitoring.

The goal is to isolate retrieval-unit design before introducing the complexity of a complete RAG platform.

---

## Roadmap

### Version 0.1

* [ ] PostgreSQL and Qdrant infrastructure
* [ ] Document ingestion
* [ ] Fixed-size chunking
* [ ] Paragraph retrieval
* [ ] Sentence retrieval
* [ ] Parent-context expansion
* [ ] Dense vector search
* [ ] Hit Rate@K
* [ ] Recall@K
* [ ] MRR

### Version 0.2

* [ ] Semantic chunking
* [ ] Strategy configuration
* [ ] Retrieval comparison reports
* [ ] Context-efficiency metrics
* [ ] Duplicate-result analysis

### Version 0.3

* [ ] Proposition extraction
* [ ] Proposition validation
* [ ] Proposition-to-source expansion
* [ ] Index-size and ingestion-cost analysis

### Version 0.4

* [ ] Late chunking
* [ ] Long-context embedding experiments
* [ ] Traditional versus late-chunking evaluation

### Version 0.5

* [ ] Hierarchical retrieval
* [ ] Section summaries
* [ ] Multi-level search
* [ ] Flat versus hierarchical experiments

### Future Work

* [ ] Table retrieval units
* [ ] Code-block retrieval
* [ ] Page-level multimodal retrieval
* [ ] Sparse retrieval
* [ ] Hybrid retrieval
* [ ] Reranking
* [ ] Answer-faithfulness evaluation
* [ ] RAG generation experiments

---

## Testing

Run all tests:

```bash
pytest
```

Run tests with verbose output:

```bash
pytest -v
```

Run only strategy tests:

```bash
pytest tests/test_strategies.py -v
```

---

## Reset the Development Environment

Delete indexed vectors and database records using the reset script:

```bash
python scripts/reset.py
```

To completely delete the local infrastructure data:

```bash
docker compose down -v
docker compose up -d
```

---

## Design Principles

This repository follows several principles.

### Use fixed-size chunking as the baseline

Advanced strategies should demonstrate measurable improvements over a simple baseline.

### Preserve the original source

Generated retrieval units should never replace the source document.

### Keep retrieval and returned context separate

The unit used for search does not always need to be the unit sent to the LLM.

### Preserve relationships

Sentences, propositions, paragraphs, sections, pages, tables, and summaries should remain connected to their original sources.

### Evaluate every strategy fairly

All strategies should use:

* The same documents.
* The same embedding model.
* The same questions.
* The same top-k value.
* The same evaluation metrics.

### Measure cost as well as quality

A strategy that slightly improves retrieval while multiplying index size and ingestion cost may not be the best production choice.

---

## Key Question

The repository is built around one design question:

> What unit of knowledge should the system retrieve for this type of content and this type of query?

The answer may be:

* A fixed window.
* A paragraph.
* A sentence.
* An atomic proposition.
* A table.
* A code block.
* A section summary.
* A document-level node.

No retrieval unit is universally optimal.

The correct choice depends on the document structure, query type, embedding model, evaluation results, latency requirements, and ingestion budget.

---

## Related Article

This repository accompanies:

**Beyond Fixed Chunks: Designing Better Retrieval Units for Production RAG**

The article explains:

* Why chunking quality controls retrieval quality.
* The precision-versus-context trade-off.
* Fixed-size chunking.
* Semantic chunking.
* Paragraph and sentence retrieval.
* Proposition retrieval.
* Late chunking.
* Hierarchical retrieval.
* Parent-child context expansion.
* Production evaluation considerations.

Add the published article URL here:

```text
ARTICLE_URL
```

---

## Contributing

Contributions are welcome.

Useful contribution areas include:

* New retrieval-unit strategies.
* Better sentence and paragraph parsing.
* Additional evaluation datasets.
* Retrieval metrics.
* Late-chunking implementations.
* Proposition extraction methods.
* Hierarchical retrieval experiments.
* Documentation improvements.

Before submitting a pull request:

```bash
pytest
```

Please keep implementations modular so every strategy can be compared using the same pipeline.

---

## License

This project is released under the MIT License.

See the `LICENSE` file for details.

---

## Author

**Your Name**

Big Data and AI Graduate focused on data engineering, Retrieval-Augmented Generation, scalable ingestion pipelines, and production AI systems.

* GitHub: https://github.com/moad-cod
* LinkedIn: https://www.linkedin.com/in/mouad-el-baz-812537332/
* Article: https://x.com/MouadEl_AI

---

## Acknowledgements

This project is inspired by research and engineering work on:

* Dense retrieval.
* Proposition-based retrieval.
* Late chunking.
* Parent-child retrieval.
* Hierarchical document representation.
* Retrieval-Augmented Generation evaluation.

Detailed research references should be added as each advanced strategy is implemented.
