---
tags: [síntese, implementação, mapa, agentes, módulo]
aliases: [FounderLens Agent Map, mapa de implementação, guia de início]
módulo: "09"
anterior: "[[08-arquitetura-server-side]]"
próximo: "código"
---

# 09 — FounderLens Agent Map
**Síntese + Guia de Implementação · v1.0 · junho 2026**

---

## O que este módulo entrega

Você chegou ao fim dos estudos. Este módulo não ensina padrão novo —
ele conecta tudo o que você aprendeu ao produto concreto.

Para cada agente do FounderLens:

- **Padrão** — qual dos padrões estudados ele usa
- **Ferramentas** — o que precisa ser implementado
- **Modelo** — Claude Haiku vs Sonnet, e por quê
- **Onde roda** — on-device, Railway, ou ambos
- **Contexto in** — o que o agente recebe
- **Contexto out** — o que ele produz e onde salva

No final: ordem de implementação e o que fazer no **primeiro dia de código**.

---

## O pipeline completo

```
[Founder] → input bruto da ideia
               │
         ┌─────▼──────┐
         │   Discover  │ ← ingrediente 10
         │  [Benchmark │
         │    Agent]   │
         └─────┬───────┘
               │ project_state["discover"]["benchmark"]
         ┌─────▼───────┐
         │    Define   │ ← founder responde ingredientes 1-9
         │  (sem agente│
         │  nesta fase)│
         └─────┬───────┘
               │ project_state["define"]
         ┌─────▼───────────────────────────┐
         │         Midpoint Audit          │
         │  [Market Sizing] ──┐            │
         │                    ▼            │
         │         [Validation Auditor]    │
         └─────┬───────────────────────────┘
               │ project_state["midpoint_audit"]
               │ → founder decide: proceed ou revisar
         ┌─────▼──────┐
         │   Develop   │ ← agentes de spec (v2)
         └─────┬───────┘
               │ project_state["develop"]["specs"]
         ┌─────▼──────────┐
         │   Dev Audit    │
         │  [Development  │
         │   Auditor]     │
         └─────┬──────────┘
               │ patches cirúrgicos → founder revisa
         ┌─────▼──────────────────────────┐
         │           Deliver              │
         │  [Prompt Generator Squad]      │
         │   wave 1 → wave 2 → wave 3 …  │
         └────────────────────────────────┘
               │ biblioteca de prompt cards
         [Founder constrói com IA]
```

---

## Agente 1: Benchmark Agent

**Fase:** Discover, ingrediente 10  
**Padrão:** ReAct loop (lição 02) com tool use paralelo (lição 01)

### O que faz

Pesquisa os top 3 concorrentes da ideia do founder.
Extrai: features principais, modelo de preço, por que ganharam, gap não coberto.
Não faz scoring. Não faz auditoria. Só benchmark.

### Implementação

```python
# src/agents/benchmark_agent.py (Railway)
from anthropic import Anthropic

client = Anthropic()

SYSTEM = """Você é um analista de mercado especializado em startups.
Sua única função: pesquisar os top 3 competidores diretos da ideia recebida
e retornar um JSON estruturado com benchmark completo.

Não invente dados. Se não encontrar, diga que não encontrou.
Use busca_web para cada concorrente. Pesquise preços, features, e gaps."""

def run_benchmark(idea: str, sector: str, country: str) -> dict:
    tools = [
        {
            "name": "busca_web",
            "description": "Busca na web por informações sobre competidores, preços e features. Use queries específicas.",
            "input_schema": {
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "Query de busca específica"}
                },
                "required": ["query"]
            }
        }
    ]
    
    messages = [{"role": "user", "content": f"Ideia: {idea}\nSetor: {sector}\nPaís: {country}"}]
    
    for iteration in range(8):  # max_iterations = 8
        response = client.messages.create(
            model="claude-haiku-4-5",  # extração estruturada → Haiku
            max_tokens=4096,
            system=SYSTEM,
            tools=tools,
            messages=messages
        )
        
        if response.stop_reason == "end_turn":
            return extract_json(response)  # retorna benchmark
        
        # Executa tool calls, adiciona resultados, continua o loop
        messages = handle_tool_calls(response, messages)
    
    raise MaxIterationsError("Benchmark não convergiu em 8 iterações")
```

