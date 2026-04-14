# PROPOSTA: Segundo Cérebro Genius-Level — OpenClaw + Obsidian
**Documento para análise autônoma pelo OpenClaw**
**Versão:** 1.1.0
**Data:** 2026-04-12
**Autor:** Gabriel (naumo)
**Destino:** OpenClaw Agent Orchestrator
**Escopo:** OpenClaw + Copilote de Capital

---

## INSTRUÇÕES PARA O OPENCLAW

Este documento é uma proposta de implementação para ser analisada e executada de forma autônoma.
Para cada módulo, o OpenClaw deve:

1. **Verificar viabilidade técnica** com o hardware disponível
2. **Identificar conflitos** com as restrições operacionais definidas na seção 3
3. **Estimar esforço** de implementação em horas/sprints
4. **Propor ordem de implementação** baseada em dependências e ROI
5. **Gerar relatório de decisão** com status: APROVAR / AJUSTAR / REJEITAR + justificativa
6. **Simular o ganho real** considerando o cenário atual do Copilote de Capital e do próprio OpenClaw

---

## 1. CONTEXTO E OBJETIVOS

### 1.1 Problema a Resolver

O Copilote de Capital opera hoje com conhecimento estático de mercado. Cada sessão começa do zero — padrões identificados em sessões anteriores não persistem, insights sobre ativos do portfólio não se conectam entre si, e o OpenClaw não aprende com as interações passadas para melhorar suas respostas futuras. O Obsidian existe como repositório passivo de notas, sem alimentar o raciocínio dos agentes em tempo real.

### 1.2 Objetivo Principal

Transformar o stack OpenClaw + Obsidian em um sistema de cognição aumentada focado em dois domínios:

- **OpenClaw:** meta-aprendizado contínuo — o orquestrador melhora a cada interação, sem fine-tuning de modelos
- **Copilote de Capital:** memória persistente de mercado — padrões, teses de investimento, análises e decisões acumulam-se e retroalimentam os agentes

Requisitos transversais:
- Opera 100% local — nenhum dado sai para cloud
- Alertas e sínteses apenas — nenhuma execução autônoma de ordens
- Não degrada performance do OpenClaw em mais de 15%

### 1.3 Projetos no Escopo

| Projeto | Papel nesta Proposta |
|---|---|
| **OpenClaw** | Beneficiário primário do meta-aprendizado (Módulo F) e orquestrador de todos os módulos |
| **Copilote de Capital** | Beneficiário primário da memória de mercado (Módulos B, C, D, E) |
| **Obsidian Vault** | Fonte de verdade do conhecimento — alimenta os embeddings pgvector |

---

## 2. JUSTIFICATIVA

### 2.1 Por Que Isso Vale o Esforço

**Para o OpenClaw:**

O OpenClaw hoje é um orquestrador reativo — responde bem ao que é perguntado, mas não acumula aprendizado entre sessões. O padrão ACE (Agentic Context Engineering, arXiv 2025) demonstra ganhos de +10,6% em benchmarks de agentes e +8,6% em tarefas de domínio sem nenhum fine-tuning do modelo base. O mecanismo é simples: um Curator extrai o que funcionou e o que falhou em cada interação e injeta esse playbook automaticamente na sessão seguinte.

No contexto do OpenClaw, isso significa que cada análise feita hoje sobre o Copilote de Capital torna a próxima análise mais precisa — sem intervenção manual.

**Para o Copilote de Capital:**

Gestão de portfólio é um domínio com alta dependência de contexto histórico. Um agente que não lembra que você analisou XEQT há três semanas e concluiu que o peso em tecnologia estava alto para o seu perfil vai repetir a mesma análise superficial. Um agente com memória persistente parte de onde parou — e conecta padrões que o raciocínio de uma única sessão nunca alcançaria.

O gap mais importante não é ter mais dados de mercado — é ter **memória das suas próprias conclusões** sobre esses dados.

### 2.2 Por Que Obsidian Como Fonte

O Obsidian é o único ponto onde o conhecimento elaborado já existe em formato estruturado. As notas de análise de mercado, as teses de investimento, as reflexões sobre o portfólio — tudo isso já está lá, ou deveria estar. Indexar o vault com pgvector não é criar algo novo: é tornar pesquisável e conectável o que já foi escrito.

