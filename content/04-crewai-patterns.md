---
tags: [frameworks, crewai, multi-agente, squad, módulo]
aliases: [CrewAI patterns, Crew, squads de agentes, role-based agents]
módulo: "04"
anterior: "[[03-langchain-patterns]]"
próximo: "05-google-adk-patterns"
---

# 04 — CrewAI: o modelo de squads
**Frameworks como professores de padrão · v1.0 · junho 2026**

---

## Por que este é o módulo mais direto para o produto

LangChain te deu o vocabulário. CrewAI te dá o **modelo mental que o FounderLens já usa
sem saber**: agentes com papéis definidos, tarefas com output esperado, e um orquestrador
que decide a ordem.

Releia a spec do [[prompt-generator-squad]]: "Orquestrador → múltiplos Prompt Writers
em paralelo, com dependency awareness". Isso é literalmente uma `Crew` com
`Process.hierarchical` e tasks assíncronas. CrewAI formalizou o padrão que você
desenhou intuitivamente — estudá-lo valida e refina a spec.

---

## O que é CrewAI

Framework Python para orquestrar múltiplos agentes que colaboram. Criado em 2023
sobre o vocabulário do LangChain (doc 03), com uma tese própria: **agentes funcionam
melhor quando modelados como funcionários de uma empresa** — cada um com papel,
objetivo e história.

Três abstrações centrais + um motor:

```
Agent   → quem trabalha (role + goal + backstory + tools)
Task    → o que precisa ser feito (description + expected_output)
Crew    → o time montado (agents + tasks + process)
Process → como o trabalho flui (sequential | hierarchical)
```

Compare com LangChain: lá a unidade é a `Chain` (sequência de operações).
Aqui a unidade é o **agente com identidade**. É uma mudança de "pipeline de dados"
para "organograma de trabalho".

---

## Agent — role + goal + backstory

```python
from crewai import Agent

benchmark_agent = Agent(
    role="Analista de Inteligência Competitiva",
    goal="Mapear os concorrentes diretos e indiretos do produto do founder, "
         "com preços, features e posicionamento documentados com fontes",
    backstory="Você passou 10 anos analisando mercados de consumer apps na "
              "América Latina. Você é cético com claims de marketing e só "
              "registra o que consegue verificar em fontes primárias. "
              "Você sabe que founders se apaixonam pela própria ideia, então "
              "seu papel é trazer os fatos — mesmo os desconfortáveis.",
    tools=[busca_web, ler_contexto_projeto],
    llm="claude-opus-4-5",
    max_iter=8,                  # o MAX_ITERACOES do doc 02
    allow_delegation=False,      # este agente não passa trabalho adiante
    verbose=True
)
```

**O insight central: role + goal + backstory são o system prompt.**

CrewAI concatena os três campos num template interno e isso vira o system prompt
do agente. Não há mágica — é exatamente o `SYSTEM_PROMPT` que você escreveu no
doc 02, decomposto em três partes com funções distintas:

| Campo | Função no prompt | Equivalente no que você já escreveu |
|---|---|---|
| `role` | Identidade — como o LLM se enxerga | "Você é um analista de mercado..." |
| `goal` | Critério de sucesso — quando parar | "Sua tarefa termina quando..." |
| `backstory` | Comportamento — vieses e estilo desejados | "Seja cético, cite fontes..." |

**Por que a decomposição importa:** quando o system prompt é um blob de texto,
você mistura identidade com critério de parada com regras de comportamento — e
fica difícil iterar. Separado em três campos, você pode ajustar o ceticismo
(backstory) sem tocar no critério de sucesso (goal).

**Padrão a extrair:** mesmo sem CrewAI, estruture os system prompts dos agentes
do FounderLens nesses três blocos. O [[validation-auditor]] tem role (auditor R-W-W),
goal (score 0-100 + veredito) e backstory (rigoroso, fundamenta com fontes) —
hoje implícitos, melhor explícitos.

---

## Task — o contrato de output