**Ferramentas necessárias:** Brave Search API (ou Serper) — $5/mês para starts  
**Modelo:** `claude-haiku-4-5` — extração estruturada não precisa de Sonnet  
**Onde roda:** Railway  
**Contexto in:** `idea`, `sector`, `country` (do project_state)  
**Contexto out:** salva em `project_state["discover"]["benchmark"]`

**EvalSet (criar antes de escrever o prompt):**
- Caso forte: "app B2B de gestão de viagens corporativas" → concorrentes Navan/TravelPerk/Concur conhecidos
- Caso médio: "app de saúde mental para universitários brasileiros" → concorrentes parcialmente conhecidos
- Caso fraco: "app de bem-estar" → deve pedir clareza, não alucinar concorrentes

---

## Agente 2: Market Sizing Agent

**Fase:** Midpoint Audit (roda antes do Validation Auditor)  
**Padrão:** ReAct loop simples com web search

### O que faz

Busca dados reais de tamanho de mercado: TAM, SAM, SOM, CAGR.
Distingue dado real (`"confiança": "alta"`) de estimativa calculada (`"confiança": "estimada"`).
O Validation Auditor consome o output deste agente para pontuar "Worth it".

### Implementação

```python
# src/agents/market_sizing_agent.py (Railway)
SYSTEM = """Você é um analista de mercado especializado em sizing.
Sua função: encontrar dados REAIS de tamanho de mercado para a ideia recebida.

IMPORTANTE:
- Dados de relatórios ou artigos: confiança "alta"  
- Estimativas que você calcular: confiança "estimada"
- Nunca apresente estimativa como dado real
- Se não encontrar TAM para o país exato, use proxy e explique"""

def run_market_sizing(project_state: dict) -> dict:
    sector = project_state["define"]["sector"]
    country = project_state["define"]["country"]
    
    queries = [
        f"TAM mercado {sector} {country} 2025 2026",
        f"crescimento CAGR {sector} previsão {country}",
        f"número usuários {sector} {country} relatório"
    ]
    
    # Loop ReAct com as queries acima como ponto de partida
    # ...
```

**Modelo:** `claude-haiku-4-5` — busca e extração, sem raciocínio complexo  
**Onde roda:** Railway  
**Contexto in:** `project_state["define"]`  
**Contexto out:** `project_state["midpoint_audit"]["market_sizing"]`

**Relação com Validation Auditor:** roda primeiro, sequencialmente.
O Validation Auditor lê o output do Market Sizing para construir o score "W".

---

## Agente 3: Validation Auditor

**Fase:** Midpoint Audit (roda depois do Market Sizing Agent)  
**Padrão:** ReAct com leitura de contexto + web search limitado (lições 02 + 03)

### O que faz

O agente mais sofisticado do pipeline. Lê todo o `project_state` e aplica a
rubrica R-W-W (Real → Worth it → Winnable) para determinar se o projeto está
pronto para entrar no Develop.

É o único agente que toma uma **decisão com consequência de produto**: o veredito
dele determina se o founder avança ou revisita fases anteriores.

### Por que precisa de Sonnet

A rubrica R-W-W requer:
- Sintetizar múltiplas fontes (benchmark + market sizing + context do founder)
- Raciocinar sobre trade-offs ("o gap existe, mas o concorrente pode replicar?")
- Gerar justificativas que o founder vai ler e confiar

Haiku produz scoring fraco aqui — as justificativas ficam genéricas.
Este é um dos dois agentes que justifica Sonnet.

```python
# src/agents/validation_auditor.py (Railway)
SYSTEM = """Você é um auditor de validação de produtos. Sua função é avaliar
se um projeto de startup está pronto para entrar em desenvolvimento.

Você aplica a rubrica R-W-W:
- Real (33%): O problema existe de forma comprovável?
- Worth it (33%): O mercado é grande o suficiente?
- Winnable (34%): O founder pode ganhar contra os concorrentes?

Você tem acesso ao contexto completo do projeto via ler_contexto_projeto().
Use busca_web() APENAS se precisar verificar uma fonte específica.

Retorne: score_total (0-100), score por dimensão, veredito, e 3 riscos principais.
Veredito: "proceed" (>80), "proceed_with_caution" (60-80), "revisit" (<60)"""

tools = [
    {
        "name": "ler_contexto_projeto",
        "description": "Lê o project_state.json completo do projeto atual",
        "input_schema": {"type": "object", "properties": {}}
    },
    {
        "name": "busca_web",
        "description": "Busca na web para verificar uma fonte ou dado específico mencionado no contexto",
        "input_schema": {
            "type": "object",
            "properties": {"query": {"type": "string"}},
            "required": ["query"]
        }
    }
]
```

