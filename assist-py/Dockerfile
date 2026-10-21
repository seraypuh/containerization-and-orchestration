# stage 1 -- builder
FROM python:3.11-slim AS builder

WORKDIR /build
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt && \
    rm -rf /root/.cache/pip

# stage 2 -- runtime
FROM python:3.11-slim

LABEL maintainer="pukharev@lab.local"
LABEL version="1.0"

WORKDIR /app
COPY --from=builder /install /usr/local
COPY app/ ./app/

EXPOSE 5433
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:5433/health', timeout=3)" || exit 1


CMD  ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "5433"]