### 2.3 Por Que Agora

O Mac Mini M4 Pro com 48GB RAM e o Geekom A8 com PostgreSQL e Redis já formam a infraestrutura necessária. Falta apenas o schema pgvector, o indexador e os agentes. O custo de oportunidade de não implementar isso é continuar com sessões sem memória num domínio — investimentos — onde a qualidade da decisão depende diretamente de contexto acumulado.

---

## 3. RESTRIÇÕES INEGOCIÁVEIS

> **OpenClaw:** Verificar CADA restrição para CADA módulo. Módulo que viola restrição → REJEITAR ou AJUSTAR obrigatoriamente.

```yaml
restricoes_dados:
  - id: DAT-001
    descricao: "Nenhum dado do Copilote de Capital sai da rede local"
    aplicavel_a: todos os módulos
    verificacao: "Módulo faz requests externos com dados de portfólio? Se sim → REJEITAR"

  - id: DAT-002
    descricao: "Credenciais e chaves de API armazenadas apenas localmente"
    aplicavel_a: módulos com autenticação
    verificacao: "Credenciais em arquivo local ou variável de ambiente local? Se não → REJEITAR"

  - id: DAT-003
    descricao: "Logs e histórico de decisões devem ser locais (PostgreSQL/Redis/arquivo)"
    aplicavel_a: módulos com persistência
    verificacao: "Storage é local? Se não → REJEITAR"

restricoes_operacionais:
  - id: OP-001
    descricao: "Nenhuma execução autônoma de ordens ou movimentação de portfólio"
    aplicavel_a: Copilote de Capital — todos os módulos
    verificacao: "Módulo executa ações financeiras? Se sim → converter para alert-only obrigatório"

  - id: OP-002
    descricao: "OpenClaw não pode degradar mais de 15% em tempo de resposta"
    aplicavel_a: Módulo F (ACE Pattern)
    verificacao: "Medir latência baseline antes e após — rejeitar se degradação > 15%"

  - id: OP-003
    descricao: "Processos pesados de indexação rodam fora do horário de uso intenso (08h-18h)"
    aplicavel_a: Módulo B (vault indexer — reindex completo)
    verificacao: "Cron job agendado para 02h00?"
```

---

## 4. MÓDULOS PROPOSTOS

### MÓDULO A — Schema pgvector para Knowledge Graph

**Descrição:** Criar schema dedicado no PostgreSQL do Geekom A8 para armazenar embeddings do vault Obsidian, memória dos agentes do Copilote de Capital e conexões cross-domínio.

**Justificativa:** É a fundação de todos os outros módulos. Sem pgvector, não há busca semântica; sem busca semântica, o conhecimento do vault é inacessível para os agentes. O custo de implementação é baixo (2-4h) e o desbloqueio é total.

**Dependências:** pgvector instalado no PostgreSQL, PostgreSQL >= 14

**Esforço estimado:** 2-4 horas

**Restrições a verificar:** DAT-001, DAT-003

