---
tags: [arquitetura, infraestrutura, apple-fm, server-side, módulo]
aliases: [Arquitetura server-side, decisão arquitetural, Apple Foundation Models]
módulo: "08"
anterior: "[[06-multica-openspec-map]]"
próximo: "[[09-founderlens-agent-map]]"
---

# 08 — Arquitetura server-side + Apple Foundation Models
**Arquitetura · v1.1 · junho 2026**

---

> **⚠️ Nota de revisão (v1.1):** A versão original deste documento fechou a decisão arquitetural escolhendo a Opção B (server-side com Claude) sem confirmação do founder — violando o princípio central do próprio FounderLens de que o founder decide antes de avançar. A direção correta, definida pelo founder após ler os anúncios do WWDC 2026: **Apple Foundation Models como caminho primário, com Claude como fallback via LanguageModel protocol.** A economia muda completamente com PCC gratuito para apps <2M usuários. Este documento mantém a pesquisa sobre Apple FM (ainda relevante) mas a seção de "A decisão" foi reescrita para refletir a direção correta. As questões abertas que os vídeos do WWDC devem esclarecer estão marcadas com `[❓ WWDC]`.

---

## Por que este módulo existe

As lições 00–07 ensinaram padrões de agente. Agora você precisa de uma resposta
concreta para a pergunta que travou a spec: **onde roda o código?**

Três opções estão abertas desde o início dos estudos (A, B, C). Este módulo fecha
essa decisão com dados reais — incluindo o que a Apple lançou no WWDC 2026 —
e define a infraestrutura do FounderLens antes de você escrever a primeira linha.

---

## O estado real da Apple em junho 2026

Antes de qualquer decisão arquitetural, você precisa saber o que realmente existe.
A Apple lançou a terceira geração dos Foundation Models no WWDC 2026 (8 de junho).
O que veio é mais sofisticado — e mais relevante para a decisão — do que a especulação
anterior sobre "modelos da Apple sendo fracos".

### Os cinco modelos do AFM 3

Apple não lançou um modelo. Lançou cinco:

| Modelo | Onde roda | Tamanho | Parâmetros ativos | Função |
|---|---|---|---|---|
| **AFM 3 Core** | On-device | 3B dense | 3B | NLU leve, roteamento, texto rápido |
| **AFM 3 Core Advanced** | On-device | 20B sparse | 1–4B por prompt | Siri nova, dictação, imagem on-device |
| **AFM 3 Cloud** | Private Cloud Compute | não divulgado | — | Texto + imagem na nuvem |
| **ADM 3 Cloud** | Private Cloud Compute | não divulgado | — | Geração de imagem (Playground, etc.) |
| **AFM 3 Cloud Pro** | NVIDIA GPUs no Google Cloud (PCC extension) | não divulgado | — | Raciocínio complexo, tool use agentico |

O que muda para a decisão do FounderLens:

**AFM 3 Core Advanced** usa uma técnica chamada Instruction-Following Pruning (IFP):
o modelo tem 20B parâmetros armazenados em flash, mas ativa apenas 1–4B por prompt.
Resultado real: qualidade de modelo ~9B num footprint de 3B. Isso é genuinamente bom
para tarefas de UI embarcadas — não é o 3B fraco que você estava imaginando.

**AFM 3 Cloud Pro** roda em GPUs NVIDIA no Google Cloud. A Apple usa outputs do Gemini
para refinar o modelo (distilação), mas o modelo em produção é da Apple, não Gemini.
A capacidade de raciocínio do Cloud Pro é forte — mas você não controla para onde vai
a requisição (pode ser on-device, PCC ou Cloud Pro dependendo da task).

### O que o Foundation Models framework expõe para devs

O framework Swift que você usa como desenvolvedor:

```swift
// 2025: só on-device, só texto
let session = LanguageModelSession()
let response = try await session.respond(to: "Analise esta ideia de produto")

// 2026 novo: imagem + texto on-device
let session = LanguageModelSession()
let response = try await session.respond(
    to: "O que você vê nesta tela?",
    images: [screenshot]  // NOVO em 2026
)

// 2026 novo: LanguageModel protocol — troca de modelo sem reescrever
let cloudSession = LanguageModelSession(model: ClaudeModel())   // Anthropic
let geminiSession = LanguageModelSession(model: GeminiModel())  // Google
let onDeviceSession = LanguageModelSession()                    // Apple default
```

