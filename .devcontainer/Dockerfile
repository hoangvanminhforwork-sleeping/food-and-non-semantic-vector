FROM python:3.10-slim

# Cài đặt git và các công cụ hệ thống không lưu cache rác
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    curl \
    git \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /workspace

# Nâng cấp pip và cài đặt thư viện Python
RUN pip install --no-cache-dir --upgrade pip setuptools wheel

COPY .devcontainer/requirements.txt .
RUN pip install -v --no-cache-dir -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple

EXPOSE 8888

CMD ["jupyter", "lab", "--ip=0.0.0.0", "--port=8888", "--no-browser", "--allow-root", "--NotebookApp.token=''"]