```sql
-- Schema proposto para validação
CREATE SCHEMA IF NOT EXISTS second_brain;

CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE second_brain.knowledge_nodes (
    id              TEXT PRIMARY KEY,           -- hash MD5 do path
    path            TEXT NOT NULL,              -- path relativo no vault
    title           TEXT,
    content         TEXT,
    content_summary TEXT,                       -- resumo comprimido pelo LLM
    embedding       vector(4096),               -- dimensão qwen2.5-coder:14b
    tags            TEXT[],
    project         TEXT,                       -- Copilote | OpenClaw | Geral
    node_type       TEXT DEFAULT 'permanent',   -- fleeting | literature | permanent | moc
    links_to        TEXT[],                     -- paths linkados dentro do vault
    importance_score FLOAT DEFAULT 0.5,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE second_brain.cross_connections (
    id              SERIAL PRIMARY KEY,
    node_a_id       TEXT REFERENCES second_brain.knowledge_nodes(id),
    node_b_id       TEXT REFERENCES second_brain.knowledge_nodes(id),
    connection_type TEXT,                       -- serendipity | explicit | temporal
    strength        FLOAT,                      -- 0.0 a 1.0
    insight         TEXT,                       -- síntese da conexão
    generated_by    TEXT DEFAULT 'openclaw',
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE second_brain.agent_memory (
    id              SERIAL PRIMARY KEY,
    agent_id        TEXT NOT NULL,              -- ID do agente OpenClaw
    memory_type     TEXT NOT NULL,              -- episodic | semantic | procedural
    project         TEXT,                       -- Copilote | OpenClaw
    content         TEXT NOT NULL,
    embedding       vector(4096),
    relevance_score FLOAT DEFAULT 1.0,
    expires_at      TIMESTAMPTZ,                -- NULL = permanente
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    accessed_at     TIMESTAMPTZ
);

CREATE TABLE second_brain.copilote_teses (
    id              SERIAL PRIMARY KEY,
    ativo           TEXT NOT NULL,              -- ex: XEQT, BTC, LINK
    tese            TEXT NOT NULL,              -- tese de investimento em texto livre
    status          TEXT DEFAULT 'ativa',       -- ativa | revisada | encerrada
    embedding       vector(4096),
    criada_em       TIMESTAMPTZ DEFAULT NOW(),
    revisada_em     TIMESTAMPTZ
);

-- Índices vetoriais
CREATE INDEX ON second_brain.knowledge_nodes
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

CREATE INDEX ON second_brain.agent_memory
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 50);

CREATE INDEX ON second_brain.copilote_teses
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 50);

CREATE INDEX ON second_brain.knowledge_nodes (project, node_type);
CREATE INDEX ON second_brain.knowledge_nodes (updated_at DESC);
```

**Critério de aprovação:** pgvector disponível + schema criado com sucesso + sem conflito com schemas existentes

---

### MÓDULO B — Obsidian Vault Indexer

**Descrição:** Serviço Python que monitora o vault Obsidian em tempo real, gera embeddings via Ollama local e persiste no schema pgvector. Indexa especialmente notas marcadas como `project: Copilote` ou `project: OpenClaw`.

**Justificativa:** O vault Obsidian já contém o conhecimento elaborado — análises de mercado, teses de investimento, reflexões sobre o portfólio. Sem indexação, esse conhecimento é acessível apenas via busca textual. Com pgvector, os agentes do Copilote de Capital passam a encontrar a nota "analisei XEQT e vi sobreposição com VEQT" mesmo quando a pergunta for "qual é a minha exposição a tecnologia global?". A busca deixa de ser literal e passa a ser semântica.

**Dependências:** Módulo A aprovado, Obsidian Local REST API plugin ativo, Ollama com qwen2.5-coder:14b no Mac Mini

**Esforço estimado:** 6-10 horas

**Restrições a verificar:** DAT-001 (embeddings gerados localmente via Ollama), DAT-003, OP-003

```python
# vault_indexer.py — estrutura para validação
# Roda no Mac Mini como serviço launchd

DEPENDENCIES = [
    "watchdog>=4.0.0",
    "psycopg2-binary>=2.9.0",
    "ollama>=0.3.0",
    "python-frontmatter>=1.1.0",
    "httpx>=0.27.0"  # para Obsidian Local REST API
]

CONFIG_REQUIRED = {
    "vault_path": "PATH_ABSOLUTO_DO_VAULT_OBSIDIAN",
    "obsidian_api_url": "http://localhost:27123",
    "ollama_url": "http://localhost:11434",
    "ollama_model": "qwen2.5-coder:14b",
    "postgres_dsn": "postgresql://user:pass@geekom-a8:5432/DB_NAME",
    "projects_filter": ["Copilote", "OpenClaw", "Geral"],
    "embedding_batch_size": 10,
    "schedule_full_reindex": "02:00"  # OP-003
}

PIPELINE = [
    "1. watchdog monitora vault_path para .md criados ou modificados",
    "2. Lê frontmatter YAML — extrai project, tags, type",
    "3. Filtra: processa apenas projetos em projects_filter",
    "4. Extrai links internos [[wikilinks]] → popula links_to",
    "5. Trunca conteúdo a 4000 tokens",
    "6. Solicita embedding ao Ollama local (DAT-001: sem saída de rede)",
    "7. Solicita summary comprimido ao Ollama local",
    "8. Upsert em second_brain.knowledge_nodes",
    "9. Detecta notas de tese de investimento → upsert em second_brain.copilote_teses"
]
```