O `LanguageModel` protocol é a novidade arquitetural mais importante do WWDC 2026.
Um único ponto de chamada no Swift consegue rotear para on-device, Claude, ou Gemini
com mínima mudança de código.

### O que o framework **não** consegue fazer (ainda)

Não tem rodeio aqui:

- **Raciocínio complexo e encadeado**: o on-device é excelente para tasks de uma etapa.
  Para uma sequência de 5 agentes com decisões dependentes (como o pipeline completo do
  FounderLens), o AFM 3 Core não tem o raciocínio necessário.
- **Contexto longo**: o pipeline do FounderLens acumula `project_state.json` com milhares
  de tokens. O on-device tem janela de contexto limitada.
- **Web search nativo**: a Apple não expõe ferramentas de busca no framework para devs.
  Você precisaria implementar a tool manualmente e chamar uma API externa.
- **Controle de orquestração**: não há equivalente ao `Process.sequential` do CrewAI
  ou ao `SequentialAgent` do ADK no framework Apple. Você orquestra manualmente.
- **Disponibilidade**: EU (iPhone/iPad excluídos do Siri AI no lançamento), China
  (Apple Intelligence indisponível). Hardware mínimo: iPhone 15 Pro ou iPhone 16.

**Conclusão sobre Apple FM para o pipeline**: o on-device é ótimo para a camada de UI
(análise de texto do usuário, sugestões rápidas, extração de estrutura de inputs simples).
Para o pipeline de agentes em si (Benchmark → Validation → Develop → Deliver),
você precisará de Claude — seja diretamente, seja via Apple LanguageModel protocol.

---

## Revisitando as três opções com dados reais

### Opção A — Pure BYOK

```
Usuário → traz chave Anthropic própria → iOS app chama API diretamente
```

**O que mudou desde a spec original**: ainda mata conversão. Um founder que não sabe
o que é uma API key não vai buscar, criar conta na Anthropic, e colar a chave no app.
Isso elimina 80%+ do seu mercado-alvo.

**Custo para o usuário**: o usuário paga diretamente à Anthropic. Para o FounderLens,
o custo de infra é zero (só o app iOS). Mas a barreira de entrada é alta demais.

**Quando faz sentido**: versão beta fechada para devs/técnicos. Não para o produto final.

---

### Opção B — Server-side (direção preferida)

```
iOS app → seu servidor → Anthropic API
                      → outras tools (web search, etc.)
```

O FounderLens controla a experiência inteira. O usuário não precisa de chave.
Você paga a Anthropic e cobra o usuário (assinatura ou pay-per-use).

**Habilita**: Python frameworks server-side, orquestração explícita completa,
observabilidade (você vê todos os logs), billing próprio.

**Custo estimado por sessão do pipeline completo** (cálculo rough):

O pipeline do FounderLens tem 5 fases com agentes ativos. Estimativa de tokens:

| Fase | Agente | Input estimado | Output estimado |
|---|---|---|---|
| Discover (ingrediente 10) | Benchmark Agent × 3 iter | 15k tokens | 3k tokens |
| Midpoint Audit | Market Sizing Agent | 8k tokens | 2k tokens |
| Midpoint Audit | Validation Auditor | 12k tokens | 2k tokens |
| Develop | Development Auditor | 15k tokens | 3k tokens |
| Deliver | Prompt Generator Squad × 5 histórias | 10k tokens | 5k tokens |
| **Total** | | **~60k input** | **~15k output** |

Com **Claude Haiku 4.5** ($0.80/$4 por milhão input/output):
- 60k × $0.80/M = $0.048 + 15k × $4/M = $0.060
- **~$0.11 por sessão completa**

Com **claude-sonnet-4-6** ($3/$15 por milhão input/output):
- 60k × $3/M = $0.18 + 15k × $15/M = $0.225
- **~$0.40 por sessão completa**

**Estratégia de modelo inteligente**: use Haiku para tasks de extração/estruturação
(Benchmark Agent coletando features de competidores, Prompt Generator Squad formatando
cards) e Sonnet apenas para as auditorias que exigem raciocínio complexo (Validation
Auditor com rubrica R-W-W, Development Auditor com 3 lentes). Isso reduz o custo
da sessão Sonnet para ~$0.20–0.25.

Com margem de segurança (cache miss, retries, contexto extra): **$0.25–0.50 por sessão**.

