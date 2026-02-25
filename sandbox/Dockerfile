FROM python:3.10-slim

# System deps for CarveMe + MEMOTE
RUN apt-get update && apt-get install -y \
    gcc g++ wget curl git \
    glpk-utils libglpk-dev \
    libexpat1-dev \
    diamond-aligner \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY mcp_server.py .
# NOTE: SKILLS.md is NOT copied in — it is bind-mounted at runtime via
# docker-compose. This means you can edit SKILLS.md without rebuilding.

EXPOSE 8080
CMD ["python", "mcp_server.py"]