```python
from crewai import Task
from pydantic import BaseModel

# Schema do output (Pydantic)
class BenchmarkOutput(BaseModel):
    concorrentes: list[dict]
    gaps_identificados: list[str]
    fontes: list[str]

benchmark_task = Task(
    description="Pesquise os concorrentes do produto descrito no contexto do "
                "projeto. Para cada um: modelo de cobrança, features principais, "
                "tempo de mercado. Identifique gaps que o founder pode explorar.",
    expected_output="Lista estruturada de 3-5 concorrentes com preços e features "
                    "verificados, mais 2-3 gaps de mercado com justificativa. "
                    "Toda afirmação precisa de fonte.",
    agent=benchmark_agent,
    output_pydantic=BenchmarkOutput,      # força output estruturado e validável
    output_file="benchmark_report.json"   # persiste o resultado
)
```

**O conceito novo aqui é `expected_output`.** Ele não é documentação — é injetado
no prompt da task. O LLM lê a descrição do que fazer E a descrição de como o
resultado deve se parecer. É um contrato.

Isso resolve um problema que você já viu: agente "termina" mas o output não serve
para a fase seguinte. Com `expected_output` + `output_pydantic`, o resultado:

1. É descrito em linguagem natural no prompt (o LLM sabe o alvo)
2. É validado contra um schema Pydantic (código rejeita output malformado)
3. Se a validação falha, o CrewAI re-prompta o agente automaticamente (retry — doc 02)

**Padrão a extrair:** o JSON de output do [[validation-auditor]]
(`score_total`, `dimensoes`, `veredicto`, `riscos`) deve existir em dois lugares:
descrito no prompt (expected_output) e validado em código (schema). O `project_state.json`
só aceita escrita de output que passou na validação — nunca avance fase com dados malformados.

---

## Crew + Process.sequential — o pipeline

```python
from crewai import Crew, Process

crew = Crew(
    agents=[benchmark_agent, market_sizing_agent, validation_auditor],
    tasks=[benchmark_task, sizing_task, audit_task],
    process=Process.sequential,
    verbose=True
)

resultado = crew.kickoff(inputs={"contexto_founder": contexto})
```

No modo sequential, as tasks rodam na ordem da lista e **o output de uma task
entra automaticamente no contexto da próxima**. O `audit_task` recebe o que o
benchmark e o sizing produziram sem você escrever código de handoff.

Você também pode declarar o handoff explicitamente com `context`:

```python
audit_task = Task(
    description="Aplique a rubrica R-W-W ao projeto...",
    expected_output="Score 0-100 com justificativa por dimensão...",
    agent=validation_auditor,
    context=[benchmark_task, sizing_task]   # ← recebe SÓ o output dessas duas
)
```

`context` é o controle fino: em vez de "tudo que veio antes", a task declara
exatamente de quais outputs depende. **Isso é o grafo de dependências do
[[prompt-generator-squad]]** — cada card declara seus pré-requisitos.

---

## Process.hierarchical — o orquestrador

```python
crew = Crew(
    agents=[prompt_writer_a, prompt_writer_b, prompt_writer_c],
    tasks=tasks_dos_cards,
    process=Process.hierarchical,
    manager_llm="claude-opus-4-5",   # CrewAI cria um manager agent automaticamente
    verbose=True
)
```

No modo hierarchical, o CrewAI cria um **manager agent** que:

1. Lê todas as tasks pendentes
2. Decide qual agente faz qual task (por role e capacidade)
3. Delega, recebe o resultado, avalia se está bom
4. Re-delega se o resultado for insatisfatório

```
              ┌──────────────┐
              │   Manager    │  ← criado pelo framework
              │  (delega e   │
              │   avalia)    │
              └──┬────┬────┬─┘
                 │    │    │
         ┌───────┘    │    └───────┐
         ▼            ▼            ▼
   Prompt Writer  Prompt Writer  Prompt Writer
        A              B              C
```

**O tradeoff (e ele é grande):** o manager é um LLM tomando decisões de
orquestração. Isso significa:

- Mais uma chamada LLM por decisão de delegação → custo e latência sobem
- A ordem de execução fica **não-determinística** — o manager decide na hora
- Debugging fica mais difícil: "por que o card X rodou antes do Y?" não tem
  resposta no seu código

É o mesmo dilema do [[loop-explicito-vs-abstraido]], um nível acima: lá era
o loop de UM agente abstraído; aqui é a **coordenação entre agentes** abstraída.

