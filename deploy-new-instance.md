# Prompt: Deploy NaumoStack (OpenClaw) em Nova Instância

> **Uso:** Cole este prompt em uma nova sessão do Claude Code (ou OpenClaw) no host de destino.
> **Pré-requisito:** O operador já clonou os repos e tem acesso ao hardware/infra listados abaixo.

---

## CONTEXTO

Você vai implantar o **NaumoStack** — um sistema de agentes colaborativos para coleta de inteligência, análise de oportunidades e produção de conteúdo para redes sociais. O sistema é composto por 6 agentes + 1 dashboard + 1 roteador de LLMs + bibliotecas compartilhadas.

A arquitetura segue o modelo **Team (não pipeline linear)**: Scout coleta → Analysts analisam → Content produz → Human aprova → (futuro) Social publica. Memória coletiva em 5 camadas (Episodic, Semantic, Procedural, Social, Proposals).

**Este prompt NÃO inclui o agente `capital-copilot`** — ele é pessoal e isolado.

---

## REPOSITÓRIOS PARA CLONAR

Clone todos na estrutura `~/.openclaw/workspace/`:

```bash
mkdir -p ~/.openclaw/workspace/{projects,lib,scripts,memory,executions,logs,handoffs}
cd ~/.openclaw/workspace/projects

# Agentes
git clone https://github.com/gabrielplatt/scout.git
git clone https://github.com/gabrielplatt/intel-watcher.git
git clone https://github.com/fmnicolay/opportunity-content-engine.git  # branch gabriel/macnaumo-onboarding
git clone https://github.com/gabrielplatt/content-engine.git
git clone https://github.com/gabrielplatt/second-brain.git
git clone https://github.com/gabrielplatt/llm-router.git
git clone https://github.com/gabrielplatt/dashboard.git
git clone https://github.com/gabrielplatt/agent-proposals.git

# Bibliotecas compartilhadas
cd ~/.openclaw/workspace
git clone https://github.com/gabrielplatt/openclawlib.git lib

# Skills
git clone https://github.com/gabrielplatt/openclaw-skills.git skills
```

**Branch:** `main` em todos (exceto opportunity que usa `gabriel/macnaumo-onboarding`).

---

## REQUISITOS DE INFRAESTRUTURA

### Hardware Mínimo
- CPU: 8+ cores (Apple Silicon M-series recomendado)
- RAM: 32 GB mínimo (48 GB recomendado para modelos 27B+)
- Disco: 100 GB livres (modelos + cache + vault)
- GPU: Apple Metal ou NVIDIA com 16+ GB VRAM

### Serviços Obrigatórios

| Serviço | Versão | Porta | Finalidade |
|---------|--------|-------|------------|
| **PostgreSQL** | 15+ | 5432 | Knowledge graph, episodic memory, analytics |
| **pgvector** | 0.7+ | (extensão PG) | Embeddings semânticos |
| **Redis** | 7+ | 6379 | Cache de queries, sessions |
| **Ollama** | 0.5+ | 11434 | Inferência LLM local |

### Modelos Ollama Necessários

```bash
ollama pull qwen2.5:14b      # Fast tier (~9 GB) — triage, summary, heartbeat
ollama pull qwen2.5:32b      # Fast/Reasoning (~19 GB) — scoring, analysis
ollama pull qwen3.5:27b      # Reasoning (~17 GB) — critique, deep analysis
ollama pull devstral-small-2:24b  # Code (~16 GB) — code generation
ollama pull nomic-embed-text  # Embeddings (~274 MB) — knowledge indexing
```

**NOTA:** Com 48 GB RAM, máximo 2 modelos carregados simultaneamente. Com 32 GB, use apenas 1.

### Serviços Opcionais
- **Telegram Bot** — Para notificações e approval flow
- **Obsidian + REST API Plugin** — Para captura via Telegram e visualização
- **GitHub Token** — Para Scout coletar GitHub Releases

---

## ESTRUTURA DE DIRETÓRIOS ESPERADA

