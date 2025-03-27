# Stage 1: Builder
FROM python:3.11-slim-bullseye AS builder

WORKDIR /app

# Install system dependencies with cache cleanup
RUN apt-get update -qq && \
    apt-get install -y --no-install-recommends \
    pkg-config \
    gcc \
    g++ \
    libicu-dev && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Create virtual environment
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Install build tools first with cache mount
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir --upgrade pip wheel hatchling build setuptools

# Copy minimal required files
COPY pyproject.toml setup.cfg ./
COPY libretranslate/ libretranslate/
COPY scripts/compile_locales.py scripts/

# Install core dependencies
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir \
    torch==2.0.1 \
    --extra-index-url https://download.pytorch.org/whl/cpu \
    "numpy<2" \
    Babel==2.12.1

# Compile translations
RUN python scripts/compile_locales.py

# Install application with proper isolation
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir -v .

# Stage 2: Runtime
FROM python:3.11-slim-bullseye

# Railway required settings
ENV PORT=8080
EXPOSE 8080

# Create non-root user
RUN addgroup --system ltgroup && \
    adduser --system --ingroup ltgroup ltuser && \
    mkdir -p /home/ltuser/.local && \
    chown -R ltuser:ltgroup /home/ltuser

USER ltuser
WORKDIR /app

# Copy virtual environment
COPY --from=builder --chown=ltuser:ltgroup /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Install models with Railway-optimized settings
ARG WITH_MODELS=false
ARG MODELS=""
RUN if [ "$WITH_MODELS" = "true" ]; then \
    python scripts/install_models.py --load_only_lang_codes "${MODELS:-en,es}"; \
    fi

# Runtime optimizations for Railway
ENV LT_NUM_THREADS=2 \
    LT_LOAD_ONLY=en,es \
    LT_HOST=0.0.0.0 \
    LT_PORT=8080

ENTRYPOINT ["libretranslate"]