# ============================================================
# Dockerfile — Serviço WEB (Achados & Perdidos)
# ============================================================
# Imagem base: Python 3.12 slim (enxuta, sem pacotes extras)
FROM python:3.12-slim

# Evitar prompts interativos e forçar saída sem buffer
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Diretório de trabalho dentro do contêiner
WORKDIR /app

# 1. Copiar APENAS o requirements.txt primeiro (cache de camadas)
COPY requirements.txt .

# 2. Instalar dependências sem guardar cache do pip
RUN pip install --no-cache-dir -r requirements.txt

# 3. Criar diretório de uploads
RUN mkdir -p /app/uploads

# 4. Criar usuário sem privilégios e atribuir propriedade
RUN addgroup --system appgroup && \
    adduser --system --ingroup appgroup appuser && \
    chown -R appuser:appgroup /app/uploads

# 5. Copiar o restante do código da aplicação
COPY . .

# 6. Trocar para o usuário sem privilégios (RNF04)
USER appuser

# 7. Documentar a porta que a aplicação utiliza
EXPOSE 5000

# 8. Iniciar com Gunicorn em exec form (CMD com colchetes)
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
