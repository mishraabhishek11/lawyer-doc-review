# ⚖️ Lawyer Doc Review – FastAPI Service

A lightweight **FastAPI application** for uploading **legal contracts** (PDF / TXT), storing them in **MongoDB**, and analysing them with the **Google Gemini API** to produce a summary, key clauses, risk flags and recommendations.

This project demonstrates **REST API design**, document text extraction, **LLM-based structured analysis**, and NoSQL persistence.

---

## 🚀 Features

### 📄 Contracts

- Upload a contract via `POST /contracts/upload` (`.pdf` or `.txt`, max 10 MB)
- Text, page count and word count are extracted automatically
- List contracts via `GET /contracts/` (text content omitted for brevity)
- Fetch a single contract via `GET /contracts/{contract_id}`

### 🤖 AI Analysis (Gemini)

- `POST /analysis/analyse/{contract_id}` sends the contract text to Gemini with a prompt tuned for **Indian contract law**
- The model returns structured JSON, validated with Pydantic, containing:
  - `summary` and `contract_type`
  - `key_clauses` – title, text, plain-language explanation, `is_standard`
  - `risk_flags` – title, description, `risk_level`, recommendation, clause reference
  - `overall_risk_level` (`low`, `medium`, `high`, `critical`)
  - `recommendations`
- Results are stored in MongoDB, so a contract can be analysed multiple times

### 📋 Retrieving Analyses

- `GET /analysis/` – all analyses
- `GET /analysis/{analysis_id}` – one analysis
- `GET /analysis/contract/{contract_id}` – all analyses for a contract

### ✅ Validation & Docs

- File type and size validation on upload
- Request/response models validated by Pydantic
- Auto-generated interactive docs (Swagger UI / ReDoc)

---

## 🛠️ Tech Stack

- **Language:** Python 3
- **Framework:** FastAPI
- **Validation:** Pydantic
- **Server:** Uvicorn
- **Database:** MongoDB (via PyMongo; database `mydb`, collections `contracts` and `analysis`)
- **AI:** Google Gemini API (`google-genai` SDK)
- **Document parsing:** PyPDF2
- **Config:** python-dotenv

---

## 📂 Project Structure

```
lawyer-doc-review/
├─ README.md
├─ requirements.txt
├─ docker-compose.yml      # Local MongoDB
└─ app/
   ├─ .env                 # MONGODB_URI, GEMINI_API_KEY (not committed)
   ├─ main.py              # FastAPI app, startup (index creation) and router registration
   ├─ config.py            # Env vars and upload limits
   ├─ database.py          # MongoDB client, collections and index creation
   ├─ models.py            # Contact, ClauseAnalysis, RiskFlag, AnalysisResult, RiskLevel
   ├─ routes/
   │  ├─ contracts.py      # /contracts endpoints
   │  └─ analysis.py       # /analysis endpoints
   └─ service/
      ├─ document_parser.py  # PDF / TXT text extraction
      ├─ gemini_analyse.py   # Gemini call and response parsing
      └─ prompt.py           # Prompt templates
```

---

## ⚙️ Installation & Setup

### Prerequisites