```
~/.openclaw/
├── stack.json                          # ← CRIAR (identidade da instância)
├── openclaw.json                       # ← CRIAR (config do CLI/gateway)
├── lib/
│   └── stack_identity.py               # ← Já vem do repo openclawlib
├── workspace/
│   ├── CLAUDE.md                       # Regras do workspace
│   ├── TEAM.md                         # Contratos inter-agentes
│   ├── projects/
│   │   ├── scout/                      # Collector (CPU-only, sem LLM)
│   │   ├── intel-watcher/              # Technology analyst
│   │   ├── opportunity-content-engine/ # Market analyst
│   │   ├── content-engine/             # Content producer
│   │   ├── second-brain/              # Knowledge service
│   │   ├── llm-router/               # Model routing proxy
│   │   ├── dashboard/                 # Web dashboard
│   │   └── agent-proposals/           # Proposal queue
│   ├── lib/openclaw_commons/           # Shared library (15 modules)
│   ├── scripts/                        # Utility scripts
│   ├── memory/
│   │   ├── episodic/{agent}/*.jsonl    # L1 — per-run logs
│   │   ├── procedural/                # L3 — lessons learned
│   │   └── social/                    # L4 — signal scorecard
│   ├── handoffs/                       # Cross-agent data exchange
│   ├── executions/                     # Dated output folders
│   └── logs/                           # Centralized logs
├── cron/
│   └── jobs.json                       # ← CRIAR (scheduled jobs)
├── models/gguf/                        # (futuro: llama.cpp models)
└── docs/
    ├── adr/                            # Architecture Decision Records
    ├── schemas/                        # JSON schemas dos contratos
    └── ARCHITECTURE.md
```

---

## CONFIGURAÇÃO: PASSO A PASSO

### 1. Criar `~/.openclaw/stack.json`

Este é o **source of truth** para toda a instância. Todos os projetos leem dele via `stack_identity.py`.

```json
{
  "instance_id": "SEU-INSTANCE-ID",
  "operator": {
    "name": "Seu Nome",
    "handle": "seu_handle_social",
    "positioning": "Breve descrição do seu posicionamento estratégico",
    "languages": ["en"],
    "default_language": "en",
    "platforms": {
      "linkedin": { "enabled": true, "handle": "@seu_linkedin" },
      "twitter": { "enabled": true, "handle": "@seu_twitter" }
    }
  },
  "infra": {
    "hostname": "SEU-HOSTNAME",
    "vault_path": "~/path/to/obsidian/vault",
    "openclaw_home": "~/.openclaw",
    "workspace": "~/.openclaw/workspace",
    "postgres": {
      "host": "SEU-PG-HOST",
      "port": 5432,
      "database": "SEU-DB",
      "user": "SEU-USER",
      "schema": "second_brain"
    },
    "redis": {
      "host": "SEU-REDIS-HOST",
      "port": 6379,
      "db": 3
    },
    "ollama": {
      "url": "http://127.0.0.1:11434"
    },
    "llm_router": {
      "url": "http://127.0.0.1:11435"
    },
    "gateway": {
      "port": 18789
    },
    "dashboard": {
      "port": 8200
    }
  },
  "content": {
    "default_language": "en",
    "default_fit_score": 8,
    "fallback_fit_score": 6,
    "hashtags": ["#AI", "#LearningInPublic"],
    "red_flags": []
  },
  "agents": {
    "defaults": {
      "workspace": "~/.openclaw/workspace"
    },
    "intel": {
      "model": "qwen2.5:32b",
      "vault_folder": "03-RESOURCES/AI-Intel"
    },
    "opportunity": {
      "model": "qwen2.5:32b",
      "vault_folder": "03-RESOURCES/Opportunities"
    },
    "content": {
      "writer_model": "qwen2.5:32b",
      "critic_model": "qwen3.5:27b"
    }
  }
}
```

### 2. Criar `.env` para cada projeto