**Decisão para o FounderLens:** o orquestrador do [[prompt-generator-squad]] NÃO
deve ser um manager LLM. O grafo de dependências entre cards é conhecido de antemão
(vem das specs do Develop) — não há decisão inteligente a tomar, só topological sort.
Orquestração determinística em código; LLM só dentro de cada Prompt Writer.
Use manager LLM apenas quando a decisão de "quem faz o quê" genuinamente
depende do conteúdo — o que não é o caso aqui.

---

## Paralelismo — async_execution e waves

CrewAI tem dois mecanismos de paralelismo:

**1. Tasks assíncronas dentro de uma crew:**

```python
card_auth = Task(..., agent=writer_a, async_execution=True)
card_voos = Task(..., agent=writer_b, async_execution=True)
card_db   = Task(..., agent=writer_c, async_execution=True)

# Task síncrona que espera as três acima
integracao = Task(
    ...,
    agent=writer_d,
    context=[card_auth, card_voos, card_db]  # ← barreira: só roda quando as 3 terminam
)
```

Tasks com `async_execution=True` rodam em paralelo. Uma task síncrona com
`context` apontando para elas funciona como **barreira de sincronização** —
só começa quando todas terminam.

**Isso É a wave execution da spec do squad:**

```
Wave 1: [card_db] [card_auth] [card_design]     ← async_execution=True
        ────────── barreira (context) ──────────
Wave 2: [card_api_voos] [card_api_claims]       ← context=[card_db], async
        ────────── barreira (context) ──────────
Wave 3: [card_tela_voo] [card_tela_claim]       ← context=[apis], async
```

**2. Crews inteiras em paralelo:**

```python
import asyncio

# Roda a mesma crew para múltiplos inputs ao mesmo tempo
resultados = await crew.kickoff_for_each_async(inputs=[
    {"historia": "autenticação"},
    {"historia": "listagem de voos"},
    {"historia": "submissão de claim"},
])
```

`kickoff_async` / `kickoff_for_each_async` rodam execuções completas
concorrentemente — útil quando as unidades de trabalho são independentes
e idênticas em estrutura (exatamente o caso dos prompt cards de uma mesma wave).

**Atenção ao custo:** paralelismo multiplica chamadas LLM simultâneas. 8 cards
em paralelo = 8 × (tokens por card). A spec do squad já anota isso como risco.
Callbacks (doc 03) por agente são o que te permite ver o custo real por card.

---

## allow_delegation — colaboração entre agentes

```python
agente = Agent(..., allow_delegation=True)
```

Com delegação ativa, o agente ganha duas ferramentas implícitas:
`Delegate work to coworker` e `Ask question to coworker`. Ele pode, no meio
do próprio loop ReAct, decidir passar uma sub-tarefa para outro agente da crew.

**Use com cautela.** Delegação é poderosa em domínios abertos, mas no FounderLens
cada agente tem responsabilidade única por design ([[validation-auditor]] não
pesquisa concorrentes; [[benchmark-agent]] não avalia rubrica). Delegação livre
quebraria essa separação — um auditor que delega pesquisa para si mesmo via
benchmark cria caminhos de execução imprevisíveis.

`allow_delegation=False` (default atual do CrewAI) é a escolha certa para
todos os agentes do FounderLens. A colaboração acontece via **pipeline e state**
(outputs estruturados no `project_state.json`), não via conversa livre entre agentes.

---

## Nota: CrewAI Flows — o pêndulo voltando

O próprio CrewAI reconheceu o limite do modelo "crew decide tudo": lançou **Flows**,
uma camada event-driven onde VOCÊ escreve o controle de fluxo em Python decorado
(`@start`, `@listen`, `@router`) e chama crews/agentes nos pontos certos.

```python
from crewai.flow.flow import Flow, listen, start

class FounderLensFlow(Flow):
    @start()
    def discover(self):
        return benchmark_crew.kickoff(...)

    @listen(discover)
    def define(self, benchmark_result):
        return define_crew.kickoff(...)

    @listen(define)
    def midpoint_audit(self, define_result):
        return validation_auditor.kickoff(...)
```

Percebe o movimento? O framework que vendia "deixe os agentes se organizarem"
adicionou um mecanismo para **controle explícito e determinístico** — porque
produção exige previsibilidade. É a indústria inteira convergindo para a posição
que você já adotou no doc 02: loop explícito, orquestração em código, LLM nas folhas.

