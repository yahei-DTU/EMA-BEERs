# Serving Models with FastAPI

Once a model is trained, you need a way for other systems to use it. The standard approach is to wrap it in an **API** — a server that accepts requests over HTTP and returns predictions. **FastAPI** is a modern Python framework for building APIs quickly, with automatic input validation and interactive documentation built in.

## What is FastAPI?

FastAPI lets you define API endpoints as plain Python functions. It handles the HTTP layer, validates incoming data, serialises responses to JSON, and generates interactive documentation automatically.

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def health_check():
    return {"status": "ok"}
```

That is a complete, runnable API. FastAPI is built on **Starlette** for the HTTP layer and **Pydantic** for data validation.

## Installing FastAPI

FastAPI and its server `uvicorn` are already in the project's dev dependencies. After `uv sync` they are available immediately:

```bash
uv add fastapi uvicorn[standard]
```

## Running the API

Start the development server with:

```bash
uv run uvicorn src.ema_beers.api:app --reload
```

- `src.ema_beers.api` — the Python module path to your `api.py` file
- `app` — the `FastAPI()` instance inside that file
- `--reload` — restarts the server automatically when you save a file

The server starts at `http://127.0.0.1:8000`.

## Defining Endpoints

Endpoints are Python functions decorated with the HTTP method and path:

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
def health_check():
    return {"status": "ok"}

@app.get("/items/{item_id}")
def get_item(item_id: int):
    return {"item_id": item_id}
```

Path parameters (like `item_id`) are declared as function arguments. FastAPI automatically converts them to the annotated type and returns a 422 error if the conversion fails.

## Request Bodies with Pydantic

For `POST` requests that carry data (e.g. sending features to a model), define the expected shape using a **Pydantic model**:

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class PredictionRequest(BaseModel):
    feature_1: float
    feature_2: float
    feature_3: float

class PredictionResponse(BaseModel):
    label: int
    probability: float

@app.post("/predict", response_model=PredictionResponse)
def predict(request: PredictionRequest):
    features = [request.feature_1, request.feature_2, request.feature_3]
    label, probability = model.predict(features)
    return PredictionResponse(label=label, probability=probability)
```

FastAPI validates the incoming JSON against `PredictionRequest` automatically. If a required field is missing or has the wrong type, it returns a clear error response without any extra code.

## Loading a Model at Startup

You do not want to reload the model on every request — that would be extremely slow. Use a **lifespan** context manager to load the model once when the server starts and keep it in memory:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

model = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global model
    model = load_model("models/best_model.pt")  # runs once at startup
    yield
    model = None  # runs once at shutdown

app = FastAPI(lifespan=lifespan)

@app.post("/predict")
def predict(request: PredictionRequest):
    label, probability = model.predict([request.feature_1, request.feature_2])
    return {"label": label, "probability": probability}
```

## Interactive Documentation

FastAPI automatically generates two interactive documentation pages:

- **`/docs`** — Swagger UI: lets you call any endpoint directly from the browser
- **`/redoc`** — ReDoc: a cleaner read-only view of the API spec

These update live as you add or change endpoints. They are invaluable for testing the API manually during development.

## Testing the API

FastAPI comes with a `TestClient` that lets you write tests for your endpoints without running a real server:

```python
from fastapi.testclient import TestClient
from ema_beers.api import app

client = TestClient(app)

def test_health_check():
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json() == {"status": "ok"}

def test_predict_returns_label():
    response = client.post("/predict", json={
        "feature_1": 1.0,
        "feature_2": 2.0,
        "feature_3": 3.0,
    })
    assert response.status_code == 200
    assert "label" in response.json()
```

These tests integrate naturally with pytest and are run as part of the `tests.yaml` CI workflow.

## Running with Docker

The project includes `dockerfiles/api.dockerfile` for packaging the API in a container:

```bash
docker build -f dockerfiles/api.dockerfile -t ema-beers-api .
docker run -p 8000:8000 ema-beers-api
```

The container starts uvicorn and exposes port 8000, which maps to `http://localhost:8000` on your machine.

!!! note
    The `--reload` flag is for development only. Never use it in production. In a Docker container or on a server, run uvicorn without `--reload` and increase the number of workers for better throughput:
    ```bash
    uvicorn src.ema_beers.api:app --host 0.0.0.0 --port 8000 --workers 4
    ```

## Exercise

!!! example "Exercise"
    1. Create a basic FastAPI app in `src/ema_beers/api.py` with a `/health` endpoint that returns `{"status": "ok"}`.

    2. Start the server and verify it works:

        ```bash
        uv run uvicorn src.ema_beers.api:app --reload
        ```

    3. Open `http://127.0.0.1:8000/docs` in your browser and try calling the endpoint from the Swagger UI.

    4. Add a `/predict` endpoint that accepts a JSON body with input features and returns a dummy prediction.

    5. Write a test for both endpoints in `tests/test_api.py` using `TestClient` and run them:

        ```bash
        uv run pytest tests/test_api.py -v
        ```

    6. **Bonus:** load a trained model at startup using the lifespan context manager and return real predictions from `/predict`.