**Critério de aprovação:** Ollama respondendo + Obsidian API acessível + vault path configurado + pelo menos 1 nota de teste indexada com sucesso

---

### MÓDULO C — Agente de Memória do Copilote de Capital

**Descrição:** Agente especializado que, antes de cada análise do Copilote de Capital, injeta automaticamente no contexto: teses ativas do portfólio, análises anteriores do ativo solicitado, e padrões identificados em sessões passadas. O agente nunca executa ordens — apenas enriquece o contexto.

**Justificativa:** O problema mais caro do Copilote hoje é a amnésia entre sessões. Cada vez que o usuário pergunta "como está minha exposição a cripto?", o agente analisa do zero sem saber que há três semanas foi identificado que a alocação em HYPE estava acima do target. Com este módulo, a pergunta chega ao agente já com o histórico relevante injetado — e a resposta parte de onde a última análise terminou, não do zero.

**Dependências:** Módulos A e B aprovados, pelo menos 20 notas de Copilote indexadas

**Esforço estimado:** 8-12 horas

**Restrições a verificar:** DAT-001, DAT-003, OP-001

```yaml
agent_config:
  id: copilote_memory_agent
  role: "Context enricher — nunca executa, apenas informa"

  trigger:
    - pre_analysis: "antes de qualquer análise do Copilote de Capital"
    - on_asset_mention: "quando ativo específico é mencionado (XEQT, BTC, etc.)"
    - scheduled: "diário às 07h00 — consolida memória da semana"

  context_injection:
    passo_1: "Recebe pergunta/ativo da sessão atual"
    passo_2: "Busca semântica em second_brain.knowledge_nodes (top-10, projeto=Copilote)"
    passo_3: "Busca tese ativa em second_brain.copilote_teses para o ativo"
    passo_4: "Busca memória episódica em second_brain.agent_memory (últimas 30 entradas)"
    passo_5: "Sintetiza contexto histórico relevante"
    passo_6: "Injeta no prompt do agente principal como seção [CONTEXTO HISTÓRICO]"

  output_format: |
    [CONTEXTO HISTÓRICO — Copilote de Capital]
    Ativo: {{ATIVO}}
    Última análise: {{DATA}} — {{RESUMO_ULTIMA_ANALISE}}
    Tese ativa: {{TESE_ATUAL}}
    Padrões identificados: {{PADROES}}
    Alertas anteriores pendentes: {{ALERTAS}}
    Notas relacionadas: {{LISTA_PATHS}}

  llm_backend:
    primary: "ollama/qwen2.5-coder:14b (local — DAT-001 ✓)"
    fallback: "nenhum — síntese de contexto deve ser 100% local"

  restricao_critica: "OP-001 — este agente NUNCA sugere execução de ordens"
```

**Critério de aprovação:** Contexto histórico corretamente injetado em teste manual + latência de injeção < 3s

---

### MÓDULO D — Morning Briefing do Copilote de Capital

**Descrição:** Agente que envia briefing diário via Telegram com o estado consolidado do portfólio sob a perspectiva do conhecimento acumulado: o que mudou desde ontem, qual tese está sendo testada pelo mercado, e qual pergunta vale investigar hoje.

**Justificativa:** O valor de ter memória persistente é proporcional à frequência com que ela é consultada. Um briefing diário garante que o conhecimento acumulado não fique esquecido no banco de dados — ele chega ativamente no começo do dia, orientando o foco de análise. É o equivalente a ter um analista que releu todas as suas notas antes de te ligar de manhã.

**Dependências:** Módulo C aprovado

**Esforço estimado:** 4-6 horas

**Restrições a verificar:** DAT-001, OP-001

