FROM  nvidia/cuda:12.4.1-base-ubuntu22.04

# Set non-interactive frontend
ENV DEBIAN_FRONTEND=noninteractive
ENV UV_LINK_MODE=copy
ENV UV_COMPILE_BYTECODE=1
ENV PYTHONUNBUFFERED=1

# 2. Install system dependencies
RUN apt-get update && apt-get install -y \
    python3.10 python3-venv espeak-ng espeak-ng-data \
    git libsndfile1 curl ffmpeg g++ \
 && apt-get clean && rm -rf /var/lib/apt/lists/*

# 3. Install UV properly
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

# 4. Create user and set up workspace
RUN useradd -m -u 1001 appuser
WORKDIR /app
RUN chown appuser:appuser /app

# 5. COPY ONLY LOCKFILES FIRST (The secret to caching)
# This layer only rebuilds if you change your dependencies
# This copies pyproject.toml and ONLY copies uv.lock if it exists
COPY --chown=appuser:appuser pyproject.toml uv.lock* ./

# 6. Install dependencies
USER appuser
RUN --mount=type=cache,target=/home/appuser/.cache/uv,uid=1001 \
    uv venv --python 3.10 && \
    uv sync --no-install-project --extra gpu

# 7. Copy the rest of the project
COPY --chown=appuser:appuser api ./api
COPY --chown=appuser:appuser web ./web
COPY --chown=appuser:appuser docker/scripts/ ./
USER root
RUN chmod +x ./entrypoint.sh
USER appuser

# 8. Final environment setup
ENV PATH="/app/.venv/bin:$PATH" \
    PYTHONPATH=/app:/app/api \
    PHONEMIZER_ESPEAK_PATH=/usr/bin \
    USE_GPU=true \
    DEVICE="gpu"

# Run final sync to install the project code itself (fast)
RUN --mount=type=cache,target=/home/appuser/.cache/uv,uid=1001 \
    uv sync --extra gpu

# 9. Expose port and set entrypoint
# Change the RUN line to use the correct path:
RUN if [ "$DOWNLOAD_MODEL" = "true" ]; then \
    python ./download_model.py --output api/src/models/v1_0; \
    fi

ENV DEVICE="gpu"
# Run FastAPI server through entrypoint.sh
CMD ["./entrypoint.sh"]
