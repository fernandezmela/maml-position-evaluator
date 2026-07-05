# Opening-Task MAML — Deployed Position Evaluator

An interactive demo that takes the meta-learning research from
[`opening-task-maml`](.) and serves it as a working web app: enter a chess
or shogi position, optionally tag it with an opening family, and watch the
model's inner-loop adaptation happen step by step.

This exists to answer one question a research-only repo can't: **can this
model run as a real service, not just a training script?**

## Architecture

```
frontend (index.html) --HTTP--> backend (FastAPI) --calls--> model.py (PyTorch)
                                                    --calls--> board_encoder.py
```

- **`frontend/index.html`** — single-file board UI. No build step, no
  framework — open it directly in a browser once the backend is running.
- **`backend/app/main.py`** — FastAPI routes. `/evaluate` is the only
  endpoint that matters; `/docs` gets you interactive Swagger docs for free.
- **`backend/app/model.py`** — adapter around the PyTorch model. Ships with
  a small dummy CNN so the whole stack runs today with zero setup.
  `load_real_model()` is the one function to fill in with your actual
  `opening-task-maml` checkpoint.
- **`backend/app/board_encoder.py`** — converts FEN (chess) / SFEN (shogi)
  strings into the tensors the model consumes. This is the other seam
  that's specific to your training pipeline — match it to whatever
  encoding you used when training.

## Running it locally

```bash
cd backend
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```

Then open `frontend/index.html` directly in your browser (double-click it,
or `open frontend/index.html` on macOS). It talks to `http://localhost:8000`
by default — change `API_BASE` at the top of the `<script>` tag if you
deploy the backend elsewhere.

## Running with Docker

```bash
cd backend
docker build -t maml-evaluator .
docker run -p 8000:8000 maml-evaluator
```

## Plugging in your real model

Two files need your real logic, everything else stays as-is:

1. **`board_encoder.py`** — replace `encode_chess` / `encode_shogi` with
   whatever tensor encoding you used during training.
2. **`model.py`** — replace `TinyEvalNet` with your real architecture
   (import it instead of defining it here), then call
   `evaluator.load_real_model(chess_ckpt_path=..., shogi_ckpt_path=...)`
   at startup in `main.py`. Also replace the placeholder loop inside
   `evaluate()` with your actual MAML inner-loop fine-tuning steps on a
   support set for the given `task_context`.

## Deploying it publicly

Cheapest path for a portfolio demo:
- **Backend**: [Render](https://render.com) or [Fly.io](https://fly.io) free
  tier, using the included Dockerfile.
- **Frontend**: GitHub Pages, or just serve `index.html` from the same
  container as a static file.

Once deployed, put the live link at the top of your GitHub profile README —
that's the whole point of doing this.

## Why this structure

The adapter pattern (`model.py` never touching the API layer, `board_encoder.py`
never touching model internals) means the serving/API code is fully testable
without a GPU or trained weights, and the research code can evolve
independently of the deployment code. That separation of concerns is itself
something worth mentioning in an interview.