Para uma assinatura de $15/mês com 5–10 sessões por usuário: margem confortável.

---

### Opção C — Apple Foundation Models híbrido

```
iOS app → AFM 3 Core on-device    → tasks leves de UI
        → seu servidor + Claude   → pipeline de agentes pesados
```

Essa opção não é uma alternativa à B — é uma extensão dela.

Com o novo `LanguageModel` protocol da Apple, o seu app Swift pode chamar Claude
através da camada de abstração da Apple. Mas o pipeline de agentes ainda precisa do
servidor por razões que têm pouco a ver com o modelo:

- Web search requer chamadas HTTP que o on-device não faz nativamente
- O `project_state.json` precisa de storage persistente entre sessões
- Orquestração de múltiplos agentes é mais simples server-side
- Observabilidade (logs, evals) funciona muito melhor no servidor

**O que a Opção C traz de real**: usar AFM 3 Core on-device para a camada de UI do iOS.
Exemplos concretos para o FounderLens:

```
On-device (grátis, offline, rápido):
- Parsing do input do founder enquanto ele digita
- Sugestões de completar frases na interface
- Classificação rápida ("isso parece uma dor? uma feature? um mercado?")
- Resumo local de project_state para exibição

Servidor + Claude (pipeline real):
- Benchmark Agent com web search
- Validation Auditor com rubrica R-W-W completa
- Development Auditor com 3 lentes
- Prompt Generator Squad em wave execution
```

**Conclusão sobre Opção C**: não é uma escolha entre B e C. A resposta correta é
B com C como camada de UI. O servidor roda o pipeline; o on-device melhora a UX.

---

## Candidatos de infraestrutura

Com Opção B confirmada, você precisa de dois componentes:

1. **Runtime dos agentes Python** (onde CrewAI/ADK/seu orchestrator roda)
2. **Database + Auth + Storage** (onde `project_state.json` e sessões ficam)

### Para o runtime de agentes: Railway

```
GitHub push → Railway detecta Dockerfile → deploy automático
```

Railway é a escolha mais simples para um founder-dev solo:

- Suporta Python nativo (não tem as restrições TypeScript-only do Cloudflare Workers)
- CrewAI, LangChain, ADK funcionam sem adaptação
- Deploy a partir do repositório GitHub: `railway up` ou push para main
- Pricing: $5/mês (Starter) para começo, escala por uso
- Logs em tempo real, sem configurar CloudWatch nem similar
- Suporta workers de background (essencial para wave execution do Deliver)

**Por que não Cloudflare Workers**: Workers é TypeScript/JavaScript. Você perderia
acesso aos frameworks Python (CrewAI, ADK). O Workers AI tem modelos, mas não Claude,
e tem limite de CPU de 30s por request — insuficiente para um pipeline de 5 fases.

**Por que não Supabase Edge Functions para os agentes**: Edge Functions roda em Deno
(TypeScript). Mesma restrição que Workers. Supabase é excelente para database e auth,
não para o runtime de agentes.

### Para database + auth: Supabase

```
iOS app → Supabase Auth (Apple Sign-In, email)
        → Supabase Postgres (project_state por usuário)
        → Supabase Storage (uploads do founder, artefatos gerados)
        → Supabase Realtime (streaming de progresso para a UI)
```

Supabase é a escolha certa para a camada de dados:

- Postgres nativo: `project_state.json` pode ser uma coluna JSONB
- Auth com Apple Sign-In integrado (crítico para iOS)
- Realtime subscriptions: o pipeline do servidor pode emitir eventos que a UI
  iOS recebe em tempo real (o founder vê progresso sem polling)
- Storage para uploads (brief do founder, PDFs, imagens de referência)
- Free tier para desenvolvimento; $25/mês para produção

**Stack final:**

```
iOS App (Swift)
    │
    ├── AFM 3 Core on-device → parsing de UI, sugestões rápidas
    │
    ├── Supabase (database + auth + realtime + storage)
    │       └── project_state.json por projeto por usuário
    │
    └── Railway (Python runtime)
            ├── FastAPI ou Flask (HTTP + webhooks)
            ├── Background workers (wave execution)
            ├── CrewAI ou ADK orchestrator
            └── Anthropic API (Claude Haiku + Sonnet)
```

---

## A decisão: Apple FM primeiro, Claude como fallback