- Python 3.10+
- Docker (for MongoDB) or an existing MongoDB instance
- A [Gemini API key](https://aistudio.google.com/app/apikey)

### 1️⃣ Clone the repository

```bash
git clone <repository-url>
```

### 2️⃣ Navigate to the project folder

```bash
cd lawyer-doc-review
```

### 3️⃣ Create and activate a virtual environment

```bash
python -m venv .venv
# Windows
.venv\Scripts\activate
# macOS / Linux
source .venv/bin/activate
```

### 4️⃣ Install dependencies

```bash
pip install -r requirements.txt
```

### 5️⃣ Start MongoDB

```bash
docker compose up -d
```

This starts MongoDB on port `27017` with a persistent `mongodb_data` volume. The root credentials are defined in `docker-compose.yml` – change them for anything beyond local development.

### 6️⃣ Configure environment variables

Create `app/.env`:

```env
MONGODB_URI=mongodb://root:mypassword@localhost:27017/?authSource=admin
GEMINI_API_KEY=your-gemini-api-key
```

| Variable | Description |
|----------|-------------|
| `MONGODB_URI` | MongoDB connection string |
| `GEMINI_API_KEY` | Google Gemini API key (required for `/analysis/analyse/...`) |

`.env` is git-ignored – never commit real keys.

### 7️⃣ Start the server

```bash
cd app
uvicorn main:app --reload
```

### 8️⃣ Open in browser

- Swagger UI: [http://localhost:8000/docs](http://localhost:8000/docs)
- ReDoc: [http://localhost:8000/redoc](http://localhost:8000/redoc)

---

## 🧑‍💻 Usage

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/contracts/upload` | Upload a PDF/TXT contract |
| GET | `/contracts/` | List uploaded contracts |
| GET | `/contracts/{contract_id}` | Get one contract (includes extracted text) |
| POST | `/analysis/analyse/{contract_id}` | Analyse a contract with Gemini |
| GET | `/analysis/` | List all analyses |
| GET | `/analysis/{analysis_id}` | Get one analysis |
| GET | `/analysis/contract/{contract_id}` | List analyses for a contract |

### Upload a contract

```bash
curl -X POST http://localhost:8000/contracts/upload \
  -F "file=@sample-nda.pdf"
```

Response:

```json
{
  "message": "File uploaded and processed successfully",
  "contract": {
    "id": "665f1c2e8a1b2c3d4e5f6a7b",
    "filename": "3f2a9c...e1.pdf",
    "original_name": "sample-nda.pdf",
    "upload_date": "2026-10-08T10:30:00.000000",
    "text_content": "...",
    "page_count": 4,
    "word_count": 1520,
    "status": "uploaded"
  },
  "id": "665f1c2e8a1b2c3d4e5f6a7b"
}
```

### Analyse a contract

```bash
curl -X POST http://localhost:8000/analysis/analyse/665f1c2e8a1b2c3d4e5f6a7b
```

Response (abridged):

```json
{
  "message": "Contract analyzed successfully",
  "analysis": {
    "contract_id": "665f1c2e8a1b2c3d4e5f6a7b",
    "summary": "A mutual non-disclosure agreement between ...",
    "contract_type": "NDA",
    "key_clauses": [
      {
        "clause_title": "Confidentiality Period",
        "clause_text": "...",
        "explanation": "Information must be kept secret for 5 years.",
        "is_standard": true
      }
    ],
    "risk_flags": [
      {
        "risk_title": "One-sided termination",
        "description": "Only the disclosing party may terminate.",
        "risk_level": "medium",
        "recommendation": "Negotiate mutual termination rights.",
        "clause_reference": "Clause 9"
      }
    ],
    "overall_risk_level": "medium",
    "recommendations": ["Negotiate mutual termination rights."]
  },
  "id": "665f1d008a1b2c3d4e5f6a7c"
}
```

### Retrieve analyses

```bash
curl http://localhost:8000/analysis/contract/665f1c2e8a1b2c3d4e5f6a7b
```

---

## 🗄️ Data Model

### `contracts` collection

| Field | Type | Notes |
|-------|------|-------|
| `_id` | ObjectId | Primary key, auto-generated |
| `filename` | str | Unique stored name (UUID + extension); unique index |
| `original_name` | str | Name of the uploaded file |
| `upload_date` | str | ISO timestamp, set automatically |
| `text_content` | str | Extracted text |
| `page_count` | int | PDF pages (TXT is counted as 1 page) |
| `word_count` | int | |
| `status` | str | Default `uploaded` |
| `analysis_status` | str | `in_progress` / `completed`, set when analysis runs |

### `analysis` collection

| Field | Type | Notes |
|-------|------|-------|
| `_id` | ObjectId | Primary key, auto-generated |
| `contract_id` | str | Reference to `contracts._id`; indexed |
| `analysis_date` | str | ISO timestamp, set automatically |
| `summary` | str | |
| `contract_type` | str | |
| `key_clauses` | list | `clause_title`, `clause_text`, `explanation`, `is_standard` |
| `risk_flags` | list | `risk_title`, `description`, `risk_level`, `recommendation`, `clause_reference` |
| `overall_risk_level` | enum | `low`, `medium`, `high`, `critical` |
| `recommendations` | list[str] | |

Collections and indexes are created on application startup. Uploaded files are also saved to the `uploads/` directory (relative to where the server is started).

---

## 🚧 Known Issues / Work in Progress

The following are present in the current code and are not yet fixed:

- Only the first 15,000 characters of a contract are sent to Gemini (`text_content[:15000]`), so long contracts are partially analysed.
- Invalid ObjectId strings in the path raise an unhandled `bson.errors.InvalidId` (HTTP 500) instead of a 400/404.
- If the Gemini call or JSON parsing fails, the contract's `analysis_status` stays `in_progress` and the client gets a 500.
- `GET /analysis/{analysis_id}` returns the raw Mongo document including an `ObjectId`, which is likely not JSON-serialisable; the list endpoints convert `_id` to `id`.
- `GET /analysis/` is missing from the endpoint list returned by `GET /`.
- `gemini_analyse.py` prints the raw model response and has an unused `GEMINI_URL` constant.
- The database name `mydb` is hard-coded in `database.py`.
- Blocking PyMongo / Gemini calls are made inside `async` handlers.
- `requirements.txt` is UTF-16 encoded; re-save as UTF-8 if your tooling has trouble reading it.

---

## 🔮 Future Enhancements

- Delete contracts / analyses
- Use the currently unused `CLAUSE_EXTRACTION_PROMPT`, `RISK_ASSESSMENT_PROMPT` and `SUMMARY_PROMPT` templates for chunked analysis of long contracts
- OCR support for scanned PDFs
- Consistent error response shape
- Unit and integration tests
- Authentication
- Dockerfile for the API itself

---

## ⚠️ Disclaimer

AI-generated output is for **informational purposes only** and is not legal advice. Always have a qualified lawyer review important contracts.

---

## 🤝 Contributing

1. Fork the repository
2. Create a new branch: `git checkout -b feat/your-feature`
3. Commit your changes: `git commit -m "feat(scope): add your message"`
4. Push to the branch: `git push origin feat/your-feature`
5. Open a Pull Request

---

## 👨‍💻 Author

Abhishek Mishra  
GitHub: [https://github.com/mishraabhishek11](https://github.com/mishraabhishek11)