#### `projects/second-brain/.env`
```bash
INSTANCE_NAME=SEU-INSTANCE-ID
POSTGRES_PASSWORD=SUA-SENHA-PG
POSTGRES_DSN=postgresql://USER:PASSWORD@HOST:5432/DATABASE
REDIS_URL=redis://:REDIS-PASSWORD@HOST:6379
REDIS_HOST=SEU-REDIS-HOST
REDIS_PORT=6379
REDIS_PASSWORD=SUA-SENHA-REDIS
REDIS_DB=3
VAULT_PATH=~/path/to/vault
OBSIDIAN_API_URL=https://127.0.0.1:27124
OBSIDIAN_API_KEY=SEU-OBSIDIAN-KEY
OLLAMA_URL=http://localhost:11434
OLLAMA_MODEL=qwen2.5:32b
OLLAMA_MAX_LOADED_MODELS=2
OLLAMA_NUM_PARALLEL=1
```

#### `projects/intel-watcher/.env`
```bash
TELEGRAM_BOT_TOKEN=SEU-BOT-TOKEN
TELEGRAM_CHAT_ID=SEU-CHAT-ID
PGPASSWORD=SUA-SENHA-PG
```

#### `projects/opportunity-content-engine/.env`
```bash
TELEGRAM_BOT_TOKEN=SEU-BOT-TOKEN
TELEGRAM_CHAT_ID=SEU-CHAT-ID
PGPASSWORD=SUA-SENHA-PG
```

### 3. Criar virtual environments

```bash
cd ~/.openclaw/workspace/projects

for proj in scout intel-watcher opportunity-content-engine content-engine second-brain; do
  echo "=== Setting up $proj ==="
  cd $proj
  python3 -m venv .venv
  source .venv/bin/activate
  if [ -f requirements.txt ]; then
    pip install -r requirements.txt
  fi
  deactivate
  cd ..
done
```

### 4. Inicializar PostgreSQL

```sql
-- Criar database e schema
CREATE DATABASE seu_database;
\c seu_database
CREATE EXTENSION IF NOT EXISTS vector;
CREATE SCHEMA IF NOT EXISTS second_brain;

-- Tabelas são criadas automaticamente pelos scripts na primeira execução
-- OU executar o schema do second-brain se existir:
-- \i projects/second-brain/schema.sql
```

### 5. Configurar Ollama

```bash
# Variáveis de ambiente recomendadas (adicionar ao shell profile)
export OLLAMA_MAX_LOADED_MODELS=1        # 2 se tiver 48GB+
export OLLAMA_KEEP_ALIVE=-1              # Nunca descarrega
export OLLAMA_FLASH_ATTENTION=1          # Otimização de atenção
export OLLAMA_KV_CACHE_TYPE=q8_0         # Cache quantizado
```

### 6. Iniciar LLM-Router

```bash
cd ~/.openclaw/workspace/projects/llm-router
python3 server.py  # Porta 11435
```

### 7. Iniciar Dashboard

```bash
cd ~/.openclaw/workspace/projects/dashboard
python3 scripts/dashboard_server_v2.py --port 8200
```

---

## PROBLEMAS CONHECIDOS DE PORTABILIDADE

### Paths Hardcoded (ARQ-1 Violations)

Os seguintes arquivos contêm caminhos absolutos `/Users/macnaumo` que **DEVEM** ser substituídos:

| Arquivo | Linhas | O que substituir |
|---------|--------|------------------|
| `scout/config.yaml` | 6, 12, 152 | `output.root`, `cache.dir`, `pirs.dir` |
| `intel-watcher/config.yaml` | 2, 9 | `vault_path`, `scout_source.raw_pool` |
| `intel-watcher/config_opportunity.yaml` | similar | idem |
| `intel-watcher/scripts/daily_scan.sh` | 2 | `cd` path |
| `intel-watcher/run_intel.sh` | 2 | `cd` path |
| `cron/jobs.json` | ~11 linhas | Todos os paths absolutos |
| `scripts/activate_v2_cron.sh` | 16, 30 | workspace paths |
| `scripts/setup_vault_audit_cron.sh` | 4, 5, 8 | workspace + homebrew paths |