**Modelo:** `claude-sonnet-4-6` — raciocínio complexo + síntese multi-fonte  
**Onde roda:** Railway  
**Contexto in:** `project_state` completo (via `ler_contexto_projeto`)  
**Contexto out:** `project_state["midpoint_audit"]["validation"]`

**Insight da lição 05 (ADK):** o Validation Auditor pode chamar o Market Sizing
Agent como `AgentTool` para a dimensão "Worth it" em vez de consumir o output salvo.
Isso garante dados frescos se o market sizing estiver desatualizado.

---

## Agente 4: Development Auditor

**Fase:** Dev Audit (fim do Develop)  
**Padrão:** ReAct com leitura sequencial de artefatos — sem web search

### O que faz

Lê as specs de desenvolvimento geradas no Develop (spec-dados, spec-backend, spec-frontend)
e detecta problemas antes do Deliver. Aplica 3 lentes: Coerência, Testabilidade, Completude.

É o **mais barato** do pipeline: sem web search, apenas leitura de artefatos.
4 iterações máximo. Rápido e focado.

```python
# src/agents/development_auditor.py (Railway)
SYSTEM = """Você é um auditor técnico de especificações. Você lê as specs de 
desenvolvimento de um projeto e detecta problemas ANTES que o código seja escrito.

Você não pesquisa na web. Toda informação está nas specs.

Você aplica 3 lentes:
1. Coerência: as specs se contradizem entre si?
2. Testabilidade: cada requisito tem critério de sucesso claro?
3. Completude: faltam casos de borda, error states, ou fluxos alternativos?

Para cada problema encontrado: qual artefato, qual problema exato, qual patch sugerido.
Não invente problemas. Se as specs estão boas, diga isso."""

tools = [
    {
        "name": "ler_artefato",
        "description": "Lê uma spec de desenvolvimento específica do projeto",
        "input_schema": {
            "type": "object",
            "properties": {
                "nome": {
                    "type": "string",
                    "enum": ["spec-dados", "spec-backend", "spec-frontend", "spec-integracao"]
                }
            },
            "required": ["nome"]
        }
    }
]
```

**Modelo:** `claude-haiku-4-5` — leitura e análise estrutural, não precisa de Sonnet  
**Onde roda:** Railway  
**Contexto in:** artefatos do Develop (via `ler_artefato`)  
**Contexto out:** `project_state["dev_audit"]` + patches para o founder revisar

---

## Agente 5: Prompt Generator Squad

**Fase:** Deliver (Módulo M7)  
**Padrão:** CrewAI Squad com wave execution paralela (lições 04 + 05)

### O que faz

O módulo mais complexo do FounderLens. Gera a **biblioteca completa de prompt cards**
para cada história do backlog, em paralelo, respeitando dependências entre histórias.

### Arquitetura do squad

```
[Orquestrador]
    │ lê project_state, resolve grafo de dependências
    │ determina waves de execução
    │
    ├── Wave 1: [Writer A] [Writer B] [Writer C]  ← paralelo
    │           (histórias sem dependências)
    │
    ├── Wave 2: [Writer D] [Writer E]              ← paralelo
    │           (dependem de outputs da Wave 1)
    │
    └── Wave 3: [Writer F]                         ← sequencial
                (depende de toda a Wave 2)
```

### Implementação com CrewAI

