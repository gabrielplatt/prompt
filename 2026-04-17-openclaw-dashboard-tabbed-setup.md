# Prompt: NaumoStack Content Factory — Dashboard 5-Tab + Stack Setup

**Data:** 2026-04-17
**Autor:** Gabriel Platt
**Para:** Filipe (colaborador)
**Objetivo:** Configurar a stack OpenClaw completa no seu ambiente e entender o dashboard reestruturado em 5 abas

---

## O que foi feito

O dashboard da Content Factory (porta 8200) foi reestruturado de uma single page com scroll para uma interface com **5 abas**:

1. **Overview** — Stats gerais + visualizacao do pipeline (signals -> ideas -> drafts -> published -> rejected) + quick summary
2. **Kanban** — Board de atividades (NEW / TO DO / DONE) com filtro por agente e criacao de items
3. **Knowledge** — Metricas do second-brain (notes indexed, cross-connections, agent memories, opportunities)
4. **Agents** (NOVA) — Metricas por agente do LLM Router, tabela de todos os 16 cron jobs, timeline das ultimas 20 chamadas
5. **Content** — Review queue top-3, all candidates, themes, signals by topic

### Novo endpoint: `/api/agents/metrics`
Agrega dados de `workspace/logs/router-decisions.jsonl` + `cron/jobs.json`:
- Per-agent: calls, escalation rate, avg latency, tokens, cost, models usados, errors
- Totals: total calls, escalation rate, cost total, tokens total, unique models/agents
- Recent: ultimas 20 chamadas com timestamp, agente, modelo, latencia
- Cron jobs: todos os 16 jobs com schedule, modelo, status, duracao, errors

### Comportamento das abas
- Estado persiste via URL hash (`#overview`, `#kanban`, `#agents`, etc.)
- Cada aba faz fetch so dos seus dados (lazy load)
- Auto-refresh 30s apenas na aba ativa
- Modais (draft viewer + kanban detail) funcionam globalmente (position:fixed)
- Responsivo mobile (tabs scrollam horizontalmente)

---

## Arquitetura Completa da Stack (excluindo projetos pessoais)

### Componentes e Portas

| Componente | Porta | Descricao |
|-----------|-------|-----------|
| OpenClaw Gateway | ws://127.0.0.1:18789 | Hub central de roteamento (Telegram, WebUI, agents) |
| LLM Router Proxy | http://127.0.0.1:11435 | Roteamento inteligente multi-tier de modelos LLM |
| Ollama | http://127.0.0.1:11434 | Servidor local de modelos LLM |
| Dashboard API | http://127.0.0.1:8200 | Dashboard Content Factory (Python stdlib) |
| PostgreSQL | 192.168.2.100:5432 | Banco second-brain (pgvector) — InfraNaumo |

### Agentes Registrados

| Agente | Modelo | Funcao |
|--------|--------|--------|
| intel-agent | qwen3.5:9b | Scout tech trends, RSS feeds, Twitter |
| opportunity-agent | qwen3.5:27b | Valida oportunidades, gera drafts LinkedIn |
| content-agent | qwen3.5:9b | Formata multi-plataforma (LinkedIn -> Twitter/Newsletter) |
| knowledge-agent | Scripts Python | Indexa vault, semantic search, embeddings |

### LLM Router — Roteamento de Modelos

Modo: `hybrid` (local first, cloud fallback)

**Tiers de roteamento:**
- `fast`: chat, tool_use, summary → qwen3.5:27b (primary), qwen3.5:9b (fallback), qwen2.5:7b (last resort)
- `code`: code → devstral-small-2:24b (primary)
- `reasoning`: analysis, compliance → qwen3.5:27b ou cloud
- Input >16K tokens → skip direto para tier `reasoning`

**Budget cloud fallback:** $1/dia, $5/mes (Claude Sonnet via Anthropic API)

**Decision log:** `workspace/logs/router-decisions.jsonl`
```json
{"ts":"...", "agent":"...", "task_type":"chat", "tier_start":"fast", "tier_final":"fast", "model":"qwen3.5:27b", "escalated":false, "latency_ms":23745, "cost_usd":0, "input_tokens":370, "output_tokens":163}
```

### Modelos Ollama Instalados

| Modelo | Tamanho | Tier | Uso |
|--------|---------|------|-----|
| qwen3.5:27b | 17 GB | fast, reasoning | Primario chat/reasoning |
| qwen3.5:9b | 6.6 GB | fast | Fallback rapido |
| devstral-small-2:24b | 16 GB | code | Primario code |
| qwen2.5:32b | 19 GB | fast, reasoning | Legacy |
| qwen2.5:7b | 4.7 GB | fast | Ultimo recurso |
| nomic-embed-text | 274 MB | embeddings | Embeddings 768d (second-brain) |

