# Stage 1: Builder
FROM python:3.11-slim-bullseye AS builder

WORKDIR /app

# Install system dependencies with cleanup
RUN apt-get update -qq \
  && apt-get install -y --no-install-recommends \
    pkg-config \
    gcc \
    g++ \
    libicu-dev \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*

# Create virtual environment
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Install build tools first
RUN pip install --no-cache-dir --upgrade pip wheel hatchling

# Copy required files
COPY pyproject.toml setup.cfg setup.py ./
COPY libretranslate/ libretranslate/
COPY scripts/ scripts/

# Install core dependencies
RUN pip install --no-cache-dir \
    torch==2.0.1 \
    --extra-index-url https://download.pytorch.org/whl/cpu \
    "numpy<2" \
    Babel==2.12.1

# Compile translations and install package
RUN python scripts/compile_locales.py \
  && pip install --no-cache-dir . \
  && pip cache purge

# Stage 2: Runtime
FROM python:3.11-slim-bullseye

# Railway requires PORT environment variable
ENV PORT=8080
EXPOSE 8080

# Create non-root user
RUN addgroup --system --gid 1032 ltuser \
  && adduser --system --uid 1032 --ingroup ltuser ltuser \
  && mkdir -p /home/ltuser/.local \
  && chown -R ltuser:ltuser /home/ltuser

USER ltuser
WORKDIR /app

# Copy virtual environment from builder
COPY --from=builder --chown=ltuser:ltuser /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Install models (if needed)
ARG WITH_MODELS=false
ARG MODELS=""
RUN if [ "$WITH_MODELS" = "true" ]; then \
      if [ -n "$MODELS" ]; then \
        python scripts/install_models.py --load_only_lang_codes "$MODELS"; \
      else \
        python scripts/install_models.py; \
      fi \
    fi

# Railway-specific optimizations
ENV LT_NUM_THREADS=1 \
    LT_LOAD_ONLY=en,es,fr \
    LT_HOST=0.0.0.0

ENTRYPOINT ["libretranslate", "--host", "0.0.0.0", "--port", "8080"]