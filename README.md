# 📚 Arabic Grammar Bot

A Retrieval-Augmented Generation (RAG) API that answers questions about **Arabic grammar (النحو)**. It searches a knowledge base of grammar texts, then asks a Gemma language model to give a short, accurate answer with a single example, along with the sources it used.

## How It Works

```
Question ──► Multilingual embedding ──► ChromaDB search (top 3 passages)
                                              │
                                              ▼
                       Prompt = reference passages + question
                                              │
                                              ▼
                         Gemma 3 27B (Google Generative AI)
                                              │
                                              ▼
                          Short answer + list of sources
```

1. The question is embedded with `paraphrase-multilingual-MiniLM-L12-v2`.
2. ChromaDB returns the 3 most relevant grammar passages from `grammar_db_flash`.
3. The passages and the question are placed in an Arabic prompt that asks for a direct definition plus one very short example.
4. `gemma-3-27b-it` generates the answer. Rate-limit errors (HTTP 429) are retried up to 3 times.

## Tech Stack

- [FastAPI](https://fastapi.tiangolo.com/) and Uvicorn: the API server
- [ChromaDB](https://www.trychroma.com/): the vector database
- [Sentence-Transformers](https://www.sbert.net/): multilingual embeddings
- [Google Generative AI](https://ai.google.dev/) (`gemma-3-27b-it`): answer generation
- Docker: containerized deployment (configured for Hugging Face Spaces)

## Project Structure

```
grammar-bot/
├── server.py               # FastAPI app and RAG logic
├── grammar_db_flash.zip    # Pre-built ChromaDB knowledge base (unzip before running)
├── requirements.txt        # Python dependencies
├── Dockerfile              # Container for Hugging Face Spaces (port 7860)
└── start.sh                # Start script for platforms that provide $PORT
```

## Getting Started

### Prerequisites
- Python 3.9 or newer
- A Google AI API key from [Google AI Studio](https://aistudio.google.com/app/apikey)

### 1. Clone the repository

```bash
git clone https://github.com/chaima-m/grammar-bot.git
cd grammar-bot
```

### 2. Install dependencies

```bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 3. Extract the knowledge base

The server reads the database from the `./grammar_db_flash` folder, so unzip it first:

```bash
unzip grammar_db_flash.zip
```

### 4. Set your API key

Provide the key through an environment variable. Never write it in the source code.

```bash
export GOOGLE_API_KEY="your-key-here"     # Windows (PowerShell): $env:GOOGLE_API_KEY="your-key-here"
```

> `server.py` must read the key with `os.environ["GOOGLE_API_KEY"]` for this to work.

### 5. Run the server

```bash
python server.py
```

The API is now available at http://localhost:8000, with interactive docs at http://localhost:8000/docs.

## API Reference

### `POST /ask`

Ask a grammar question (in Arabic).

**Request body**

```json
{
  "question": "ما هو الفاعل؟"
}
```

**Response**

```json
{
  "answer": "الفاعل هو الاسم المرفوع الذي يدل على من قام بالفعل. مثال: كتبَ الطالبُ الدرسَ.",
  "sources": ["Source name #12", "Source name #48"]
}
```

**Example with curl**

```bash
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "ما هو الفاعل؟"}'
```

**Errors**

| Status | Meaning |
|--------|---------|
| 400 | The question is empty |
| 422 | Invalid request body |

If no relevant passage is found, the answer is `لا توجد معلومات كافية.` ("Not enough information.").

## Docker

```bash
docker build -t grammar-bot .
docker run -p 7860:7860 -e GOOGLE_API_KEY="your-key-here" grammar-bot
```

The container serves the API on port **7860**, the port Hugging Face Spaces expects. Make sure the database is extracted inside the image (for example by adding `RUN unzip grammar_db_flash.zip` to the Dockerfile).

## Deployment

- **Hugging Face Spaces (Docker):** push the repo to a Space and add `GOOGLE_API_KEY` as a **secret** in the Space settings.
- **Render, Railway or similar:** use `start.sh` as the start command. The platform provides `$PORT`, and you set `GOOGLE_API_KEY` in its environment settings.

## Configuration

| Setting | Where | Default |
|---------|-------|---------|
| Model | `MODEL_NAME` in `server.py` | `gemma-3-27b-it` |
| Database path | `DB_PATH` in `server.py` | `./grammar_db_flash` |
| Passages retrieved | `n_results` in `server.py` | `3` |
| Embedding model | `server.py` | `paraphrase-multilingual-MiniLM-L12-v2` |

## Limitations

- Answers are only as good as the knowledge base. Questions outside it may return no answer.
- The prompt is tuned for short answers, not detailed explanations.
- The free Gemini API tier has rate limits, so you may see "Server busy (429)" under heavy use.
- Language models can make mistakes. Check important grammar rulings against a reliable reference.

## Roadmap

- Add CORS and a simple web or chat front end
- Support follow-up questions with conversation history
- Add API authentication and rate limiting
- Add more grammar sources to the knowledge base

## License

All rights reserved unless you add a license file. Contact the author for permission to use it.

## Author

Made by [chaima-m](https://github.com/chaima-m)