```yaml
agent_config:
  id: copilote_briefing_agent
  schedule: "07:00 diário (dias úteis)"

  data_sources:
    - second_brain.knowledge_nodes (notas Copilote modificadas últimas 48h)
    - second_brain.cross_connections (conexões geradas ontem)
    - second_brain.copilote_teses (teses ativas — status e última revisão)
    - second_brain.agent_memory (alertas pendentes do agente Copilote)

  output_sections:
    - "📊 TESES ATIVAS: resumo das teses em andamento e se o mercado as confirma ou refuta"
    - "🔗 CONEXÃO DO DIA: insight entre duas notas que o agente identificou como relacionadas"
    - "📝 NOTA RELEVANTE: análise mais recente que merece revisão hoje"
    - "❓ PERGUNTA DO DIA: hipótese de investigação gerada pelo Módulo C"
    - "⚠️ ALERTAS PENDENTES: alertas de sessões anteriores ainda sem resolução"

  privacidade:
    - sem_dados_de_posicao_exata: true   # não expõe valores nominais
    - sem_execucao: true                 # OP-001
    - dados_ficam_locais: true           # DAT-001
```

**Critério de aprovação:** Telegram bot ativo + briefing de teste enviado com sucesso + formato legível

---

### MÓDULO E — Captura Rápida via Telegram → Obsidian

**Descrição:** Qualquer mensagem enviada ao OpenClaw via Telegram com prefixo `/nota`, `/tese`, `/insight` ou `/alerta` é processada, enriquecida com links para notas existentes, e salva como nota no Obsidian sob o projeto correto.

**Justificativa:** A captura é o gargalo de qualquer sistema de conhecimento. Se o atrito para salvar uma observação de mercado é alto (abrir Obsidian, criar nota, adicionar tags, linkar manualmente), a captura não acontece. Com este módulo, um `/tese LINK parece subvalorizado para o ciclo de tokenização` via Telegram vira nota estruturada no vault em segundos, já com links sugeridos para notas existentes sobre LINK e sobre o ciclo atual.

**Dependências:** Módulo A aprovado, Obsidian Local REST API acessível

**Esforço estimado:** 3-5 horas

**Restrições a verificar:** DAT-001

```yaml
agent_config:
  id: quick_capture_agent
  trigger: "mensagem Telegram com /nota, /tese, /insight, /alerta, /decisao"

  pipeline:
    - "1. Recebe mensagem — detecta prefixo e projeto (Copilote ou OpenClaw)"
    - "2. Extrai ativos mencionados (XEQT, BTC, LINK, etc.)"
    - "3. Busca top-5 notas similares no pgvector"
    - "4. Sugere 3 links para notas existentes"
    - "5. Cria nota .md em /vault/fleeting/ via Obsidian REST API"
    - "6. Responde no Telegram: confirmação + links sugeridos"
    - "7. Se prefixo /tese: também upsert em second_brain.copilote_teses"

  note_template: |
    ---
    date: {{ISO_DATE}}
    project: {{PROJETO_DETECTADO}}
    type: fleeting
    ativos: [{{ATIVOS_MENCIONADOS}}]
    tags: [{{TAGS_SUGERIDAS}}]
    links_sugeridos: [{{TOP_3_LINKS}}]
    status: processar
    ---

    {{CONTEUDO_DA_MENSAGEM}}

    ## Capturado via OpenClaw
    Timestamp: {{TIMESTAMP}}
    Notas relacionadas: {{RESUMO_NOTAS_RELACIONADAS}}
```

**Critério de aprovação:** Nota criada no vault em teste manual + link correto aparecendo na resposta Telegram

---

### MÓDULO F — ACE Pattern para Meta-Aprendizado do OpenClaw

**Descrição:** Implementar o padrão Generator → Reflector → Curator no pipeline do OpenClaw. Cada interação — especialmente análises do Copilote de Capital — alimenta um context playbook que melhora automaticamente as sessões seguintes, sem modificar os modelos LLM.

**Justificativa:** O OpenClaw hoje trata cada sessão como independente. O padrão ACE (arXiv 2025) resolve isso com um loop de três agentes: o Generator produz a resposta normal; o Reflector avalia o que funcionou e o que faltou; o Curator extrai os aprendizados e atualiza um playbook persistente. Na próxima sessão, esse playbook é injetado automaticamente. O resultado documentado no paper é +10,6% em benchmarks de agentes sem nenhum fine-tuning. No contexto do Copilote de Capital, isso significa que o OpenClaw aprende gradualmente o perfil de risco, as preferências de análise, e os padrões de raciocínio do usuário — tornando cada análise mais afinada que a anterior.

**Dependências:** Módulo A aprovado