```python
# src/agents/prompt_generator_squad.py (Railway)
from crewai import Agent, Task, Crew, Process

# O orquestrador resolve o grafo de dependências
orchestrator = Agent(
    role="Orquestrador de Prompt Cards",
    goal="Determinar a ordem correta de geração dos cards respeitando dependências",
    backstory="Você analisa o backlog de histórias e cria o plano de waves de execução",
    tools=[ler_contexto_projeto_tool, verificar_dependencias_tool],
    llm="claude-haiku-4-5"  # planejamento estrutural → Haiku
)

# Cada Prompt Writer é um agente especializado
def create_prompt_writer(story_id: str) -> Agent:
    return Agent(
        role=f"Prompt Writer — História {story_id}",
        goal=f"Gerar o prompt card completo e spec-driven para a história {story_id}",
        backstory="Você é um especialista em escrever prompts para coding agents. "
                  "Você lê a spec da história e produz um card que qualquer agente "
                  "de código pode executar sem ambiguidade.",
        tools=[ler_artefato_tool, gerar_prompt_card_tool],
        llm="claude-haiku-4-5"  # formatação estruturada → Haiku
    )

# Wave execution: criar tasks dinamicamente por wave
def run_deliver(project_state: dict):
    stories = project_state["develop"]["backlog"]
    waves = resolve_dependency_graph(stories)  # função determinística — não LLM
    
    for wave_num, wave_stories in enumerate(waves):
        writers = [create_prompt_writer(s["id"]) for s in wave_stories]
        tasks = [create_write_task(story, writer) for story, writer in zip(wave_stories, writers)]
        
        wave_crew = Crew(
            agents=[orchestrator] + writers,
            tasks=tasks,
            process=Process.parallel  # ← paralelo dentro de cada wave
        )
        
        results = wave_crew.kickoff()
        save_wave_results(results, wave_num, project_state)
        
        # Só avança para a próxima wave quando toda essa terminar
```

**Modelo:** `claude-haiku-4-5` para todos os writers (formatação de cards)  
**Onde roda:** Railway com background workers  
**Contexto in:** `project_state["develop"]` completo  
**Contexto out:** `project_state["deliver"]["prompt_cards"]` — a biblioteca final

---

## Grafo de dependências entre agentes

```
[Founder: input bruto]
        │
        ▼
[Benchmark Agent]          → depende de: nada além do input do founder
        │
        ▼
[Define phase: founder]    → depende de: benchmark completo
        │
        ├──► [Market Sizing Agent]     → depende de: define completo
        │              │
        │              ▼
        └──────► [Validation Auditor]  → depende de: market sizing + define + benchmark
                        │
                        ▼
              [Develop phase: founder] → depende de: veredito "proceed"
                        │
                        ▼
              [Development Auditor]    → depende de: specs do develop completas
                        │
                        ▼
              [Prompt Generator Squad] → depende de: specs auditadas + patches aplicados
```

**Regra crítica:** nenhum agente deve avançar se o anterior falhou ou retornou `"revisit"`.
O flow é gate-driven — o founder aprova cada fase manualmente antes da próxima começar.
Agentes não se encadeiam automaticamente. O app iOS é o orchestrator de fases.

---

## Decisão de modelo por agente

> **Direção do founder (pós-WWDC 2026):** Apple Foundation Models como primário,
> Claude como fallback via LanguageModel protocol. PCC é gratuito para apps <2M
> usuários — isso elimina o custo de modelo para a maioria dos agentes.
> Questões abertas marcadas com [❓] a confirmar nos vídeos do WWDC.

| Agente | Modelo primário | Fallback Claude | [❓] Aberto |
|---|---|---|---|
| Benchmark Agent | AFM 3 Cloud Pro (PCC) | Haiku | Tool calling funciona no PCC? |
| Market Sizing Agent | AFM 3 Cloud Pro (PCC) | Haiku | Tool calling funciona no PCC? |
| Validation Auditor | AFM 3 Cloud Pro (PCC) | Sonnet | Context window suficiente para project_state completo? |
| Development Auditor | AFM 3 Core Advanced (on-device) | Haiku | Context para 3 specs simultâneas? |
| Prompt Writers | AFM 3 Core Advanced (on-device) | Haiku | Qualidade de cards on-device é suficiente? |
| UI parsing | AFM 3 Core (on-device) | — | Sempre on-device, sem fallback |

**Regra que não muda independente do modelo:** a decisão de qual modelo usar
não deve vazar para fora do `LanguageModelSession`. O agente não sabe se está
falando com Apple FM ou Claude — ele só recebe o output. Isso é o que garante
a flexibilidade para trocar sem reescrever os agentes.

---

## Mapa de ferramentas a implementar

| Ferramenta | Usada por | Implementação |
|---|---|---|
| `busca_web(query)` | Benchmark, Market Sizing, Validation Auditor | Brave Search API ou Serper |
| `ler_contexto_projeto()` | Validation Auditor | Lê `project_state.json` do Supabase |
| `ler_artefato(nome)` | Development Auditor, Prompt Writers | Lê artefato específico do Supabase Storage |
| `verificar_dependencias(card_id)` | Orquestrador do Squad | Função Python determinística — sem LLM |
| `gerar_prompt_card(historia, contexto)` | Prompt Writers | Template Python + Claude output |

