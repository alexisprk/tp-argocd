# Squelette à compléter — étape 3 du TP.
# Contraintes (non négociables, lisez le polycopié) :
#   - multi-stage,
#   - image finale ≤ 200 Mo,
#   - utilisateur non-root,
#   - pas de secret en ENV/ARG,
#   - LABEL org.opencontainers.image.source pointant vers le dépôt Git.

# ---- Stage 1 : build / install des dépendances ----
FROM node:20-alpine AS build

WORKDIR /app

# Optimisation du cache Docker en copiant d'abord package.json
COPY package*.json ./
RUN npm install --no-audit --no-fund

# Copie du reste du code source
COPY src/ ./src/

# ---- Stage 2 : image runtime ----
FROM node:20-alpine AS runtime

LABEL org.opencontainers.image.source="https://github.com/carro/tp-argocd"

WORKDIR /app

# Copie des éléments nécessaires au runtime depuis le stage build
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/src ./src
COPY --from=build /app/package.json ./package.json

# Création d'un utilisateur non-root avec UID 1001 pour la sécurité
RUN addgroup -g 1001 appgroup && adduser -u 1001 -G appgroup -D appuser
USER 1001

ENV PORT=8080 \
    LOG_LEVEL=info \
    NODE_ENV=production

EXPOSE 8080

CMD ["node", "src/index.js"]