**Esforço estimado:** 10-16 horas (maior complexidade — impacta o pipeline central do OpenClaw)

**Restrições a verificar:** DAT-001, DAT-002, OP-002

```yaml
ace_loop:
  generator:
    descricao: "Agente principal existente — sem modificação"
    nota: "O ACE envolve o Generator, não o modifica"

  reflector:
    descricao: "Novo agente — avalia a qualidade da resposta do Generator"
    triggers:
      - "após cada interação com o usuário"
      - "após conclusão de análise do Copilote de Capital"
    output:
      quality_score: "float 0-1"
      errors_detected: "list[str]"
      missing_context: "list[str] — o que o Generator não sabia mas deveria saber"
      improvement_suggestions: "list[str]"

  curator:
    descricao: "Novo agente — extrai aprendizados e atualiza playbook"
    storage:
      primary: "second_brain.agent_memory (PostgreSQL local — DAT-001 ✓)"
      format: "entry por tipo de tarefa + projeto"
    playbook:
      arquivo: "skills.md por projeto + learnings_global.md"
      injecao: "automática no contexto do Generator na próxima sessão"

  ganho_esperado:
    fonte: "paper ACE, arXiv 2025"
    benchmark_gain: "+10.6% em tarefas de agente"
    domain_gain: "+8.6% em tarefas de domínio específico"
    metodo: "sem fine-tuning do LLM — apenas injeção de playbook"
    restricao_op002: "medir latência baseline antes de ativar em produção"
```

**Critério de aprovação:** Módulo A funcionando + latência baseline medida + degradação após ACE < 15% (OP-002) + pelo menos 5 ciclos de aprendizado testados

---

## 5. MATRIZ DE DEPENDÊNCIAS

```
MÓDULO A (pgvector schema)  ← FUNDAÇÃO — bloqueia todos os outros
    ├─► MÓDULO B (vault indexer)
    │       ├─► MÓDULO C (memória Copilote)
    │       │       └─► MÓDULO D (morning briefing)
    │       └─► MÓDULO E (captura rápida)  [paralelo com C]
    └─► MÓDULO F (ACE pattern)             [paralelo com B]
```

**Ordem recomendada:**
1. **Módulo A** — fundação (2-4h) — desbloqueia tudo
2. **Módulo E** — quick win (3-5h) — reduz atrito de captura imediatamente
3. **Módulo B** — indexação do vault (6-10h) — constrói massa crítica de conhecimento
4. **Módulo F** — ACE no OpenClaw (10-16h) — melhora o orquestrador
5. **Módulo C** — memória do Copilote (8-12h) — requer vault com notas suficientes
6. **Módulo D** — briefing diário (4-6h) — requer Módulo C estável

---

## 6. SIMULAÇÃO DE GANHO REAL — SOLICITAÇÃO AO OPENCLAW

> **OpenClaw:** Esta é a seção mais importante do documento. Após concluir a verificação de hardware e restrições (passos 1-3), você deve realizar uma simulação fundamentada do ganho real que cada módulo trará para o Copilote de Capital e para o seu próprio funcionamento.

### 6.1 Dados a Coletar Antes de Simular

A simulação deve ser baseada no **cenário atual real**, não em benchmarks genéricos. Para isso, o OpenClaw deve primeiro coletar:

```yaml
estado_atual_copilote_de_capital:
  - Quantas sessões de análise o Copilote teve nos últimos 30 dias?
  - Qual é o tempo médio de uma sessão de análise?
  - Com que frequência o usuário repete contexto já fornecido anteriormente?
  - Quantas notas no vault estão marcadas como projeto Copilote?
  - Quantas teses de investimento estão documentadas?
  - Quais ativos têm mais notas de análise (maior base para memória)?

estado_atual_openclaw:
  - Quantas interações o OpenClaw processa por semana?
  - Quantos contextos diferentes o usuário precisa reintroduzir por sessão?
  - O arquivo skills.md existe atualmente? Qual é seu tamanho e última atualização?
  - Qual é a latência média de resposta atual (baseline para OP-002)?

infraestrutura:
  - pgvector já está instalado? Versão?
  - Quantas notas existem no vault no total?
  - Qual é a carga atual do Mac Mini durante análises do Copilote (CPU/RAM)?
```