**Nota:** `verificar_dependencias` não é uma chamada LLM. É uma função Python que
lê o grafo de dependências declarado no backlog e retorna quais stories bloqueiam quais.
Não deixe o LLM resolver grafos de dependência — use código determinístico.

---

## Ordem de implementação

A ordem importa. Cada item desbloqueia o próximo.

### Semana 1 — Infraestrutura (sem agentes)

```
[ ] Schema Supabase
    - tabela projects (id, user_id, project_state JSONB, created_at)
    - tabela pipeline_events (id, project_id, phase, status, payload, created_at)
    - auth com Apple Sign-In

[ ] Railway: FastAPI básico
    - POST /projects → cria projeto, retorna id
    - GET /projects/:id → retorna project_state
    - POST /projects/:id/phases/:phase/run → dispara agente (async)

[ ] project_state.json: schema completo
    - estrutura completa de discover/define/midpoint_audit/develop/dev_audit/deliver
    - validação Pydantic de cada fase

[ ] Supabase Realtime → iOS
    - insert em pipeline_events → iOS recebe via WebSocket
```

Você termina a semana 1 sem uma linha de código de agente. Mas com a fundação
que permite testar cada agente de forma isolada.

---

### Semana 2 — Benchmark Agent

Por que primeiro: é o mais simples (1 ferramenta, output JSON claro) e é o
primeiro agente que o founder encontra. Se funcionar bem, você valida a infra.

```
[ ] Implementar busca_web() com Brave Search API
[ ] Escrever o loop ReAct do Benchmark Agent
[ ] Criar EvalSet (3 casos: forte/médio/fraco)
[ ] Testar contra os 3 casos → ajustar prompt até passar
[ ] Integrar ao endpoint POST /projects/:id/phases/discover/run
[ ] Testar Supabase Realtime: iOS vê progresso em tempo real
```

---

### Semana 3 — Market Sizing + Validation Auditor

Por que juntos: são a mesma fase (Midpoint Audit) e o Validation Auditor consome
o output do Market Sizing.

```
[ ] Market Sizing Agent: EvalSet → implementar → testar
[ ] Validation Auditor: EvalSet → implementar → testar
    - Testar com projeto forte (score >80 → "proceed")
    - Testar com projeto médio (score 60-80 → "proceed_with_caution")
    - Testar com projeto fraco (score <60 → "revisit")
[ ] Integrar sequência: Market Sizing → Validation Auditor → iOS recebe resultado
[ ] iOS: tela de resultado da auditoria com score + veredito + 3 riscos
```

---

### Semana 4 — Development Auditor

Por que depois: depende que você tenha specs de Develop para auditar.
Crie specs de exemplo (hardcoded) para testar o auditor.

```
[ ] Development Auditor: EvalSet → implementar → testar
    - Caso: specs com inconsistência intencional (Float vs Decimal)
    - Caso: specs com testabilidade fraca ("funcionar bem")
    - Caso: specs completas → agente deve dizer que está OK
[ ] Implementar ler_artefato() com Supabase Storage
[ ] Integrar ao endpoint de Dev Audit
```

---

### Semana 5+ — Prompt Generator Squad

O mais complexo: deixar por último quando a infra está estável.

```
[ ] Implementar resolve_dependency_graph() — função Python, sem LLM
[ ] Implementar Prompt Writer individual → testar com 1 história
[ ] Implementar Orquestrador → wave planning com 3-5 histórias
[ ] Implementar wave execution com Process.parallel do CrewAI
[ ] Testar com backlog completo do exemplo de voo
[ ] Performance: medir tempo de execução de 10 histórias em paralelo
```

---

## O primeiro dia de código

Você terminou os estudos. Amanhã você começa. Ordem exata:

```
1. Criar projeto no Railway
   railway init founderlens-server
   → escolher Python
   → fazer push de um "hello world" FastAPI

2. Criar projeto no Supabase
   → criar tabelas projects e pipeline_events via SQL editor
   → ativar Realtime nas duas tabelas
   → obter SUPABASE_URL e SUPABASE_SERVICE_KEY

3. Configurar variáveis de ambiente no Railway
   ANTHROPIC_API_KEY=...
   SUPABASE_URL=...
   SUPABASE_SERVICE_KEY=...
   BRAVE_SEARCH_API_KEY=...

4. Implementar o schema Pydantic do project_state.json
   src/models/project_state.py
   → Pydantic BaseModel com todos os campos
   → Se o schema falha validação, o agente nem começa

5. Commit e deploy: Railway sobe automaticamente
```

