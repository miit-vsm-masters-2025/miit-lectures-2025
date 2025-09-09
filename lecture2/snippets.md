# Lecture 2 snippets

## uv
https://docs.astral.sh/uv/

## Fastapi app
https://fastapi.tiangolo.com/

```python
import logging
import uvicorn
from typing import Union
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
def read_root():
    return {"Hello": "World"}


@app.get("/items/{item_id}")
def read_item(item_id: int, q: Union[str, None] = None):
    return {"item_id": item_id, "q": q}


if __name__ == '__main__':
    logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)-8s %(message)s")
    uvicorn.run(app, host="0.0.0.0")
```

## Settings

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    app_name: str
    max_food_supply_kg: int = 100

    model_config = SettingsConfigDict(env_file="config.env")
```

## Docker
```Dockerfile
FROM python:3.13-slim

WORKDIR /app

RUN pip install uv

COPY pyproject.toml uv.lock ./
RUN uv sync

COPY . .

CMD ["uv", "run", "python", "main.py"]
```

## build.sh
```bash
set -e
docker build -t miit/demo-app .
#docker push miit/wtrn-news
```

## Docker Compose
```yaml
services:
  app:
    build: .
    ports:
      - "8000:8000"
    restart: unless-stopped
```