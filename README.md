# Mangr

App financeiro para casais, com **modo Individual** e **modo Casal**. Lançamentos entram por texto, áudio, comprovante ou WhatsApp, a IA estrutura os dados e a pessoa confirma.

> Este repositório público contém **apenas a documentação de arquitetura**. O código-fonte é privado.

## Problema

Casais dividem despesas, cartões e metas, mas controlam tudo em planilhas ou conversas. O Mangr organiza renda, rateio, cartões, orçamento e prioridades, sem perder o histórico individual de cada pessoa quando o casal se vincula.

## Visão geral da arquitetura

```
 App Expo (iOS/Android)
   │  HTTPS (REST/JSON, cliente gerado do OpenAPI)
   ▼
 API Bun + Hono (monólito modular)
   ├─ auth (Better Auth) · households · transactions · cards · budgets
   ├─ priority-matrix · goals · investments · alerts
   └─ ingestion (receipt, text, whatsapp, openfinance)
   │
   ▼
 PostgreSQL ── pg-boss (filas) ── Workers (mesmo código, processo separado)
                                   ├─ ai-parse-receipt (imagem/PDF → IA com visão → JSON)
                                   ├─ ai-parse-text (texto ou áudio → JSON)
                                   └─ recorrências, faturas, alertas, push
```

## Stack e por que foi escolhida

| Camada | Escolha | Motivo |
|---|---|---|
| App | Expo (React Native) + TypeScript | Uma base para iOS e Android |
| Estado | TanStack Query + Zustand | Cache e sincronização simples |
| Runtime | Bun | TypeScript nativo, workspaces, test runner e start rápido |
| API | Hono + Zod OpenAPI | Feito para Bun, tipos de ponta a ponta. NestJS foi descartado pelo atrito em Bun |
| Banco | PostgreSQL + Drizzle ORM | Dados relacionais e integridade para dinheiro, migrations versionadas |
| Auth | Better Auth | Sessões, OAuth (Apple e Google) e proteção contra abuso prontos |
| Fila | pg-boss | Jobs em cima do próprio Postgres, sem Redis (uma peça a menos) |
| IA | Groq (visão, texto e Whisper) | Inferência rápida e barata, chamada só pelo backend |
| Infra | Docker + Oracle Cloud ARM + GitHub Actions | Começar simples, sem Kubernetes |

## Decisões de arquitetura

- **Dinheiro em centavos inteiros** (`bigint`), nunca float. Datas em UTC com fuso do household para fechamento de mês.
- **Individual → Casal sem migrar dados.** Todo usuário nasce com um *household* de um membro. Ao vincular o parceiro, os dados pessoais continuam `personal` e os novos podem ser `shared`. Nada é copiado.
- **A IA sugere, o usuário confirma.** Todo lançamento vindo de IA, WhatsApp ou Open Finance entra como `pending_review`.
- **Monólito modular** por domínio, com workers rodando o mesmo código em processo separado.
- **Soft delete e auditoria** em todas as tabelas.
- **Qualidade:** Biome, TypeScript estrito e testes com Postgres real no CI.

## Infraestrutura

VM ARM Ampere A1 (Always Free) na Oracle Cloud, provisionada por script e `cloud-init`, com Docker Compose (API, Postgres e Caddy). Veja [cloud-foundation](https://github.com/mathsarvas-dev/cloud-foundation).

## Autor

**Matheus Mosconi Sarvas** · [GitHub](https://github.com/mathsarvas-dev)
