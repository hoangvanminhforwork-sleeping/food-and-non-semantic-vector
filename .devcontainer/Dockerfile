FROM python:3.10-slim

# Install system dependencies including git and tree command
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    curl \
    git \
    tree \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /workspace

# Upgrade core Python packaging tools
RUN pip install --no-cache-dir --upgrade pip setuptools wheel

# Copy and install dependencies from requirements.txt
COPY requirements.txt /tmp/requirements.txt
RUN pip install --no-cache-dir -r /tmp/requirements.txt

EXPOSE 8888