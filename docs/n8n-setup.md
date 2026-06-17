# Configuração dos Workflows n8n

## Pré-requisitos

- Instância n8n rodando (Railway recomendado, com `DB_TYPE=postgresdb` para persistência)
- PostgreSQL acessível com as tabelas do AvisaZap criadas
- Conta Evolution API configurada com instância WhatsApp conectada
- API key do OpenRouter com acesso a `gpt-4o-mini` e `google/gemini-2.5-flash-lite`

---

## Credenciais necessárias no n8n

Configure estas credenciais em **Settings → Credentials → Add Credential** antes de importar os workflows:

| Nome da credencial | Tipo | Dados necessários |
|---|---|---|
| `Postgres account` | PostgreSQL | Host, porta, usuário, senha e nome do banco |
| `EvoGo account` | HTTP Header Auth | API key da Evolution API (header `apikey`) |
| `OpenRouter` | OpenAI-compatible | API key e base URL `https://openrouter.ai/api/v1` |

---

## Como importar os workflows

1. Acesse sua instância n8n
2. No menu lateral, vá em **Workflows**
3. Clique no botão **⋯** (menu) → **Import from File**
4. Selecione o arquivo JSON da pasta `n8n/workflows/`
5. Revise os nós com alerta de credencial faltando e associe as credenciais criadas acima
6. Salve e ative o workflow

---

## Configuração do AgenteAvisaZap

Este workflow recebe mensagens do WhatsApp, extrai entidades via AI Agent e cadastra alertas no banco.

**Passos:**

1. **Nó Webhook**: configure a URL pública do seu n8n. O path pode ser qualquer string — copie a URL gerada e use no próximo passo.

2. **Evolution API → Webhook**: No painel da sua instância Evolution API, configure o webhook de eventos de mensagem para apontar para:
   ```
   https://SEU_N8N_URL/webhook/SEU_PATH
   ```

3. **Nós Postgres**: substitua a credencial `Postgres account` em todos os nós de banco de dados.

4. **Nós Evolution API**: substitua a credencial `EvoGo account` nos nós de envio de mensagem.

5. **AI Agent**: o nó OpenRouter Chat Model usa `gpt-4o-mini` por padrão. Para trocar o modelo, edite o nó e altere o campo **Model**.

6. **Teste**: envie uma mensagem para o número conectado na Evolution API. O agente deve responder e cadastrar o alerta no banco.

---

## Configuração do Workflow_Marketplace

Este workflow é o template de monitoramento. Deve ser duplicado uma vez por marketplace (mercadolivre, amazon, shopee).

**Passos:**

1. **Nó HTTP Request** — ajuste a URL para o endereço do seu Python Worker:
   - Railway (rede interna): `http://python-worker.railway.internal:8000/trigger`
   - Railway (URL pública, para testes): `https://python-worker-XXXX.up.railway.app/trigger`
   - Local: `http://localhost:8000/trigger`

2. **Body do HTTP Request** — altere o campo `marketplace` para o marketplace desejado:
   ```json
   {"marketplace": "amazon"}
   ```
   Valores aceitos: `"mercadolivre"`, `"amazon"`, `"shopee"`.

3. **Schedule Trigger** — o cron padrão é `0 20 8-23 * * *` (a cada hora, no minuto 20, entre 8h e 23h). Ajuste conforme necessário.

4. **Nós Evolution API** — substitua a credencial `EvoGo account` nos nós de envio.

5. **Duplicar para outros marketplaces**: duplique este workflow e altere apenas o campo `marketplace` no body do HTTP Request. Cada marketplace tem seu próprio Schedule Trigger independente.

**Para forçar um ciclo completo ignorando o cache Redis (testes):**
```json
{"marketplace": "amazon", "force": true}
```

**Respostas possíveis do worker:**
| Código | Significado |
|---|---|
| `200` | Ciclo concluído — `alerts[]` no body (pode ser array vazio) |
| `400` | Marketplace inválido |
| `409` | Ciclo já em execução — ignorar e aguardar |
| `500` | Erro interno — verificar logs do worker |

---

## Tabelas PostgreSQL necessárias

O schema completo está no repositório privado. O contrato esperado pelas queries do n8n:

**`avisa_users`**
```
whatsapp_id  VARCHAR  — identificador único do usuário (número WhatsApp)
name         VARCHAR  — nome do usuário
status       VARCHAR  — estado da conversa (ex: "menu", "aguardando_confirmacao")
is_active    BOOLEAN  — se o usuário está ativo
updated_at   TIMESTAMPTZ
```

**`avisa_alerts`**
```
id            SERIAL PK
whatsapp_id   VARCHAR   — FK para avisa_users
product_name  VARCHAR   — categoria do produto em minúsculo (mandatório para match)
brand         VARCHAR   — marca (NULL quando não informado — nunca string "null")
model         VARCHAR   — modelo (NULL quando não informado)
color         VARCHAR   — cor (NULL quando não informado)
features      TEXT[]    — specs técnicas como array (omitir campo quando vazio)
max_price     DECIMAL   — preço máximo (NULL quando não informado)
is_active     BOOLEAN
created_at    TIMESTAMPTZ
```

> **Atenção**: campos opcionais devem ser `NULL` SQL, **nunca** a string `"null"`. O AI Agent está instruído a omitir campos sem valor, mas verifique os dados inseridos nas primeiras utilizações.

**`avisa_price_history`**
```
id           SERIAL PK
recorded_at  TIMESTAMP
category     VARCHAR
full_title   TEXT
brand        VARCHAR
model        VARCHAR
color        VARCHAR
features     TEXT[]
price        DECIMAL
old_price    DECIMAL
source_group VARCHAR   — ex: "Amazon Ofertas", "Mercado Livre Ofertas", "Shopee Ofertas"
```

> Os índices em `(brand, model)` e `recorded_at` são recomendados para consultas futuras de histórico de preço.