### 6.2 Formato da Simulação por Módulo

Para cada módulo aprovado, gerar:

```yaml
simulacao_modulo_X:
  cenario_sem_modulo:
    descricao: "o que acontece hoje sem este módulo"
    custo_estimado: "tempo ou qualidade perdida por sessão/semana"
    exemplo_concreto: "situação real que ilustra o problema"

  cenario_com_modulo:
    descricao: "o que muda após implementação"
    ganho_quantitativo: "estimativa em minutos economizados ou % de melhoria"
    ganho_qualitativo: "o que fica melhor na qualidade da análise"
    exemplo_concreto: "a mesma situação, resolvida pelo módulo"

  premissas:
    - lista de premissas usadas
    - onde a estimativa pode ser otimista ou conservadora

  break_even:
    esforco_implementacao: "horas"
    ganho_por_semana: "horas ou valor qualitativo"
    semanas_para_recuperar: "esforco / ganho_semanal"
```

### 6.3 Simulação Consolidada

Ao final, gerar visão consolidada:

```yaml
simulacao_consolidada:
  ganho_openclaw:
    reducao_contexto_repetido_por_sessao: "X minutos estimados"
    melhoria_qualidade_respostas: "X% baseado em ACE benchmark ajustado ao domínio"
    velocidade_de_aprendizado: "estimativa de entradas no playbook por semana"

  ganho_copilote_de_capital:
    reducao_tempo_orientacao_por_sessao: "X minutos"
    analises_com_contexto_historico: "X% das sessões (vs 0% hoje)"
    teses_rastreadas_automaticamente: "número esperado após 30 dias"
    insights_cross_nota_por_semana: "estimativa baseada no volume atual do vault"

  roi_geral:
    esforco_total_implementacao: "X horas (soma dos módulos aprovados)"
    ganho_semanal_estimado: "X horas + qualidade"
    payback_em_semanas: "calculado"
    recomendacao_final: "IMPLEMENTAR COMPLETO | IMPLEMENTAR PARCIAL | ADIAR"
    justificativa_recomendacao: string
```

---

## 7. CRITÉRIOS DE SUCESSO MENSURÁVEIS

```yaml
metricas_semana_1:
  - notas_copilote_indexadas: ">= 20"
  - embeddings_gerados_sem_erro: "100%"
  - latencia_busca_semantica: "< 200ms para top-10"
  - captura_telegram_funcionando: "sim/não"

metricas_semana_2:
  - capturas_via_telegram_por_semana: ">= 3"
  - links_sugeridos_aceitos_pelo_usuario: ">= 50%"
  - contexto_historico_injetado_em_analises: ">= 80% das sessões Copilote"

metricas_semana_4:
  - teses_documentadas_e_rastreadas: ">= 5"
  - insights_morning_briefing_uteis: ">= 3 de 5 (avaliação subjetiva)"
  - ciclos_ace_executados: ">= 20"
  - degradacao_latencia_openclaw: "< 15% (OP-002)"

metricas_mes_2:
  - reducao_tempo_orientacao_por_sessao: ">= 30%"
  - analises_com_referencia_a_historico: ">= 60% das sessões"
  - playbook_openclaw_entries: ">= 50 aprendizados"
```

---

## 8. AÇÕES SOLICITADAS AO OPENCLAW