---

## Padrão a extrair pro FounderLens

### 1. System prompt em três blocos (role / goal / backstory)

Estruture o prompt de cada agente da spec nesses três campos, mesmo implementando
direto na Anthropic API. Itera-se comportamento sem quebrar critério de parada.

### 2. expected_output + schema = contrato de fase

Todo agente descreve o output esperado no prompt E valida contra schema em código.
Output inválido → retry. Output válido → escreve no `project_state.json`. Nunca
avance fase com dado malformado.

### 3. Orquestração determinística, inteligência nas folhas

O grafo de waves do squad é resolvido em código (topological sort), não por
manager LLM. Reserve LLM-as-orchestrator para quando a delegação genuinamente
depender de julgamento sobre conteúdo.

### 4. Paralelismo com barreiras

`async_execution` + `context` como barreira = wave execution. Implementável sem
CrewAI com `asyncio.gather()` por wave: dispara todos os cards da wave, espera
todos, avança.

```python
# Wave execution sem framework
for wave in grafo_de_waves:
    resultados = await asyncio.gather(*[
        executar_prompt_writer(card) for card in wave
    ])
    salvar_no_state(resultados)   # barreira: só avança quando todos terminam
```

---

## Mapeamento ao FounderLens

| Conceito CrewAI | No FounderLens | Implementação |
|---|---|---|
| `Agent(role, goal, backstory)` | Cada agente da spec | System prompt em 3 blocos (Anthropic API) |
| `Task(expected_output)` | Contrato de output por fase | Descrição no prompt + schema Pydantic/Codable |
| `output_pydantic` | Validação antes do state | Rejeita output malformado, retry |
| `context=[task_a, task_b]` | Grafo de dependências dos cards | `verificar_dependencias(card_id)` da spec |
| `Process.sequential` | Pipeline Discover → Define → Audit | Orquestração em código |
| `Process.hierarchical` | ❌ não usar | Orquestrador do squad é determinístico |
| `async_execution` + barreira | Wave execution do M7 | `asyncio.gather()` por wave |
| `allow_delegation` | ❌ não usar | Responsabilidade única; colaboração via state |
| `kickoff_for_each_async` | Múltiplos cards da mesma wave | Mesma estrutura, inputs diferentes |
| Flows (`@start`, `@listen`) | O pipeline inteiro do Double Diamond | Confirma a escolha pelo loop explícito |

---

## Ponto de checagem

1. CrewAI monta o system prompt a partir de três campos. Quais são, e qual a
   vantagem de mantê-los separados em vez de um blob de texto?

2. Qual problema o `expected_output` resolve que a `description` sozinha não
   resolve? Como isso se conecta à validação do `project_state.json`?

3. Por que o orquestrador do [[prompt-generator-squad]] não deve ser um
   `Process.hierarchical` com manager LLM? Em que situação um manager LLM
   seria justificado?

4. Como você implementa wave execution com barreiras usando só `asyncio`,
   sem CrewAI? Onde fica a barreira?

5. O lançamento do CrewAI Flows confirma uma decisão arquitetural que você
   já tinha tomado. Qual?

---

## Próximo

→ **[05 — Google ADK patterns](./05-google-adk-patterns.md)**
State machines, handoffs formais entre agentes, evaluation loops e Agent-as-Tool.
O ADK traz o que falta no CrewAI: avaliação sistemática de qualidade — central
para os auditores do FounderLens.

## Relacionados

- [[react-pattern]] — cada Agent do CrewAI roda este loop internamente
- [[loop-explicito-vs-abstraido]] — hierarchical manager = abstração no nível da orquestração
- [[state-memoria]] — context entre tasks = handoff via output estruturado
- [[max-iterations]] — `max_iter` do Agent é o mesmo guardrail
- [[tool-description-as-prompt]] — backstory é "description as prompt" para o próprio agente
- [[callbacks]] — essencial para medir custo de execuções paralelas
- [[prompt-generator-squad]] — a spec que este módulo valida e refina
- [[validation-auditor]] — candidato a role/goal/backstory + output_pydantic
- [[MOC-agentes]] — mapa completo do grafo


---

*Última atualização: junho 2026*