### Pipeline de Conteudo

```
Signals (JSON) → Themes (clustering) → Ideas (LLM scoring) → Drafts (markdown)
     |                  |                      |                    |
     ▼                  ▼                      ▼                    ▼
  intel-agent    opportunity-agent      opportunity-agent     content-agent
  (RSS/feeds)    (validate+score)       (thesis+hooks)       (format+publish)
```

**Scoring formula:**
```
total = (fit * 0.25) + (confidence * 0.15) + (novelty * 0.15) + (business_value * 0.15) + (content_strength * 0.15) + (repetition * 0.10) + (strategic_value * 0.05)
```

### Second Brain (PostgreSQL + pgvector)

**Schema:** `second_brain` no banco `homelab`

| Tabela | Descricao |
|--------|-----------|
| knowledge_nodes | Notas do Obsidian vault (content, embedding 768d, frontmatter JSONB) |
| agent_memory | Memorias episodicas/semanticas por agente |
| cross_connections | Links semanticos entre notas |
| opportunities | Oportunidades validadas de negocio |

**Embedding:** nomic-embed-text via Ollama, 768 dimensoes, indice IVFFlat (cosine)

### Cron Jobs (16 jobs via Gateway)

Os jobs rodam via `cron/jobs.json` no Gateway. Cada job tem schedule, modelo, timeout, e delivery (Telegram ou silencioso).

Categorias:
- **Heartbeats:** morning, midday, close, evening (portfolio check)
- **Content:** intel-agent:daily-scan, opportunity-agent:process-signals, content-agent:format-approved
- **Maintenance:** log-rotation, session-cleanup, healthcheck
- **Weekly:** weekly-briefing, opportunity-agent:weekly-report

---

## Estrutura de Diretorios

```
~/.openclaw/
├── openclaw.json                         # Config principal (NÃO COMMITAR — tem tokens)
├── cron/jobs.json                        # Estado dos cron jobs
├── workspace/
│   ├── projects/
│   │   ├── opportunity-content-engine/   # Pipeline de conteudo
│   │   │   ��── scripts/dashboard_server.py  # Dashboard 5-tab (porta 8200)
│   │   │   ├── scripts/ingest_signals.py
│   │   │   ├── scripts/cluster_themes.py
│   │   │   ├── scripts/score_ideas.py
│   │   │   ├── scripts/generate_drafts.py
│   │   │   ├── config/operator.example.json  # Template — copiar para operator.json
│   │   │   ├── data/{signals,themes,ideas,drafts,queue,published,rejected}/
│   │   │   └── ARCHITECTURE.md
│   │   ├── second-brain/
│   │   │   ├── scripts/vault_indexer.py
│   │   │   ├── scripts/semantic_search.py
│   │   │   ├── scripts/morning_briefing.py
│   │   │   ├── scripts/telegram_capture.py
│   │   │   ├── init-db/01-schema.sql
│   │   │   ├���─ requirements.txt          # psycopg2, requests, PyYAML, watchdog
│   │   │   └── .env.example              # Template para credenciais DB
│   │   ├── llm-router/
│   │   │   ├── config.yaml               # Modelos, tiers, budget, validation
│   │   │   ├── server.py                 # Proxy server
│   │   │   └── requirements.txt          # PyYAML
│   │   ├── intel-watcher/
│   │   │   ├── scripts/                  # RSS analysis
│   │   │   ├── config.yaml               # Tech feeds
│   │   │   └── config_opportunity.yaml   # Monetization signals
│   │   └── content-engine/
│   │       └── scripts/                  # plan_week, validate_post, convert_formats
│   ├── logs/
│   │   └── router-decisions.jsonl        # Todas decisoes de roteamento (fonte da aba Agents)
│   └── CLAUDE.md
└── docs/
    ├── ARCHITECTURE.md                   # Documentacao completa (PT-BR)
    └── adr/                              # Architecture Decision Records
```

---

## Setup no seu ambiente

### Pre-requisitos
- macOS ou Linux
- Python 3.11+
- Node.js 22+
- Ollama instalado
- PostgreSQL 16+ com pgvector (pode ser Docker)

### Passo a passo