**Direção definida pelo founder:** Apple Foundation Models como caminho primário.
Claude (e outros modelos) como fallback via `LanguageModel` protocol.

### Por que essa direção faz sentido

**Economics radicalmente diferentes com PCC gratuito:**

A Opção B (servidor com Claude) custava $0.25–0.50 por sessão porque cada chamada
ia para a Anthropic API. Com Apple FM:

```
On-device (AFM 3 Core / Core Advanced) → $0
Apple PCC (AFM 3 Cloud / Cloud Pro)    → $0 para apps <2M usuários
Claude via LanguageModel protocol       → fallback pago só quando necessário
```

Para um produto com <2M usuários (que é onde o FounderLens vai estar nos primeiros anos),
o custo de modelo é potencialmente zero. Isso muda a viabilidade do negócio.

**Flexibilidade preservada via LanguageModel protocol:**

```swift
// A mesma chamada funciona com qualquer modelo
let session = LanguageModelSession()           // Apple FM (default, free)
let session = LanguageModelSession(model: ClaudeModel())  // Claude (fallback)
let session = LanguageModelSession(model: GeminiModel())  // Gemini (alternativa)
```

Você não está preso a nenhum vendor. Se amanhã Claude superar Apple FM numa dimensão
crítica, você troca um argumento. Se Apple FM melhorar o raciocínio e eliminar a
necessidade de fallback, você remove o fallback.

### Questões abertas a resolver nos vídeos do WWDC

Antes de finalizar a arquitetura, estes pontos precisam de confirmação:

**[❓ WWDC] PCC routing:** o app controla quando vai para PCC ou é automático?
Você consegue forçar que o Validation Auditor (reasoning pesado) use Cloud Pro?
Ou o sistema decide baseado na complexidade da task?

**[❓ WWDC] Tool calling no PCC:** o AFM 3 Cloud Pro aceita ferramentas customizadas
(`busca_web`, `ler_artefato`)? Tool calling no PCC funciona igual ao on-device?
Se não, o Benchmark Agent (que precisa de web search) não pode rodar via PCC.

**[❓ WWDC] Python SDK + Linux + PCC:** o Python SDK tem acesso ao PCC, ou só
aos modelos on-device? Se só on-device, os agentes precisam rodar em Swift no iOS,
não em Python no Railway. Isso é uma mudança de linguagem de implementação.

**[❓ WWDC] Context window do Cloud Pro:** qual o limite de contexto do AFM 3 Cloud Pro?
O Validation Auditor precisa ler o `project_state.json` completo. Se o context window
for menor que o Claude Sonnet, pode ser um blocker para alguns agentes.

### Mapa provisório de modelos por agente (a confirmar após WWDC)

| Agente | Modelo primário | Fallback | Condição de fallback |
|---|---|---|---|
| Benchmark Agent | AFM 3 Cloud Pro (PCC) | Claude Haiku | Se tool calling não funcionar no PCC |
| Market Sizing Agent | AFM 3 Cloud Pro (PCC) | Claude Haiku | Se tool calling não funcionar no PCC |
| Validation Auditor | AFM 3 Cloud Pro (PCC) | Claude Sonnet | Se context window insuficiente ou reasoning fraco |
| Development Auditor | AFM 3 Core Advanced (on-device) | Claude Haiku | Se context window insuficiente |
| Prompt Writers | AFM 3 Core Advanced (on-device) | Claude Haiku | Se qualidade de cards for fraca |
| UI parsing | AFM 3 Core (on-device) | — | Sempre on-device |

**O que BYOK ainda serve**: usuários técnicos que queiram trazer chave Anthropic própria
como alternativa ao fallback automático. Opcional, não requisito.

---

## Padrões de agente nesta arquitetura

Com a decisão tomada, como os padrões que você estudou se encaixam:

### CrewAI no Railway (Opção B — agentes no servidor)

```python
# Railway: src/agents/benchmark_agent.py
from crewai import Agent, Task, Crew, Process
import anthropic

benchmark_agent = Agent(
    role="Analista de Competidores",
    goal="Extrair features, preços e gaps dos top 3 competidores da ideia {idea}",
    backstory="...",
    tools=[web_search_tool, extract_tool],
    llm="claude-haiku-4-5"   # Haiku para extração
)

benchmark_task = Task(
    description="Pesquise os top 3 competidores para {idea}",
    expected_output="JSON com competitor_name, key_features, pricing, gaps para cada um",
    agent=benchmark_agent
)

# Crew orquestrado, não LLM-as-orchestrator
benchmark_crew = Crew(
    agents=[benchmark_agent],
    tasks=[benchmark_task],
    process=Process.sequential
)
```

