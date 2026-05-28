# Squelette à compléter — étape 3.
# Mêmes contraintes que les autres services (multi-stage, non-root, ≤ 200 Mo).

# ---- Stage 1 : build / install des dépendances ----
FROM python:3.12-slim AS build

WORKDIR /app

# Création d'un environnement virtuel pour isoler les dépendances et éviter d'installer globalement
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY app/ ./app/

# ---- Stage 2 : runtime ----
FROM python:3.12-slim AS runtime

LABEL org.opencontainers.image.source="https://github.com/carro/tp-argocd"

WORKDIR /app

# Copie du venv et du code depuis le stage build
COPY --from=build /opt/venv /opt/venv
COPY --from=build /app/app ./app

# Création de l'utilisateur non-root 1001 pour la sécurité
RUN groupadd -g 1001 appgroup && useradd -r -u 1001 -g appgroup appuser
USER 1001

ENV PORT=8080 \
    LOG_LEVEL=info \
    PYTHONUNBUFFERED=1 \
    PATH="/opt/venv/bin:$PATH"

EXPOSE 8080

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080"]