```bash
# 1. Clone os repos (se ainda nao fez)
# Acesso via Gitea no InfraNaumo ou GitHub

# 2. Instalar Ollama e modelos (ajustar ao seu hardware)
# IMPORTANTE: so baixe os modelos que seu hardware aguenta
ollama pull qwen3.5:9b          # 6.6 GB — minimo pra fast tier
ollama pull nomic-embed-text    # 274 MB — embeddings
# Opcional (se tiver >=32GB RAM):
# ollama pull qwen3.5:27b       # 17 GB
# ollama pull devstral-small-2:24b  # 16 GB

# 3. Configurar LLM Router
cd ~/.openclaw/workspace/projects/llm-router
cp config.yaml config.yaml.bak
# Editar config.yaml — ajustar modelos disponiveis ao seu hardware
# Remover modelos que voce nao baixou

# 4. Configurar Opportunity Content Engine
cd ~/.openclaw/workspace/projects/opportunity-content-engine
cp config/operator.example.json config/operator.json
# Editar operator.json com seu nome e positioning
# LLM endpoint: http://127.0.0.1:11435 (router) ou http://127.0.0.1:11434 (ollama direto)

# 5. Configurar Second Brain
cd ~/.openclaw/workspace/projects/second-brain
cp .env.example .env
# Editar .env — apontar para seu PostgreSQL
# Se usando InfraNaumo compartilhado: host=192.168.2.100, port=5432
# Se local: host=localhost
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt

# 6. Inicializar banco
psql -h <SEU_HOST> -U <SEU_USER> -d homelab -f init-db/01-schema.sql

# 7. Subir o dashboard
cd ~/.openclaw/workspace/projects/opportunity-content-engine
python3 scripts/dashboard_server.py --port 8200
# Acesse: http://127.0.0.1:8200
# Teste aba Agents: http://127.0.0.1:8200/#agents

# 8. Subir o LLM Router
cd ~/.openclaw/workspace/projects/llm-router
python3 server.py
# Porta: 11435
```

### O que ajustar ao seu hardware

- Se tiver <16GB RAM: usar so qwen3.5:9b, remover modelos maiores do config.yaml
- Se tiver >=32GB RAM: pode manter qwen3.5:27b como primario
- Se nao tiver GPU: os modelos rodam em CPU (mais lento, mas funciona)
- Se nao tiver PostgreSQL: a aba Knowledge fica vazia, resto funciona

### Verificacao

```bash
# Dashboard respondendo?
curl http://127.0.0.1:8200/api/health

# Agents metrics?
curl http://127.0.0.1:8200/api/agents/metrics | python3 -m json.tool | head -20

# Ollama rodando?
curl http://127.0.0.1:11434/api/tags

# LLM Router rodando?
curl http://127.0.0.1:11435/health
```

---

## Pontos de atencao

1. **NAO commitar:** `openclaw.json`, `.env`, `operator.json`, qualquer arquivo com tokens/API keys
2. **Sincronizar modelos:** Sempre que mudar modelos no Ollama, atualizar `llm-router/config.yaml`
3. **router-decisions.jsonl:** Esse arquivo cresce — a aba Agents le ele inteiro. Se passar de 10K linhas, considerar rotacao
4. **LaunchAgents:** No macOS, os servicos sao gerenciados por plist em `~/Library/LaunchAgents/ai.openclaw*.plist`. No Linux, traduzir para systemd ou crontab
5. **Vault path:** O Obsidian vault esta em `~/GabrielOS/vault/` — ajustar se seu vault estiver em outro lugar

---

## Endpoints da API (referencia rapida)

| Endpoint | Tab | Descricao |
|----------|-----|-----------|
| `/api/health` | — | Health check |
| `/api/content/queue` | Overview, Content | Fila semanal + top-3 |
| `/api/content/themes` | Overview, Content | Temas ativos |
| `/api/content/signals/stats` | Overview, Content | Stats de signals |
| `/api/content/drafts` | Content | Lista de drafts |
| `/api/content/drafts/<id>` | Content | Draft completo (markdown) |
| `/api/content/approve/<id>` | Content | Aprovar draft |
| `/api/content/reject/<id>` | Content | Rejeitar draft |
| `/api/factory/status` | Overview | Status do pipeline |
| `/api/factory/agents` | — | Status agentes (3 content agents) |
| `/api/factory/knowledge` | Knowledge | Health do second-brain |
| `/api/agents/metrics` | Agents | Metricas completas de todos agentes + cron jobs |
| `/api/kanban/items` | Kanban | GET lista, POST cria item |
| `/api/kanban/items/<id>` | Kanban | POST atualiza item |
| `/api/stack` | Header | Identidade da instancia |
