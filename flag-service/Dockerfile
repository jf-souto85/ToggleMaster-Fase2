FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt "Werkzeug==2.2.3"

RUN groupadd --system app \
    && useradd --system --gid app app

COPY --chown=app:app . .

USER app

EXPOSE 8002

CMD ["gunicorn", "--bind", "0.0.0.0:8002", "--access-logfile", "-", "--error-logfile", "-", "app:app"]
