FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    DB_PATH=/app/data/shop.db

WORKDIR /app

COPY requirements.txt .
RUN pip install --require-hashes --no-cache-dir -r requirements.txt

RUN useradd --uid 10001 --no-create-home --shell /usr/sbin/nologin appuser

COPY app.py .
COPY uploads/ uploads/

# Seed the demo database at build time; the app owns only the data/ dir.
RUN mkdir data \
    && SECRET_KEY=build-time-only python -c "import app; app.init_db()" \
    && chown -R appuser:appuser data

USER appuser

EXPOSE 5000
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "5000"]
