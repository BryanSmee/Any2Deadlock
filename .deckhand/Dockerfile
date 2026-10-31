# .deckhand/Dockerfile: uv for a Python project (the base image has python3 3.11, no pip)
ARG DECKHAND_BASE=deckhand-sandbox
FROM ghcr.io/astral-sh/uv:0.13.6 AS uv
FROM $DECKHAND_BASE
USER root
COPY --from=uv /uv /uvx /usr/local/bin/
USER node
# The uv cache is a separate volume from /workspace, so hardlinks fail: copy instead.
ENV UV_LINK_MODE=copy
