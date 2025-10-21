# Resume Ingestion Pipeline – DOCX → Chunks → LLM Metadata → Weaviate
This repository contains a Jupyter notebook **`Resume_Ingestion_Pipeline.ipynb`** that ingests resumes from `.docx` files, cleans and chunks the text, extracts structured candidate metadata using an LLM, and indexes both the chunks and extracted metadata into a **Weaviate** vector database (with **OpenAI** embeddings).

Note - Following aspects are missings and need to work on it
1.This is not a production ready code yet
2.Data Ingestion for Resume can be made in parallel processing way
3.No Evaluation stratey is added, whci can be added
4.No Agentic aspect added yet

**High level flow**
1. **Partition & Read**: Load `.docx` resumes and extract text using `unstructured`.
2. **Chunking**: Split the text into semantically meaningful chunks (title‑aware) and clean the content.
3. **LLM Extraction**: Use `LangChain + OpenAI` (Chat model) to extract structured fields:
   - `experience_years_total` (number, with decimals if possible)
   - `skills` (normalized names)
   - `certifications` (exact names, if present)
   - `key_responsibilities` (list)
4. **Vectorization & Storage**: Push cleaned chunks and metadata to **Weaviate**, configured with **text-embedding-3-small** for vectors.

## ✨ Features
- DOCX ingestion via `unstructured.partition.docx`
- Title‑aware chunking (`unstructured.chunking.title.chunk_by_title`)
- Text cleaning utilities (bullet/dash/ASCII cleanup, lowercasing, trailing punctuation fixes)
- LLM JSON extraction with `LangChain` and `ChatOpenAI` (prompt provided in notebook)
- Weaviate Cloud connection and **collection auto‑creation** with schema:
  - `resume_link` (TEXT)
  - `key_responsibilities` (TEXT_ARRAY)
  - `certifications` (TEXT_ARRAY)
  - `skills` (TEXT_ARRAY)
  - Vectorization configs for OpenAI embeddings
- Deterministic UUIDs via `weaviate.util.generate_uuid5`

## 🧱 Architecture (Notebook Stages)
```
[.docx resumes]
      │
      ▼
[Unstructured partitioning]
      │
      ▼
[Chunking: chunk_by_title]
      │
      ▼
[Cleaning: bullets/dashes/ASCII, lowercase, punctuation]
      │
      ├──► [LLM (ChatOpenAI) JSON extraction: experience, skills, certs, key_responsibilities]
      │
      ▼
[Weaviate collection (auto-create if missing)]
      │
      └──► [Ingest: chunks + normalized metadata, vectorized with OpenAI embeddings]
```

## 🧩 Requirements
- Python 3.10+ (recommended)
- Jupyter Notebook
- Accounts & API keys:
  - **OpenAI API key** for Chat + Embeddings
  - **Weaviate** Cloud URL and API key

Install Python dependencies (from notebook cells):
```bash
pip install -U langchain-openai langchain langchain-community sentence-transformers unstructured[all-docs]
pip install weaviate-client==4.16.6
tqdm python-docx
```

## 🔐 Environment Variables
Create a `.env` (or export in your shell) **before** running the notebook:

```bash
# OpenAI
export OPENAI_API_KEY="YOUR_OPENAI_API_KEY"

# Weaviate (Cloud)
export WEAVIATE_URL="YOUR_WCS_CLUSTER_URL"             # e.g., abc123.c0.region.gcp.weaviate.cloud
export WEAVIATE_API_KEY="YOUR_WCS_API_KEY"
```

> **Security Note:** The notebook currently shows placeholders and example values in cells. > Always prefer environment variables or a secrets manager. Avoid committing secrets to version control.

## ⚙️ Setup & Project Structure
```
.
├─ Resume_Ingestion_Pipeline.ipynb   # The main notebook
├─ data/
│  └─ resumes/                       # Put your .docx resumes here
└─ README.md
```

Create the data folder and add your `.docx` files:
```bash
mkdir -p data/resumes
cp /path/to/resumes/*.docx data/resumes/
```

## 🚀 Usage
1. Open the notebook:
   ```bash
   jupyter notebook Resume_Ingestion_Pipeline.ipynb
   ```
2. Ensure your environment variables (OpenAI/Weaviate) are set.
3. Run the cells top‑to‑bottom. The typical sequence is:
   - **Load Resumes**: scans `data/resumes/*.docx` (see `load_resumes`)
   - **Partition & Chunk**: uses `unstructured` + `chunk_by_title`
   - **Clean Chunks**: `clean_chunks` helper lowers case, removes bullets/dashes, trims punctuation
   - **LLM Extraction**: prompt (`PROMPT_TEMPLATE`) + `ChatOpenAI` → JSON with experience/skills/certs/responsibilities
   - **Weaviate**:
     - Connect to your cluster (`weaviate.connect_to_weaviate_cloud`)
     - Create collection if missing (see schema cell)
     - Ingest each chunk + associated metadata

### LLM Prompt (summarized)
The notebook defines:
```text
Act like an expert in extracting resume facts and respond in strict JSON.
Input: {resume_text}

Keys:
- experience_years_total: number
- skills: string[]
- certifications: string[]
- key_responsibilities: string[]
```
You can modify the template in the notebook to add fields (e.g., location/education).

## 🗂 Weaviate Collection Schema (from notebook)
- **Collection name**: `resume`
- **Properties**:
  - `resume_link`: TEXT
  - `key_responsibilities`: TEXT_ARRAY
  - `certifications`: TEXT_ARRAY
  - `skills`: TEXT_ARRAY
- **Vectors**:
  - OpenAI embeddings `text-embedding-3-small`
  - Example vectorization configs (`Configure.Vectorizer.text2vec_openai`)

> Tip: You can add another property like `resume_chunk` (TEXT) to store the raw chunk text, or link to original file.

## ⚙️ Configuration Knobs
- **Model**: `gpt-4o` (change `ChatOpenAI(model=...)` if needed)
- **Embedding Model**: `text-embedding-3-small` (update in Weaviate config)
- **Chunking**: tweak `chunk_by_title` parameters to fit your resume style
- **Normalization**: adjust cleaning helpers in `clean_chunks`
- **Collection Name**: change `CollectionName = "resume"` if you need a different namespace

## 🔧 Extending the Pipeline
- **Support PDFs**: use `unstructured.partition.pdf`
- **Add more fields**: education, companies, roles, time ranges
- **Rerankers**: integrate `sentence-transformers` CrossEncoders for retrieval reranking
- **Quality checks**: validate JSON schema & handle LLM fallbacks/retries
- **Synchronous/Batch ingest**: convert notebook cells to a Python script/CLI for automation

## 🧯 Troubleshooting
- **Weaviate auth/URL errors**: confirm `WEAVIATE_URL` and `WEAVIATE_API_KEY`
- **OpenAI errors / rate limits**: verify `OPENAI_API_KEY` and model availability in your region
- **`unstructured` parsing quirks**: resumes vary—try alternative partitioners or pre‑clean DOCX
- **Vectorizer mismatch**: ensure the Weaviate module for OpenAI embeddings is enabled and models match config

## 📄 License
This project is provided as‑is for educational/demo purposes. Ensure you have the right to process any resumes you ingest and comply with data protection policies.
