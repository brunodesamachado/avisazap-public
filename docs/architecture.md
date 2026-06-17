# Arquitetura — AvisaZap

## Diagrama de fluxo completo

```mermaid
flowchart TD
    A[Usuário envia mensagem\nno WhatsApp] --> B[Evolution API\nrecebe e repassa]
    B --> C[n8n — AgenteAvisaZap\nWebhook]
    C --> D[AI Agent\ngpt-4o-mini via OpenRouter\nExtrai entidades do pedido]
    D --> E[(PostgreSQL\navisa_alerts)]

    F[n8n — Schedule Trigger\nCron a cada hora por marketplace] --> G[HTTP POST /trigger\npython-worker.railway.internal:8000]

    G --> H{Python Worker}

    H --> I[Poller do marketplace\nColeta ofertas]
    I --> J{Redis\nDedup de oferta\nTTL 12h}
    J -->|Nova oferta| K[Gemini 2.5 Flash\nvia OpenRouter\nExtrai atributos em lotes de 10]
    J -->|Duplicata| X1[Descarta]
    K -->|Falha de lote| K2[Fallback individual\npor produto]
    K --> L[Match contra avisa_alerts\nno PostgreSQL]
    K2 --> L
    L -->|Match| M[avisa_price_history\nRegistra preço]
    L -->|Sem match| X2[Descarta]
    M --> N{Anti-flood\nMais barato por alerta\nDelay incremental 60s}
    N --> O{Redis\nDedup de notificação\npor usuário TTL 12h}
    O -->|Já notificado| X3[Descarta ANTI-DUP]
    O -->|Novo| P[Inclui em alerts[]\ncom send_delay_seconds]
    P --> Q[Response HTTP 200\nalerts array]

    Q --> R[n8n — Loop sobre alerts]
    R --> S[Wait send_delay_seconds]
    S --> T[Evolution API\nEnvia imagem + texto\nno WhatsApp]
    T --> A2[Usuário recebe\na notificação]
```

---

## Pollers por marketplace

| Marketplace | Estratégia | Volume/ciclo | Detalhe técnico |
|---|---|---|---|
| Mercado Livre | HTML scraping (`aiohttp` + `BeautifulSoup4`) | ~30 produtos | GET único na página `/ofertas`; extrai título, preço, preço antigo, link, imagem, `free_shipping`, `is_full`, `installments` |
| Amazon | SSR JSON (`curl_cffi` Chrome fingerprint) | ~30 produtos | GET em `/deals`; extrai `productSearchResponse` embutido no HTML via `window.Globals`; retry 3x com backoff (0s, 10s, 30s); produtos sem `priceToPay` descartados; paginação não implementada (SSR ignora `startIndex`) |
| Shopee | API GraphQL oficial de afiliados | ~150 produtos | `POST` em `open-api.affiliate.shopee.com.br/graphql`; autenticação SHA256; 3 páginas × 50 produtos; filtro `priceDiscountRate >= 10` (≥10% desconto); `offerLink` já contém tracking de afiliado |

---

## Deduplicação em camadas

O sistema aplica três camadas independentes para evitar spam e reprocessamento desnecessário:

**Camada 1 — Deduplicação de oferta (Redis TTL 12h)**
Antes de enviar ao pipeline LLM, cada oferta é verificada no Redis pela sua URL normalizada (`{marketplace_key}:{link}`). Para Amazon, a chave usa o ASIN (`https://www.amazon.com.br/dp/{asin}`) para estabilidade — o SSR retorna os mesmos 30 produtos por ciclo, então após o primeiro ciclo completo todos ficam cacheados por 12h. Ciclos seguintes processam apenas produtos genuinamente novos ou com cache expirado.

**Camada 2 — Deduplicação de notificação por usuário (Redis TTL 12h)**
Após o match, verifica `notified:{whatsapp_id}:{sha256(product_name)}`. Se o usuário já foi notificado sobre aquele alerta dentro do `REDIS_NOTIF_TTL` (padrão 43200s), o alerta é descartado e logado como `[ANTI-DUP]`. Contatos em `REDIS_BYPASS_CONTACTS` ignoram essa verificação (`[BYPASS]`).

**Camada 3 — Anti-flood por ciclo**
Antes do Redis, aplica duas regras sobre o conjunto de matches do ciclo: (1) mesmo usuário + mesmo alerta com múltiplas ofertas → envia apenas a mais barata; (2) mesmo usuário com alertas de produtos diferentes → `send_delay_seconds` incremental de 60s entre cada mensagem para evitar flood.

O parâmetro `force: true` no body do `/trigger` ignora as camadas 1 e 2 globalmente. Útil para testes sem precisar limpar o Redis.

---

## AI Agent (n8n)

O workflow **AgenteAvisaZap** usa um AI Agent do n8n com `gpt-4o-mini` via OpenRouter para processar mensagens livres do usuário no WhatsApp.

Quando o usuário envia *"Me avisa quando sair notebook da Acer abaixo de R$ 3.000"*, o agente extrai entidades estruturadas:
- `product_name`: categoria genérica em português minúsculo (ex: `notebook`)
- `brand`: `acer`
- `max_price`: `3000`
- `model`, `color`, `features`: omitidos quando não informados

O agente usa um nó Postgres do n8n como ferramenta para inserir diretamente em `avisa_alerts`. O fluxo cobre também listagem de alertas, exclusão com confirmação e menu dinâmico baseado no estado do usuário.

**Bug conhecido e correção aplicada:** O nó Postgres com modo "Defined automatically by the model" serializava campos opcionais ausentes como a string `"null"` em vez de `NULL` SQL, quebrando o match (`ILIKE '%null%'` nunca encontra correspondência). Solução: descrição explícita em cada campo opcional no nó Postgres e regra no System Message do agente — campos sem valor devem ser omitidos completamente, nunca enviados como string `"null"`. O campo `features` (tipo `_text` / array) requer atenção especial: string vazia `''` causa erro de tipo no PostgreSQL — o campo deve ser omitido inteiramente quando não informado.

---

## Infraestrutura Railway

O projeto roda no Railway com quatro serviços independentes:

| Serviço | Tipo | Função |
|---|---|---|
| `python-worker` | Dockerfile | Core de mineração, extração LLM e match. Porta `8000`. |
| `postgres` | Plugin Railway | Persistência de `avisa_users`, `avisa_alerts` e `avisa_price_history`. `DATABASE_URL` injetada automaticamente. |
| `redis` | Plugin Railway | Cache de deduplicação de ofertas e notificações. `REDIS_URL` injetada automaticamente. |
| `n8n` | Serviço Docker (`n8nio/n8n`) | Agendamento, AI Agent e envio WhatsApp. Porta `5678`. Requer `DB_TYPE=postgresdb` para persistência — sem isso perde workflows a cada redeploy. |

**Comunicação interna:** Serviços usam a rede privada do Railway via hostnames `.railway.internal`. O n8n aciona o worker em `http://python-worker.railway.internal:8000/trigger`. URL pública exposta apenas para webhooks externos (Evolution API → AgenteAvisaZap).

**Deploy:** Configurado via `railway.toml` com `dockerfilePath = "src/Dockerfile"` e política de restart `ON_FAILURE` com 3 tentativas.
