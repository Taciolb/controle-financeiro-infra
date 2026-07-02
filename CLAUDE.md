# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Visão Geral

Repositório de infraestrutura do sistema **Controle Financeiro** (TLB Tech). Contém apenas o `docker-compose.yml` que orquestra todos os microsserviços e seus bancos de dados. O código-fonte de cada serviço fica em repositórios irmãos na pasta pai.

## Comandos Principais

```bash
# Subir toda a infraestrutura
docker compose up -d

# Subir apenas os bancos de dados
docker compose up -d postgres-usuarios postgres-centrocusto postgres-lancamentos postgres-notificacao postgres-evolution

# Subir um serviço específico
docker compose up -d ms-usuarios

# Ver logs de um serviço
docker compose logs -f ms-usuarios

# Derrubar tudo
docker compose down

# Reconstruir imagem de um serviço após mudança de código
docker compose up -d --build ms-usuarios
```

## Configuração de Ambiente

Copie `.env.example` para `.env` e preencha os valores reais antes de subir os containers. O `.env` não é comitado (está no `.gitignore`).

Variáveis obrigatórias: `POSTGRES_USER`, `POSTGRES_PASSWORD`, `JWT_SECRET`, `JWT_EXPIRATION`, `INTERNAL_SECRET`, `SENDGRID_API_KEY`, `SENDGRID_FROM_EMAIL`, `EVOLUTION_API_KEY`, `EVOLUTION_INSTANCE`.

## Arquitetura

```
Internet → api-gateway:8080
               ├── ms-usuarios:8081       → postgres-usuarios:5433
               ├── ms-centro-custo:8082   → postgres-centrocusto:5434
               ├── ms-lancamentos:8083    → postgres-lancamentos:5435
               │       └── (chama ms-centro-custo internamente)
               ├── ms-fluxo-caixa:8084    → (chama ms-lancamentos internamente)
               └── bff-financeiro:8085    → (agrega ms-lancamentos, ms-centro-custo, ms-fluxo-caixa)

ms-notificacao:8086  → postgres-notificacao:5436
                     → evolution-api:8089 (WhatsApp)
                     → SendGrid (e-mail)

evolution-api:8089   → postgres-evolution:5437
```

**Padrões arquiteturais:**
- Database-per-service: cada microsserviço tem seu próprio PostgreSQL dedicado.
- Comunicação interna via HTTP usando URLs de serviço Docker (ex: `http://ms-centro-custo:8082`).
- Autenticação via JWT — todos os serviços recebem `JWT_SECRET` e `JWT_EXPIRATION`.
- Chamadas internas entre microsserviços usam `INTERNAL_SECRET` para autorização sem JWT de usuário.
- `api-gateway` é o único ponto de entrada externo (porta 8080).
- `bff-financeiro` agrega dados de múltiplos serviços para o frontend.
- `ms-notificacao` envia e-mails (SendGrid) e mensagens WhatsApp (Evolution API).

## Localização dos Repositórios

Os builds usam caminhos relativos ao diretório pai:

| Serviço         | Caminho do build |
|-----------------|-----------------|
| ms-usuarios     | `../ms-usuarios` |
| ms-centro-custo | `../ms-centro-custo` |
| ms-lancamentos  | `../ms-lancamentos` |
| ms-fluxo-caixa  | `../ms-fluxo-caixa` |
| bff-financeiro  | `../bff-financeiro` |
| ms-notificacao  | `../ms-notificacao` |
| api-gateway     | `../api-gateway` |

## Mapeamento de Portas

| Container         | Porta externa | Porta interna |
|-------------------|---------------|---------------|
| api-gateway       | 8080          | 8080          |
| ms-usuarios       | 8081          | 8081          |
| ms-centro-custo   | 8082          | 8082          |
| ms-lancamentos    | 8083          | 8083          |
| ms-fluxo-caixa    | 8084          | 8084          |
| bff-financeiro    | 8085          | 8085          |
| ms-notificacao    | 8086          | 8086          |
| evolution-api     | 8089          | 8080          |
| postgres-usuarios | 5433          | 5432          |
| postgres-centrocusto | 5434       | 5432          |
| postgres-lancamentos | 5435       | 5432          |
| postgres-notificacao | 5436       | 5432          |
| postgres-evolution   | 5437       | 5432          |