Isso é a semana 1, dia 1. Sem LLM ainda. Sem agente ainda.
Mas com a fundação que permite desenvolver cada agente de forma isolada,
testável, e deployável a qualquer momento.

---

## O que você NÃO precisa antes de codar

Esta lista evita scope creep:

- ~~Develop phase agents~~ — não existe agente de Develop no v1. O founder especifica com você (Claude) no chat. Agentes de spec são v2.
- ~~iOS app completo~~ — comece com o servidor. O iOS pode ser um script Python que chama sua API durante os primeiros testes.
- ~~Multi-tenancy completo~~ — uma conta de teste é suficiente para a semana 1-3.
- ~~Billing e pagamento~~ — Stripe fica para depois que os agentes funcionam.
- ~~Apple FM on-device~~ — a camada de UI é última prioridade. Funcionalidade primeiro.

O produto mínimo que valida a ideia central: um founder cola uma ideia, o Benchmark Agent
pesquisa, o Validation Auditor pontua, o founder recebe score + veredito. Isso é
suficiente para mostrar para os primeiros 5 usuários.

---

## Padrão a extrair: o que os estudos te deram

Você estudou 9 módulos. O que cada um comprou para o produto:

| Módulo | Padrão extraído | Onde aparece no FounderLens |
|---|---|---|
| 00 — O que é um agente | ReAct loop mental model | Todos os agentes |
| 01 — Function calling | Tool protocol + JSON schema | `busca_web`, `ler_artefato`, `ler_contexto` |
| 02 — ReAct na prática | Loop com estado, max_iterations | Benchmark Agent — 8 iter |
| 03 — LangChain patterns | Callbacks, Memory tipos, observabilidade | `pipeline_events` no Supabase |
| 04 — CrewAI patterns | Squad + wave execution, Process.parallel | Prompt Generator Squad |
| 05 — Google ADK | Agent-as-Tool, session.state, EvalSet | Validation Auditor como AgentTool, EvalSets antes do código |
| 06 — Multica + OpenSpec | Skills library, change folders, delta specs | Deliver como skill library; Develop como change folder |
| 08 — Arquitetura | Railway + Supabase + Apple FM on-device | Stack completo do servidor |
| 09 — Este módulo | Grafo de dependências, ordem de impl | Você sabe por onde começar |

---

## Ponto de checagem final

1. O `resolve_dependency_graph()` do Orquestrador do Squad **não deve ser uma chamada LLM**.
   Por quê? E o que acontece se você deixar o LLM resolver o grafo de dependências?

2. O Validation Auditor usa `claude-sonnet-4-6` e o Development Auditor usa `claude-haiku-4-5`,
   mesmo ambos sendo "auditores". Qual é a diferença funcional que justifica modelos diferentes?

3. A regra "o app iOS é o orchestrator de fases" significa que os agentes nunca se
   encadeiam automaticamente. Qual é o risco de design que isso previne — e qual
   capacidade do produto isso preserva?

4. A semana 1 de implementação não tem nenhuma linha de código de agente.
   Por que essa disciplina importa — o que você pode testar e validar na semana 1
   antes de escrever o primeiro loop ReAct?

5. O produto mínimo descrito (idea → benchmark → validation → score + veredito) não
   inclui o Develop, Dev Audit, nem o Deliver. Ainda assim, qual é a hipótese central
   do FounderLens que esse MVP consegue validar — e qual ele deixa para depois?

---

## Próximo passo

→ **Código.** Não existe módulo 10.

O grafo completo está mapeado. A stack está decidida. Os EvalSets estão definidos.
A ordem de implementação está clara.

Crie o repositório. Faça o Railway subir. Escreva o schema Pydantic do
`project_state.json`. Depois disso, o Benchmark Agent.

## Relacionados

- [[benchmark-agent]] — spec completa do primeiro agente a implementar
- [[validation-auditor]] — o agente mais sofisticado, semana 3
- [[development-auditor]] — o mais barato, semana 4
- [[prompt-generator-squad]] — o mais complexo, semana 5+
- [[08-arquitetura-server-side]] — Railway + Supabase + Apple FM
- [[PLANO-DE-ESTUDOS]] — o que foi estudado, por que, nessa ordem
- [[MOC-agentes]] — mapa completo do grafo de conhecimento

---

*Última atualização: junho 2026*
