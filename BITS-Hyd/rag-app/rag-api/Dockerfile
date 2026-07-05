FROM python:3.12-slim

WORKDIR /app

# sentence-transformers pulls in torch (CPU build is fine here since
# this container only does embeddings, not LLM inference)
RUN pip install --no-cache-dir --extra-index-url https://download.pytorch.org/whl/cpu torch

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
