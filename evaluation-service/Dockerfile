
# Stage 1 - Builder
FROM golang:1.21-alpine AS builder

# Dependências de sistema necessárias para compilação
RUN apk add --no-cache git

# Permite que o Go atualize/crie go.sum automaticamente durante o build
ENV GOFLAGS=-mod=mod

WORKDIR /app

# Copia os arquivos de módulo primeiro para aproveitar o cache de camadas
COPY go.mod ./

# Baixa as dependências
RUN go mod download

# Copia o restante do código-fonte
COPY *.go ./

RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-s -w" \
    -trimpath \
    -o evaluation-service .


# Stage 2 - Runtime

FROM alpine:3.19

# Certificados TLS necessários para conexões com PostgreSQL e serviços externos
RUN apk add --no-cache ca-certificates tzdata

# Executa como usuário não-root por segurança
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

WORKDIR /app

# Copia apenas o binário compilado do estágio anterior
COPY --from=builder /app/evaluation-service .

EXPOSE 8004

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD wget -qO- http://localhost:8004/health || exit 1

ENTRYPOINT ["./evaluation-service"]
