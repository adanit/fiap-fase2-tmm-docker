# Stage 1 - Build da Aplicação
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Stage 2 - Criar a imagem final
FROM python:3.12-slim
RUN adduser --disabled-password --gecos '' appuser
WORKDIR /app
COPY --from=builder /root/.local /home/appuser/.local
COPY . .
RUN chown -R appuser:appuser /app /home/appuser/.local
USER appuser
ENV PATH="/home/appuser/.local/bin:$PATH"
EXPOSE 8003
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8003/health').read()" || exit 1
CMD ["gunicorn", "--bind", "0.0.0.0:8003", "app:app"]