### ADK no Railway (alternativa para pipelines com estado)

```python
# Railway: src/agents/pipeline.py
from google.adk.agents import SequentialAgent, LlmAgent
from google.adk.sessions import InMemorySessionService

# Session state é o project_state.json em memória durante a sessão
# Persiste no Supabase antes e depois
pipeline = SequentialAgent(
    name="founderlens_pipeline",
    sub_agents=[
        benchmark_agent,      # Discover
        validation_auditor,   # Midpoint Audit
        development_auditor,  # Dev Audit
    ]
)
```

### Foundation Models no iOS (UI layer)

```swift
// iOS: Views/IdeaInputView.swift
import FoundationModels

class IdeaAnalyzer {
    private let session = LanguageModelSession()  // on-device, grátis

    func quickClassify(_ text: String) async -> IdeaType {
        // Roda on-device — nenhuma chamada de rede
        let response = try? await session.respond(
            to: "Classifique em: dor_real | feature_desejada | ideia_mercado | outro. Texto: \(text)"
        )
        return IdeaType(from: response?.content ?? "outro")
    }
}
```

### Realtime: Railway → Supabase → iOS

```python
# Railway: o pipeline emite eventos conforme progresa
from supabase import create_client

supabase = create_client(SUPABASE_URL, SUPABASE_KEY)

def emit_progress(project_id: str, phase: str, status: str):
    supabase.table("pipeline_events").insert({
        "project_id": project_id,
        "phase": phase,
        "status": status
    }).execute()
```

```swift
// iOS: recebe eventos em tempo real sem polling
supabase.realtime
    .channel("pipeline_\(projectId)")
    .on(.insert) { event in
        self.updatePhaseUI(event)
    }
    .subscribe()
```

---

## EvalSet antes de construir (padrão ADK aplicado)

Antes de escrever o código do servidor, você precisará de um EvalSet por agente.
O padrão é o que você estudou na lição 05:

```python
# evals/benchmark_agent_evals.json
{
  "eval_set_id": "benchmark_agent_v1",
  "eval_cases": [
    {
      "eval_id": "strong_project_travel_b2b",
      "conversation": [
        {
          "role": "user",
          "content": "Analise competidores para: plataforma B2B de gestão de viagens corporativas"
        }
      ],
      "reference_answer": {
        # Forte: mercado definido, concorrentes conhecidos (Navan, TravelPerk, Concur)
        "expected_competitors": ["Navan", "TravelPerk", "SAP Concur"],
        "expected_gap_identified": true
      }
    },
    {
      "eval_id": "weak_project_vague_idea",
      "conversation": [
        {"role": "user", "content": "Analise competidores para: app de bem-estar"}
      ],
      "reference_answer": {
        # Fraco: ideia vaga, muitos competidores possíveis — agente deve pedir clareza
        "expected_behavior": "ask_for_clarification",
        "should_not_hallucinate_competitors": true
      }
    }
  ]
}
```

Crie 3 casos por agente (forte / médio / fraco) antes de escrever o prompt.
A lição 05 explicou por quê: você valida o agente antes de estabilizar o prompt,
não depois de já ter código rodando em prod.

---

## Mapeamento ao FounderLens

| Componente | Onde roda | Framework/Tech | Custo |
|---|---|---|---|
| UI parsing (texto do founder) | On-device | AFM 3 Core (Apple FM) | $0 |
| Benchmark Agent | Railway | CrewAI + Claude Haiku | ~$0.02/sessão |
| Market Sizing Agent | Railway | CrewAI + Claude Haiku | ~$0.01/sessão |
| Validation Auditor | Railway | CrewAI + Claude Sonnet | ~$0.08/sessão |
| Development Auditor | Railway | CrewAI + Claude Sonnet | ~$0.10/sessão |
| Prompt Generator Squad | Railway | CrewAI + Claude Haiku | ~$0.05/sessão |
| Auth + Database | Supabase | Postgres + Auth | $25/mês fixo |
| Storage (artefatos) | Supabase | Storage | incluído |
| Realtime progress | Supabase Realtime | WebSockets | incluído |

