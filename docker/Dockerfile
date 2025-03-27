# Stage 1: Builder with explicit Python 3.11
FROM python:3.11.13-slim-bullseye AS builder

WORKDIR /app

# 1. System dependencies
RUN apt-get update -qq && \
    apt-get install -y --no-install-recommends \
    pkg-config \
    gcc \
    g++ \
    libicu-dev && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# 2. Create isolated venv
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# 3. Install build tools FIRST
RUN pip install --no-cache-dir --upgrade "pip==24.0" "hatchling==1.24.0" "build==1.2.1"

# 4. Copy ONLY required build files
COPY pyproject.toml setup.cfg ./
COPY libretranslate/__init__.py libretranslate/
COPY scripts/compile_locales.py scripts/

# 5. Install core deps with version pins
RUN pip install --no-cache-dir \
    "torch==2.0.1+cpu" \
    --extra-index-url https://download.pytorch.org/whl/cpu \
    "numpy<1.26" \
    "Babel==2.12.1"

# 6. Compile translations
RUN python scripts/compile_locales.py

# 7. Install application with explicit source
RUN pip install --no-cache-dir -e .

# Stage 2: Runtime
FROM python:3.11.13-slim-bullseye

# Railway mandatory settings
ENV PORT=8080
EXPOSE 8080

# Non-root user setup
RUN addgroup --system ltgrp && \
    adduser --system --ingroup ltgrp ltusr && \
    mkdir -p /home/ltusr/.local && \
    chown -R ltusr:ltgrp /home/ltusr

USER ltusr
WORKDIR /app

# Copy venv with proper permissions
COPY --from=builder --chown=ltusr:ltgrp /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Model installation conditional
ARG INSTALL_MODELS=false
ARG MODELS="en,es"
RUN if [ "$INSTALL_MODELS" = "true" ]; then \
    python -c "from libretranslate import main; main.load_models()" --load-only "$MODELS"; \
    fi

# Runtime optimizations
ENV LT_NUM_THREADS=1 \
    LT_LOAD_ONLY=en,es \
    LT_HOST=0.0.0.0

ENTRYPOINT ["libretranslate", "--host", "0.0.0.0", "--port", "8080"]