# Lecture 2

# Telemost
https://telemost.yandex.ru/j/50516757095107

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

## Github Organization
https://github.com/miit-vsm-masters-2025

### Список юзернеймов или email'ов для инвайтов
- 1255015qwe@gmail.com
- W1zzK1
- Leejoen 
- FlashgoKless
- inputnickhere **(not found)** 
- s21dlemanskiy (s21d_lemanskiy@179.ru)
- ShagaVadim


## Markdown-редактор
https://md.wtrn.ru/fXv62VAaR82KeQg1BhXIhQ



# Сервис питания на борту
Исходим из следующих предположений:
- Заказ - через локальный сайт (aka Попутчик)
- В реальности еду скорее всего будут просто разносить готовую. Но для усложнения архитектуры считаем, что есть кухня и процесс готовки, а одни и те жеингридиенты могут использоваться в разных блюдах.
- Поддерживаем и предоплату (картой онлайн), и постоплату (терминал или наличные). 
- Понимаем, что в любой момент связь может отвалиться и онлайн-оплату завершить не удастся. Тогда даем пользователю вариант переключения на постоплату (терминал/наличные) или отказаться от заказа.

## Интерфейсы
* Для покупателей/пассажиры
* Для кладовщика
* Для поваров
* Для проводников/официантов

## База данных
### Таблицы
* Меню
    * ID
    * Название
    * Цена
    * Состав (JSONB Array)
        * ID ингридиента
        * Количество ингридиента
* Ингридиенты
    * ID
    * Название
    * Остаток
* Заказы
    * ID заказа
    * ID места
    * Время создания
    * ID транзакции
    * Форма оплаты
        * Оплата картой онлайн
        * Оплата картой терминал
        * Наличка
    * Статус
        * Ожидает оплаты
        * Оплачен
        * Готовится
        * Частично доставлен
        * Завершен
    * Стоимость
* Задачи на готовку
    * ID задачи
    * ID заказа
    * ID блюда (из меню)
    * Время создания
    * Статус
        * В очереди
        * Готовится
        * Готов
        * Доставляется
        * Доставлен
