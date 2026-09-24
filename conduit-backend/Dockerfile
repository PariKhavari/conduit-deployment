# ---------- Base stage: shared Debian archive fix ----------
FROM python:3.6-slim-buster AS base
RUN sed -i \
    -e 's|deb.debian.org/debian|archive.debian.org/debian|g' \
    -e 's|security.debian.org/debian-security|archive.debian.org/debian-security|g' \
    -e '/buster-updates/d' \
    /etc/apt/sources.list


# ---------- Stage 1: Builder ----------
FROM base AS builder
WORKDIR /app
RUN apt-get update \
    && apt-get install -y --no-install-recommends build-essential libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt


# ---------- Stage 2: Runtime ----------
FROM base AS runtime

RUN apt-get update \
    && apt-get install -y --no-install-recommends libpq5 \
    && rm -rf /var/lib/apt/lists/*
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app
COPY --from=builder /install /usr/local
COPY --chown=appuser:appuser . .

RUN chmod +x entrypoint.sh
ENV DJANGO_SETTINGS_MODULE=conduit.settings PYTHONUNBUFFERED=1
RUN mkdir -p /app/staticfiles && chown -R appuser:appuser /app/staticfiles

USER appuser
EXPOSE 8000

ENTRYPOINT ["/bin/sh", "/app/entrypoint.sh"]
CMD ["gunicorn", "conduit.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "3"]