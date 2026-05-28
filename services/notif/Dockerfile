# Squelette — étape 3.
# Cible : image FINALE ≤ 50 Mo (Go produit un binaire statique, profitez-en).

# ---- Stage 1 : build du binaire Go ----
FROM golang:1.22-alpine AS build

WORKDIR /src

# Optimisation du cache docker pour les modules
COPY go.mod ./
RUN go mod download

# Copie du code et compilation statique (CGO_ENABLED=0, optimisé)
COPY cmd/ ./cmd/
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /out/notif ./cmd

# ---- Stage 2 : runtime minimal ----
FROM gcr.io/distroless/static:nonroot AS runtime

LABEL org.opencontainers.image.source="https://github.com/carro/tp-argocd"

# Copie du binaire compilé statiquement
COPY --from=build /out/notif /notif

ENV PORT=8080 \
    LOG_LEVEL=info

EXPOSE 8080

ENTRYPOINT ["/notif"]