**Solução recomendada:**
```bash
# Substituição em massa (adaptar para seu username/home)
find ~/.openclaw -name "*.yaml" -o -name "*.json" -o -name "*.sh" | \
  xargs sed -i '' 's|/Users/macnaumo|'$HOME'|g'
```

### IP Hardcoded `192.168.2.100`

Presente em `.env` files. Substituir pelo IP do seu servidor PostgreSQL/Redis.

### Dependência de `/opt/homebrew/bin`

Alguns shell scripts referenciam paths do Homebrew (macOS). Em Linux, usar paths do sistema (`/usr/bin/python3`, etc.).

---

## ORDEM DE VALIDAÇÃO (Smoke Tests)

Execute na ordem — cada passo valida o anterior:

```bash
# 1. Stack Identity
python3 -c "
import sys; sys.path.insert(0, '$HOME/.openclaw/lib')
from stack_identity import STACK
print(f'Instance: {STACK.instance_id}')
print(f'Operator: {STACK.operator_name}')
print(f'Vault: {STACK.vault_path}')
print(f'PG: {STACK.pg_host}:{STACK.pg_port}/{STACK.pg_database}')
print(f'Ollama: {STACK.ollama_url}')
"

# 2. PostgreSQL
psql "$POSTGRES_DSN" -c "SELECT 1;"
psql "$POSTGRES_DSN" -c "CREATE EXTENSION IF NOT EXISTS vector;"

# 3. Redis
redis-cli -h SEU-REDIS-HOST -a SUA-SENHA ping

# 4. Ollama
curl -s http://127.0.0.1:11434/api/tags | python3 -m json.tool

# 5. Scout (CPU-only, sem deps externas pesadas)
cd ~/.openclaw/workspace/projects/scout
.venv/bin/python3 run.py --dry-run --max-per-feed 1

# 6. LLM-Router
cd ~/.openclaw/workspace/projects/llm-router
python3 server.py &  # background
sleep 3
curl -s http://127.0.0.1:11435/health | python3 -m json.tool

# 7. Intel-Watcher (requer Ollama + Scout raw_pool)
cd ~/.openclaw/workspace/projects/intel-watcher
.venv/bin/python3 run.py --dry-run

# 8. Second-Brain (requer PostgreSQL + Ollama embeddings)
cd ~/.openclaw/workspace/projects/second-brain
.venv/bin/python3 scripts/vault_indexer.py --index-all --with-chunks

# 9. Content-Engine (requer LLM-Router + handoffs)
cd ~/.openclaw/workspace/projects/content-engine
.venv/bin/python3 scripts/produce_draft.py --dry-run

# 10. Dashboard (read-only, requer todos acima)
cd ~/.openclaw/workspace/projects/dashboard
python3 scripts/dashboard_server_v2.py --port 8200 &
curl -s http://127.0.0.1:8200/api/factory/status
```

---

## INSTRUÇÃO PARA O CLAUDE CODE DA NOVA INSTÂNCIA

Após clonar os repos e configurar a infra básica (PostgreSQL, Redis, Ollama), use o seguinte prompt:

---

### PROMPT PARA COLAR:

```
Você está configurando uma nova instância do NaumoStack (OpenClaw) — um sistema de agentes
colaborativos para inteligência de mercado e produção de conteúdo.

Os repositórios já foram clonados em ~/.openclaw/workspace/projects/. A infraestrutura
(PostgreSQL + pgvector, Redis, Ollama) está rodando.

## SUA TAREFA

1. **ANALISAR** todos os projetos clonados:
   - Ler AGENT.md de cada projeto (scout, intel-watcher, opportunity-content-engine,
     content-engine, second-brain, llm-router, dashboard)
   - Ler TEAM.md e CLAUDE.md do workspace
   - Ler lib/stack_identity.py para entender o sistema de configuração
   - Verificar requirements.txt de cada projeto
   - Identificar configurações que precisam de adaptação

2. **VERIFICAR** o que precisa ser ajustado:
   - Paths hardcoded (/Users/macnaumo ou IPs específicos) em YAML, JSON e shell scripts
   - .env files que precisam ser preenchidos
   - config.yaml de cada projeto (feeds RSS, modelos, thresholds)
   - stack.json — validar se existe e se os campos estão corretos
   - Conexão com PostgreSQL, Redis e Ollama
   - Modelos Ollama instalados vs requeridos pelo llm-router/config.yaml

3. **PROPOR PLANO DE IMPLANTAÇÃO** faseado:

   FASE 0 — Configuração base:
   - Criar/validar stack.json
   - Criar .env files (de .env.example onde existir)
   - Substituir paths hardcoded
   - Criar venvs e instalar dependências
   - Validar conexões (PG, Redis, Ollama)

   FASE 1 — Agentes standalone (sem interdependência):
   - Scout (collector, CPU-only) — validar com --dry-run
   - Second-Brain (knowledge indexer) — validar vault_indexer.py
   - LLM-Router — iniciar e validar /health

   FASE 2 — Analysts (dependem de Scout + LLM-Router):
   - Intel-Watcher — validar com --dry-run
   - Opportunity-Content-Engine — validar com --dry-run

   FASE 3 — Content (depende de Analysts):
   - Content-Engine — validar produce_draft.py --dry-run
   - Dashboard — iniciar e validar endpoints

   FASE 4 — Produção:
   - Configurar LaunchAgents/cron para execução automática
   - Pipeline completo end-to-end (Scout → Intel → Content)
   - Configurar Telegram bot (se disponível)
   - Monitorar 24h para estabilidade

4. **EXECUTAR** cada fase com confirmação do operador entre fases.

## REGRAS IMPORTANTES

- NÃO instalar/baixar modelos Ollama sem autorização explícita
- NÃO modificar dados de outros projetos (cada agente tem seu diretório)
- NÃO expor serviços na rede pública (bind 127.0.0.1 only)
- NÃO commitar .env files com credenciais
- Respeitar ARQ-1: ZERO paths hardcoded de user/host no código
- Respeitar DEV-4: Requests externos SEMPRE com timeout explícito
- Manter correlation_id end-to-end em toda comunicação cross-agent
- Capital-copilot é EXCLUÍDO — ignorar completamente

## INFORMAÇÕES DO OPERADOR

- Nome: [PREENCHER]
- Plataformas: [twitter/linkedin/ambos]
- Idioma do conteúdo: [en/pt-br/outro]
- Domínio de interesse: [AI/tech/finance/outro]
- Hardware: [descrever: CPU, RAM, GPU]
- PostgreSQL host: [IP ou hostname]
- Redis host: [IP ou hostname]
- Vault path (Obsidian): [caminho]
- Telegram bot token: [se disponível, ou "não tenho"]

Comece analisando os projetos e me apresente o diagnóstico antes de fazer qualquer modificação.
```

---

## NOTAS FINAIS

### O que NÃO está nos repos (precisa criar manualmente)
- `~/.openclaw/stack.json` — identidade da instância
- `~/.openclaw/openclaw.json` — config do gateway/CLI
- `~/.openclaw/cron/jobs.json` — jobs agendados
- `.env` files em cada projeto
- Virtual environments (`.venv/`)
- Vault do Obsidian (conteúdo pessoal)
- LaunchAgent plists (macOS) ou systemd units (Linux)

### Schemas PostgreSQL criados automaticamente
- `second_brain.knowledge_nodes` — notas indexadas
- `second_brain.knowledge_chunks` — chunks para retrieval
- `second_brain.knowledge_entities` — entidades LightRAG
- `second_brain.knowledge_relations` — relações LightRAG
- `second_brain.knowledge_communities` — comunidades detectadas
- `second_brain.cross_connections` — conexões semânticas

### Contratos JSON (docs/schemas/)
- `analyzable_item.json` — Scout → Analysts
- `signal.json` — Analyst ↔ Analyst
- `approved_for_content.json` — Analyst → Content
- `lesson_learned.json` — Retro → Procedural memory
