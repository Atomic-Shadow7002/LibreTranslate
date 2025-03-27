# Change base image to Python 3.11 (3.10 has compatibility issues)
FROM python:3.11-slim-bullseye AS builder

WORKDIR /app

# Add missing ICU library dependency
ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get update -qq \
  && apt-get -qqq install --no-install-recommends -y \
    pkg-config \
    gcc \
    g++ \
    libicu-dev \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*  # Better cleanup

# Create virtual environment in standard location
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Install build tools first
RUN pip install --no-cache-dir --upgrade "pip==23.3.1" "build==0.10.0" "setuptools==68.2.2" "hatchling==1.18.0"

COPY . .

# Modified build sequence
RUN pip install "Babel==2.12.1" && \
    python scripts/compile_locales.py && \
    pip install "torch==2.0.1+cpu" --extra-index-url https://download.pytorch.org/whl/cpu && \
    pip install "numpy<1.26" && \
    pip install --no-cache-dir -e .  # Use editable install

# ------------------------
# Runtime Stage
# ------------------------
FROM python:3.11-slim-bullseye

# Railway requires PORT env var
ENV PORT=5000
EXPOSE 5000

# User setup remains same
RUN addgroup --system --gid 1032 libretranslate && \
    adduser --system --uid 1032 libretranslate && \
    mkdir -p /home/libretranslate/.local && \
    chown -R libretranslate:libretranslate /home/libretranslate/.local

USER libretranslate
WORKDIR /app

# Copy entire venv from builder
COPY --from=builder --chown=1032:1032 /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Model installation fix
ARG with_models=false
ARG models=""
RUN if [ "$with_models" = "true" ]; then \
      if [ -n "$models" ]; then \
        python scripts/install_models.py --load_only_lang_codes "$models"; \
      else \
        python scripts/install_models.py; \
      fi \
    fi

# Railway-specific entrypoint
ENTRYPOINT [ "libretranslate", "--host", "0.0.0.0", "--port", "5000" ]