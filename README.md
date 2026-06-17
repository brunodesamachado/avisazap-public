# AvisaZap

![Em produção](https://img.shields.io/badge/status-em%20produção-brightgreen) ![Python 3.12](https://img.shields.io/badge/python-3.12-blue) ![Docker](https://img.shields.io/badge/docker-ready-blue) ![n8n](https://img.shields.io/badge/n8n-workflows-orange) ![PostgreSQL](https://img.shields.io/badge/postgresql-database-336791) ![Redis](https://img.shields.io/badge/redis-cache-red)

Sistema de monitoramento de ofertas em marketplaces com notificação automática via WhatsApp.

## Como funciona

```
Usuário (WhatsApp)
       │
       ▼
┌─────────────────┐
│  n8n Agente IA  │  ← Extrai entidades do pedido (produto, marca, preço máximo...)
│  (gpt-4o-mini)  │
└────────┬────────┘
         │ Salva alerta
         ▼
   ┌───────────┐
   │ PostgreSQL │  ← Tabela avisa_alerts (critérios de match por usuário)
   └─────┬─────┘
         │
         ▼
┌─────────────────┐
│  n8n Scheduler  │  ← Cron a cada hora → dispara POST /trigger por marketplace
└────────┬────────┘
         │
         ▼
┌──────────────────────────────────────────────────────┐
│                   Python Worker                       │
│                                                      │
│  Pollers (ML / Amazon / Shopee)                      │
│       │                                              │
│       ▼                                              │
│    Redis ──── Deduplicação de ofertas (TTL 12h)      │
│       │                                              │
│       ▼                                              │
│  Gemini 2.5 Flash ── Extração de atributos (lotes)   │
│       │                                              │
│       ▼                                              │
│  Match contra avisa_alerts                           │
│       │                                              │
│    Redis ──── Dedup de notificações por usuário      │
│       │                                              │
│       ▼                                              │
│  Response HTTP com alerts[]                          │
└────────┬─────────────────────────────────────────────┘
         │
         ▼
┌─────────────────┐
│  n8n Workflow   │  ← Formata e envia via Evolution API
│  (por marketplace) │   com delay incremental entre mensagens
└────────┬────────┘
         │
         ▼
  Usuário (WhatsApp)
```

## Arquitetura

### Parte 1 — Python Worker (Docker / Railway)

Servidor HTTP assíncrono (`aiohttp`) com endpoint `POST /trigger`. Executa o ciclo completo de mineração para um marketplace por chamada: coleta de ofertas → deduplicação Redis → extração de atributos via LLM em lotes → match com alertas cadastrados → anti-flood → retorno dos alertas no body da resposta HTTP.

**Tecnologias-chave:** `aiohttp`, `BeautifulSoup4`, `curl_cffi`, `asyncio`, `asyncpg`, `redis-py`, `OpenRouter (Gemini 2.5 Flash)`

### Parte 2 — n8n Workflows

Dois workflows independentes: **AgenteAvisaZap** (recebe mensagem do usuário via webhook, usa AI Agent para extrair entidades e cadastra no PostgreSQL) e **Workflow_Marketplace** (Schedule Trigger → HTTP POST ao Python Worker → loop sobre alerts[] → envio WhatsApp com delay configurável).

**Tecnologias-chave:** `n8n`, `Evolution API`, `OpenRouter (gpt-4o-mini)`, `PostgreSQL`, `LangChain AI Agent`

## Stack

| Tecnologia | Uso no projeto |
|---|---|
| Python 3.12 | Runtime do worker de mineração e match |
| aiohttp | Servidor HTTP `/trigger` + scraping do Mercado Livre |
| BeautifulSoup4 | Parse de HTML do Mercado Livre |
| curl_cffi | Scraping da Amazon com TLS fingerprint do Chrome |
| asyncio | Concorrência assíncrona no worker |
| PostgreSQL | Persistência de usuários, alertas e histórico de preços |
| Redis | Deduplicação de ofertas e notificações (TTL 12h) |
| Docker | Containerização do Python Worker |
| Railway | PaaS — deploy de todos os serviços |
| n8n | Orquestração de workflows, agendamento e envio WhatsApp |
| Evolution API | Gateway WhatsApp (envio/recebimento de mensagens) |
| OpenRouter | Roteamento de LLMs (Gemini para extração, GPT para agente) |
| Gemini 2.5 Flash | Extração técnica de atributos de produtos em lotes |
| LangChain (n8n) | AI Agent no workflow de cadastro de alertas |

## Decisões técnicas

- **`curl_cffi` para Amazon**: `aiohttp` e `requests` recebem 503 na página de deals da Amazon, que bloqueia por TLS fingerprint. `curl_cffi` impersona o handshake TLS do Chrome 120, contornando o bloqueio sem necessidade de browser headless.

- **n8n como orquestrador**: Agendamento (cron), AI Agent com tool use, formatação de mensagem e envio WhatsApp em nós visuais — sem reinventar infraestrutura de scheduling ou gestão de estado de conversação. O Python Worker foca exclusivamente em mineração e match.

- **HTTP trigger em vez de loop interno**: `POST /trigger` com lock de concorrência (`409 Busy`) permite controle externo de cada ciclo. O n8n dispara marketplaces de forma independente, com schedules próprios. Facilita testes pontuais com `{"force": true}` sem alterar código.

- **Fallback individual por item no LLM**: O extrator processa produtos em lotes de 10 para economizar tokens. Se o JSON de resposta do LLM for truncado (falha de lote), cada produto é reprocessado individualmente. Garante que uma falha parcial não descarte o lote inteiro.

## Pré-requisitos

| Serviço | Link |
|---|---|
| OpenRouter | https://openrouter.ai |
| Railway | https://railway.app |
| Amazon Associates | https://associados.amazon.com.br |
| Mercado Livre Afiliados | https://afiliados.mercadolivre.com.br |
| Shopee Affiliate Open API | https://affiliate.shopee.com.br/open_api |
| Evolution API | https://evolution-api.com |

## Como rodar

1. Copie `.env.example` para `.env` e preencha com suas credenciais
2. Veja `docs/architecture.md` para detalhes completos da arquitetura e deploy no Railway

## Workflows n8n

Os workflows estão em `n8n/workflows/`. Veja `docs/n8n-setup.md` para o guia completo de importação e configuração.

## Status

Em produção — monitorando Mercado Livre, Amazon e Shopee.