```yaml
analise_solicitada:

  passo_1_verificacao_hardware:
    descricao: "Verificar disponibilidade de cada componente"
    saida_esperada:
      - status de cada serviço (UP/DOWN/UNKNOWN)
      - RAM disponível no Mac Mini
      - pgvector instalado no PostgreSQL do Geekom A8 (sim/não + versão)
      - Obsidian REST API acessível na porta 27123 (sim/não)
      - path do vault Obsidian
      - contagem de notas .md total e por projeto

  passo_2_verificacao_restricoes:
    descricao: "Para cada módulo (A-F), verificar restrições da seção 3"
    saida_esperada:
      - tabela: módulo × restrição × status (OK / VIOLA / PRECISA_AJUSTE)
      - lista de ajustes necessários por módulo

  passo_3_avaliacao_viabilidade:
    descricao: "Para cada módulo, emitir veredicto fundamentado"
    formato_veredicto:
      status: "APROVAR | AJUSTAR | REJEITAR"
      justificativa: string
      pre_requisitos_faltando: list[str]
      ajustes_necessarios: list[str]
      esforco_estimado_horas: int

  passo_4_simulacao_ganho:
    descricao: "Executar simulação de ganho real conforme seção 6"
    instrucao: "Coletar dados do estado atual ANTES de simular — não usar benchmarks genéricos"
    saida_esperada:
      - simulacao por módulo aprovado (formato seção 6.2)
      - simulacao consolidada (formato seção 6.3)
      - recomendação final com justificativa

  passo_5_plano_execucao:
    descricao: "Gerar plano de execução para módulos aprovados"
    saida_esperada:
      - ordem_de_implementacao baseada na matriz de dependências
      - sprint_1 (semana 1): módulos e tarefas concretas
      - sprint_2 (semana 2): módulos e tarefas concretas
      - primeiro_passo_concreto: ação que pode ser iniciada hoje

  passo_6_relatorio_telegram:
    descricao: "Consolidar e enviar relatório via Telegram"
    formato: |
      ## 🧠 Segundo Cérebro — Análise OpenClaw

      ### 🖥️ Hardware
      [status de cada componente]

      ### 📦 Módulos — Veredicto
      [tabela: módulo | status | esforço estimado]

      ### 📈 Simulação de Ganho Real
      [resumo por módulo + ROI consolidado baseado no cenário atual]

      ### 🗓️ Plano de Execução
      [Sprint 1 e Sprint 2 com tarefas concretas]

      ### ▶️ Próximo Passo Imediato
      [ação concreta para iniciar hoje]

      ### 🚧 Bloqueios
      [pré-requisitos em falta antes de começar]
```

---

## 9. NOTAS TÉCNICAS

### pgvector — Dimensão do Embedding

O modelo `qwen2.5-coder:14b` via Ollama gera embeddings de **4096 dimensões**. Se o modelo for substituído por outro com dimensão diferente, o schema precisa ser migrado. Criar uma view de abstração facilita migrações futuras.

### Obsidian REST API

Plugin: **Local REST API** (por coddingtonbear). Porta padrão: 27123. Requer API key configurada nas opções do plugin. O OpenClaw deve verificar se está ativo antes de tentar qualquer operação no vault.

### Isolamento de Schema

O schema `second_brain` deve ser criado separado do schema `public`. Isso garante que operações de manutenção em outros schemas não afetem o segundo cérebro.

### Batch de Indexação

Para o reindex completo inicial do vault, rodar em batch de 10 notas com sleep de 1s entre ciclos. Agendar para 02h00 (OP-003). O Mac Mini M4 Pro com 48GB RAM suporta `qwen2.5-coder:14b` com folga.

### Compatibilidade com MultiAgent Factory

Os módulos devem ser compatíveis com a arquitetura MultiAgent Factory (8-agent team) existente. O Módulo F (ACE) integra-se ao pipeline de orquestração — não o substitui.

---

## 10. GLOSSÁRIO PARA O OPENCLAW

```yaml
termos:
  vault_obsidian: "diretório de arquivos .md gerenciados pelo Obsidian"
  embedding: "vetor numérico que representa o significado semântico de um texto"
  pgvector: "extensão PostgreSQL para busca vetorial por similaridade semântica"
  cosine_similarity: "métrica de similaridade entre vetores (0=diferente, 1=idêntico)"
  ACE_pattern: "loop Generator→Reflector→Curator para meta-aprendizado de agentes"
  context_playbook: "base de aprendizados acumulados injetada automaticamente no contexto"
  fleeting_note: "nota de captura rápida, ainda não processada"
  permanent_note: "síntese elaborada em palavras próprias, densamente linkada"
  tese_de_investimento: "hipótese documentada sobre um ativo — rastreada ao longo do tempo"
  copilote_de_capital: "sistema de agentes para análise e acompanhamento de portfólio pessoal"
  note_gardening: "sessão periódica de processamento de notas fleeting em permanent"
```

---

*Fim do documento de proposta.*
*OpenClaw: iniciar pelo passo_1_verificacao_hardware. Executar passos em sequência.*
*O passo_4_simulacao_ganho é o mais valioso desta análise — coletar dados reais antes de simular.*