**Custo total estimado para 100 usuários ativos/mês com 5 sessões cada:**
- 500 sessões × $0.26 médio = **$130 Anthropic**
- Railway: ~$10/mês
- Supabase: $25/mês
- **Total: ~$165/mês**

Com 100 usuários a $15/mês = $1.500 receita → margem bruta de ~89%.

---

## Padrão a extrair

### 1. Separar o "onde pensa" do "onde coordena"

O agente que pensa (LLM call) e o código que coordena (loop, state, handoff) são
coisas diferentes. No Railway, o orchestrator (CrewAI Crew ou ADK SequentialAgent)
é código Python determinístico. O LLM é chamado apenas quando necessário.
Não confunda "o servidor roda o agente" com "o servidor é o agente".

### 2. On-device para latência, servidor para capacidade

O Foundation Models framework on-device elimina latência de rede para UX.
O servidor com Claude elimina limitações de raciocínio e contexto para o pipeline.
Use cada um para o que é bom: não force o on-device a fazer raciocínio complexo,
não mande texto simples de classificação para o servidor.

### 3. project_state.json como truth source

O banco de dados tem linhas. O FounderLens tem estado de projeto. A melhor abstração
é um campo JSONB no Supabase que contém o `project_state.json` completo por projeto.
O Railway carrega esse JSON no início de cada fase, modifica, e salva de volta.
Isso é mais simples do que normalizar o estado em 15 tabelas relacionais.

### 4. Emitir eventos, não fazer polling

O pipeline do servidor dura minutos. A UI iOS não deve ficar em polling.
Supabase Realtime com insert triggers resolve isso: o servidor insere eventos,
a UI iOS recebe via WebSocket. O founder vê progresso em tempo real sem infraestrutura
adicional.

### 5. BYOK como desconto, não como requisito

A arquitetura server-side permite BYOK opcional. O app verifica se o usuário tem
chave própria configurada; se sim, usa ela (e cobra menos); se não, usa a chave
do FounderLens. A lógica de roteamento é uma linha:

```python
api_key = user.byok_key or os.environ["FOUNDERLENS_ANTHROPIC_KEY"]
client = anthropic.Anthropic(api_key=api_key)
```

---

## Ponto de checagem

1. A Apple lançou cinco modelos no WWDC 2026, não apenas um. Qual modelo você usaria
   para a **camada de UI** do FounderLens iOS (parsing de input enquanto o founder
   digita) e por quê? E qual você usaria para o **Validation Auditor** — e onde ele
   roda?

2. O custo estimado por sessão completa do pipeline varia entre $0.11 (Haiku puro) e
   $0.40 (Sonnet puro). A estratégia de modelo misto visa ~$0.25. Quais agentes do
   FounderLens devem usar Haiku e quais devem usar Sonnet — e qual é o critério para
   essa distinção?

3. Railway foi escolhido sobre Cloudflare Workers para o runtime de agentes.
   Qual é a razão técnica principal — e o que o Workers teria de resolver para mudar
   essa decisão?

4. O `LanguageModel` protocol da Apple (2026) permite trocar de modelo com minimal
   code changes. Isso torna a Opção C "Apple FM híbrido" mais atraente que era antes.
   Mas a recomendação ainda é B (servidor) para o pipeline. Qual é a razão que persiste,
   independente de qual modelo a Apple lance?

5. O padrão "emitir eventos, não fazer polling" resolve um problema real da UX do
   FounderLens. Qual é o problema — e como o Supabase Realtime resolve sem que você
   precise construir infraestrutura de WebSocket própria?

---

## Próximo

→ **[09 — FounderLens Agent Map](./09-founderlens-agent-map.md)**
Conecta tudo ao produto. Para cada agente: padrão, ferramentas, onde roda, como
passa contexto. O grafo de dependências completo. Entrada para a primeira semana de código.

## Relacionados

- [[PLANO-DE-ESTUDOS]] — as três opções A/B/C que este doc fecha
- [[05-google-adk-patterns]] — ADK que roda no Railway
- [[04-crewai-patterns]] — CrewAI que roda no Railway
- [[prompt-generator-squad]] — wave execution no Deliver
- [[validation-auditor]] — o agente mais exigente do pipeline (precisa Sonnet)
- [[MOC-agentes]] — mapa completo

---

*Última atualização: junho 2026*
