---
tags: [frameworks, google-adk, multi-agente, evaluation, state-machine, módulo]
aliases: [Google ADK, ADK patterns, Agent Development Kit, Agent-as-Tool]
módulo: "05"
anterior: "[[04-crewai-patterns]]"
próximo: "06-multica-openspec-map"
---

# 05 — Google ADK: composição estrutural e avaliação
**Frameworks como professores de padrão · v1.0 · junho 2026**

---

## Por que ADK depois do CrewAI

CrewAI te deu o modelo de squads. Mas tem dois problemas que ele não resolve:

**Problema 1 — Orquestração ainda depende de escolhas do framework.**
`Process.sequential` parece código, mas é uma decisão do CrewAI sobre como passar
contexto entre tasks. O Flows (`@listen`) é melhor, mas você ainda está declarando
fluxo dentro do vocabulário do framework. Se o framework mudar, muda o contrato.

**Problema 2 — Não tem avaliação sistemática.**
Você pode escrever agentes com CrewAI, lançar, e nunca saber se o
`validation-auditor` está dando scores corretos. Não há mecanismo nativo para
testar agentes contra casos conhecidos e medir qualidade.

O ADK (Google Agent Development Kit, 2024) resolve os dois:
- Composição é **estrutural em código** — você usa classes Python, não declarações de framework
- Tem **evaluation loop nativo** — você testa agentes como testa funções

Esses dois padrões são os que mais importam para o FounderLens.

---

## O modelo de árvore — não uma crew

CrewAI organiza agentes em uma `Crew` (um time plano com um processo).
ADK organiza agentes em uma **árvore de composição**:

```
              ┌─────────────────────────────┐
              │    SequentialAgent          │  ← o pipeline inteiro
              │    (Double Diamond)         │
              └──────────┬──────────────────┘
                         │
          ┌──────────────┼──────────────┐
          │              │              │
    LlmAgent       ParallelAgent   LlmAgent
    (Discover)     (Midpoint Audit) (Develop)
                        │
              ┌─────────┴─────────┐
              │                   │
          LlmAgent            LlmAgent
          (Market Sizing)   (Validation Auditor)
```

A diferença crucial: **a estrutura é código, não configuração**.
Um `SequentialAgent` é uma classe Python que você instancia.
Não há runtime do framework decidindo a ordem — a ordem está na estrutura do objeto.

Quatro tipos de nó na árvore:

| Tipo | O que faz |
|---|---|
| `LlmAgent` | Roda um LLM com tools. É a folha da árvore. |
| `SequentialAgent` | Roda sub-agentes em ordem. Output de um vira input do próximo via `session.state`. |
| `ParallelAgent` | Roda sub-agentes concorrentemente. Barreira implícita quando todos terminam. |
| `LoopAgent` | Roda sub-agentes em loop até condição de saída ou `max_iterations`. |

---

## LlmAgent — a folha da árvore

```python
from google.adk.agents import LlmAgent
from google.adk.tools import ToolContext

def buscar_concorrentes(query: str, tool_context: ToolContext) -> str:
    """
    Busca concorrentes do produto descrito na query.
    Use quando precisar de dados sobre empresas que resolvem o mesmo problema.
    Retorna uma lista de concorrentes com preços e features.
    """
    resultado = chamar_api_busca(query)
    # state é persistente na sessão inteira
    tool_context.state["ultima_busca"] = query
    return resultado

benchmark_agent = LlmAgent(
    name="benchmark_agent",
    model="gemini-2.0-flash",          # ou qualquer LLM via LiteLLM
    instruction="""
        Você é um analista de inteligência competitiva especializado em
        consumer apps na América Latina.
        
        Seu objetivo: mapear os 3 principais concorrentes do produto descrito
        no contexto, com preços, features principais e gaps exploráveis.
        
        Você é cético com claims de marketing. Só registra o que consegue
        verificar em fontes primárias. Toda afirmação precisa de fonte.
    """,
    tools=[buscar_concorrentes, ler_contexto_projeto],
    output_key="benchmark_result",   # ← escreve output em session.state["benchmark_result"]
)
```

`instruction` é o system prompt. Você percebe que é o mesmo que o CrewAI chama de
`role + goal + backstory` — ADK não decompõe em campos, mas o padrão de separar
identidade / critério / comportamento dentro do prompt ainda se aplica.

`output_key` é o que conecta agentes na árvore: o output do `benchmark_agent`
vai automaticamente para `session.state["benchmark_result"]`, onde o próximo
agente pode lê-lo.

---

## SequentialAgent — pipeline em código

```python
from google.adk.agents import SequentialAgent

midpoint_audit = SequentialAgent(
    name="midpoint_audit",
    sub_agents=[
        market_sizing_agent,    # roda primeiro
        validation_auditor,     # roda depois, lê output do market_sizing
    ]
)
```

O `SequentialAgent` não usa LLM. É Python puro orquestrando outros agentes.
A ordem é determinística — definida por você na lista `sub_agents`.

Como o contexto flui: cada agente lê de `session.state` e escreve via `output_key`.
O `market_sizing_agent` escreve `session.state["market_sizing_result"]`.
O `validation_auditor` lê esse campo ao rodar.

Compare com CrewAI `Process.sequential`: lá o framework injeta o output da task
anterior automaticamente no prompt. Aqui você lê explicitamente de `session.state`.
Mais verboso, mais auditável.

---

## ParallelAgent — wave execution estrutural

```python
from google.adk.agents import ParallelAgent

wave_1 = ParallelAgent(
    name="wave_1",
    sub_agents=[
        prompt_writer_db,
        prompt_writer_auth,
        prompt_writer_design,
    ]
)

wave_2 = ParallelAgent(
    name="wave_2",
    sub_agents=[
        prompt_writer_api_voos,   # depende de db → só na wave 2
        prompt_writer_api_claims,
    ]
)

squad_pipeline = SequentialAgent(
    name="prompt_generator_squad",
    sub_agents=[wave_1, wave_2, wave_3]  # barreira entre waves é a estrutura
)
```

Isso **é** a wave execution da spec do [[prompt-generator-squad]], expressa como
estrutura de objetos Python. A barreira entre waves não é um `asyncio.gather` —
é o fato de que `wave_2` está depois de `wave_1` no `SequentialAgent`.

O grafo de dependências vira a árvore de composição. Dependências explícitas em
código, não inferidas pelo framework.

---

## Agent-as-Tool — o padrão central do ADK

Este é o conceito que o ADK formaliza melhor do que qualquer outro framework.

**Ideia:** qualquer agente pode ser transformado em ferramenta e chamado por outro
agente como se fosse uma função.

```python
from google.adk.tools import AgentTool

# O benchmark_agent existe como agente independente
# Mas pode ser envolto como ferramenta:
benchmark_tool = AgentTool(agent=benchmark_agent)

# O validation_auditor pode agora chamar o benchmark como tool:
validation_auditor = LlmAgent(
    name="validation_auditor",
    instruction="""
        Você aplica a rubrica R-W-W (Real / Worth it / Winnable) ao projeto.
        
        Para a dimensão 'Winnable', você tem acesso a benchmark_tool — use-o
        para buscar dados reais de concorrentes antes de avaliar.
        
        Retorne score 0-100 com justificativa por dimensão e veredito final.
    """,
    tools=[
        benchmark_tool,          # ← outro agente, chamado como ferramenta
        ler_contexto_projeto,
        busca_web,
    ],
    output_key="audit_result",
)
```

Quando o `validation_auditor` chama `benchmark_tool`, o `benchmark_agent` roda
seu loop ReAct completo — busca web, itera, produz output estruturado — e retorna
o resultado como se fosse o retorno de uma função.

**Por que isso importa para o FounderLens:**

O `validation_auditor` não precisa saber como o benchmark funciona. Ele declara
"preciso de dados de concorrentes" e chama a ferramenta. O orquestrador não precisou
decidir nada — o próprio auditor sabe quando precisa de benchmark.

**Compare com CrewAI `allow_delegation=True`:**

| | CrewAI delegation | ADK Agent-as-Tool |
|---|---|---|
| Quem delega | O agente decide livremente para quem delegar | O agente chama a ferramenta explícita |
| O que é delegado | Qualquer coisa, para qualquer agente da crew | Exatamente a capacidade declarada no AgentTool |
| Previsibilidade | Baixa — o agente pode delegar inesperadamente | Alta — você vê no código quais tools o agente tem |
| Debug | Difícil — "por que delegou isso?" | Simples — está no trace de tools chamadas |

Agent-as-Tool é delegação com contrato explícito.

---

## Session.state — o project_state.json formalizado

No ADK, toda a execução acontece dentro de uma `Session`. O `session.state` é um
dicionário persistente que sobrevive durante a sessão inteira:

```python
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService

session_service = InMemorySessionService()
runner = Runner(
    agent=pipeline_completo,
    app_name="founderlens",
    session_service=session_service,
)

# Inicia sessão com dados do founder
session = await session_service.create_session(
    app_name="founderlens",
    user_id="founder_lucas",
    state={                          # ← estado inicial
        "projeto": {
            "nome": "AeroClaim",
            "problema": "passageiros não sabem reivindicar compensação...",
        }
    }
)
```

Dentro de qualquer ferramenta, `ToolContext.state` é o mesmo dicionário:

```python
def salvar_benchmark(dados: dict, tool_context: ToolContext) -> str:
    tool_context.state["benchmark_result"] = dados
    tool_context.state["fase_atual"] = "define"   # ← tracking de fase
    return "Benchmark salvo com sucesso."
```

E qualquer agente pode ler:

```python
def ler_contexto_projeto(tool_context: ToolContext) -> dict:
    return tool_context.state.get("projeto", {})
```

**O padrão que isso valida:** o `project_state.json` que você desenhou na spec
não é uma invenção — é exatamente como frameworks de produção gerenciam estado.
A diferença é que a spec usa arquivo JSON em disco; ADK usa memória (ou banco via
`DatabaseSessionService`). O padrão é idêntico.

**Dois tipos de estado na sessão:**

| Tipo | Onde fica | Lifetime | Equivalente no FounderLens |
|---|---|---|---|
| `session.state` | Dicionário mutável | Duração da sessão | `project_state.json` |
| `session.events` | Lista imutável de mensagens | Duração da sessão | Histórico de turns |

`session.state` é para dados estruturados que agentes leem e escrevem.
`session.events` é para o histórico de conversação — não modifique diretamente.

---

## LoopAgent — retry e state machine

O `LoopAgent` é o mecanismo de retry e state machine do ADK:

```python
from google.adk.agents import LoopAgent

def verificar_qualidade_output(tool_context: ToolContext) -> dict:
    """
    Verifica se o output do benchmark atende ao critério de qualidade.
    Retorna escalate=True se o output for aprovado.
    """
    benchmark = tool_context.state.get("benchmark_result", {})
    
    tem_fontes = len(benchmark.get("fontes", [])) >= 3
    tem_concorrentes = len(benchmark.get("concorrentes", [])) >= 2
    
    if tem_fontes and tem_concorrentes:
        tool_context.actions.escalate = True   # ← sinal de saída do loop
        return {"aprovado": True}
    
    return {"aprovado": False, "motivo": "Fontes ou concorrentes insuficientes"}

verificador_agent = LlmAgent(
    name="verificador",
    instruction="Verifique a qualidade do benchmark em session.state e chame verificar_qualidade_output.",
    tools=[verificar_qualidade_output],
)

benchmark_com_retry = LoopAgent(
    name="benchmark_com_retry",
    sub_agents=[
        benchmark_agent,    # executa o benchmark
        verificador_agent,  # verifica qualidade, seta escalate se aprovado
    ],
    max_iterations=3,       # ← equivalente ao max_iter do CrewAI e MAX_ITERACOES do doc 02
)
```

O ciclo:
```
iter 1: benchmark_agent roda → verificador_agent verifica → não aprovado → continua
iter 2: benchmark_agent roda de novo (lê o que faltou) → verificador verifica → aprovado → escalate=True → para
```

**Por que `LoopAgent` é melhor que manager LLM para retry:**

O manager LLM do CrewAI `Process.hierarchical` re-delega quando acha que o output
é ruim — mas "achar que é ruim" é uma decisão não-determinística de um LLM.
`LoopAgent` + `verificador_agent` com `tool_context.actions.escalate` é:
- Determinístico: a condição de saída é código Python
- Auditável: você vê exatamente por que o loop parou
- Barato: o verificador pode ser um modelo pequeno ou até uma função pura

Essa é a mesma lógica do [[max-iterations]] e do [[loop-explicito-vs-abstraido]] —
só que agora aplicada ao nível de multi-agente, não de um agente único.

---

## Evaluation — o que faltava no CrewAI

Este é o padrão mais importante que o ADK traz e que nenhum outro framework tem
de forma nativa.

**O problema:** você escreve o `validation_auditor`, testa manualmente uma vez,
parece funcionar. Muda o prompt. Testa de novo. "Parece bom." Como você sabe que
não quebrou o que funcionava? Como você sabe que o score é calibrado?

**A solução — EvalSet:**

```python
from google.adk.evaluation import EvalSet, EvalCase, AgentEvaluator

# Casos de teste — outputs que você sabe que são corretos
eval_set = EvalSet(
    eval_id="validation_auditor_calibration_v1",
    eval_cases=[
        EvalCase(
            eval_id="caso_app_voos_forte",
            conversation=[
                {"role": "user", "content": "Avalie este projeto: app de reivindicação..."}
            ],
            expected_final_response="score_total entre 70-80, veredicto proceed_with_caution",
            reference_final_response={
                "score_total": 74,
                "veredicto": "proceed_with_caution",
                "dimensoes": {
                    "real": {"score": 85},
                    "worth_it": {"score": 70},
                    "winnable": {"score": 68},
                }
            }
        ),
        EvalCase(
            eval_id="caso_ideia_vaga_fraca",
            conversation=[
                {"role": "user", "content": "Avalie: quero fazer um app de receitas..."}
            ],
            expected_final_response="score abaixo de 50, veredicto revisit",
        ),
    ]
)

# Roda avaliação
evaluator = AgentEvaluator()
result = await evaluator.evaluate(
    agent=validation_auditor,
    eval_set=eval_set,
    config=EvaluationConfig(
        criteria=["tool_trajectory_avg_score", "response_match_score"]
    )
)

print(result.summary())
# tool_trajectory_avg_score: 0.83  ← usou as ferramentas certas na ordem certa?
# response_match_score: 0.71       ← output condiz com o esperado?
```

**Duas métricas centrais:**

`tool_trajectory_avg_score` — o agente usou as ferramentas certas, na ordem certa?
Para o `validation_auditor`: primeiro `ler_contexto_projeto`, depois `busca_web`
para market data, depois `benchmark_tool` para Winnable. Se a ordem for errada,
o score cai.

`response_match_score` — o output final condiz com o esperado? Para o auditor:
score na faixa correta, veredito correto, dimensões com justificativa.

**Como isso muda o desenvolvimento:**

Sem evaluation:
```
escreve agente → testa na mão → "parece bom" → sobe para produção → founder reclama que score está errado
```

Com evaluation:
```
escreve agente → roda eval_set → score 0.45 → ajusta prompt → roda de novo → score 0.82 → sobe para produção
```

Evaluation transforma o desenvolvimento de agentes de arte em engenharia.

**Padrão a extrair:** mesmo sem ADK, estruture seus agentes com um conjunto de
casos de teste desde o início. Para o [[validation-auditor]]: pelo menos 3 casos
(projeto forte / médio / fraco) com outputs de referência. Antes de alterar o
prompt, valide que os 3 casos ainda produzem outputs dentro da faixa esperada.
Isso é o mesmo que um suite de testes unitários — só que o "teste" é a qualidade
do output do LLM.

---

## Callbacks — observabilidade granular por agente

ADK tem callbacks por agente, mais granulares que LangChain:

```python
from google.adk.agents import LlmAgent
from google.adk.agents.callback_context import CallbackContext
from google.adk.models import LlmResponse

def log_inicio_agente(callback_context: CallbackContext) -> None:
    agente = callback_context.agent_name
    estado = callback_context.state.get("fase_atual", "desconhecida")
    print(f"[{agente}] iniciando | fase: {estado} | ts: {time.time()}")

def log_fim_agente(
    callback_context: CallbackContext,
    llm_response: LlmResponse
) -> LlmResponse | None:
    agente = callback_context.agent_name
    tokens = llm_response.usage_metadata.total_token_count
    print(f"[{agente}] fim | tokens: {tokens}")
    return None   # None = não modifica o response

benchmark_agent = LlmAgent(
    name="benchmark_agent",
    instruction="...",
    tools=[...],
    before_agent_callback=log_inicio_agente,
    after_agent_callback=log_fim_agente,
    before_tool_callback=log_tool_chamada,
    after_tool_callback=log_tool_resultado,
)
```

Diferença para LangChain: callbacks são por instância de agente, não globais.
Você pode ter logging detalhado no `benchmark_agent` e logging mínimo no
`verificador_agent`. Isso importa quando você quer medir custo por componente —
o benchmark provavelmente é o mais caro e precisa de mais instrumentação.

---

## Handoff explícito — transfer_to_agent

Para multi-agente onde um agente precisa passar controle para outro (não sub-chamar
como ferramenta, mas realmente transferir), ADK tem `transfer_to_agent` como
ferramenta built-in:

```python
root_agent = LlmAgent(
    name="root",
    instruction="""
        Você é o orquestrador do FounderLens.
        Quando o founder completar o Discover, transfira para o define_agent.
        Quando o define estiver completo, transfira para o midpoint_audit.
    """,
    sub_agents=[discover_agent, define_agent, midpoint_audit],
    # transfer_to_agent é adicionado automaticamente quando sub_agents existe
)
```

**Mas — a mesma decisão do CrewAI se aplica aqui.**

`transfer_to_agent` é um LLM decidindo quando transferir. Para o FounderLens,
a progressão entre fases não deve ser uma decisão de LLM — é uma decisão do
founder ("estou pronto para avançar"). O founder clica em "continuar" na UI.
A UI chama o próximo agente. Não precisa de `transfer_to_agent`.

Use `transfer_to_agent` apenas quando a decisão de "quando avançar" genuinamente
depende de análise de conteúdo. Por exemplo: se o `validation_auditor` puder
decidir autonomamente "este projeto precisa voltar para Discover" (sem interação
do founder), aí faz sentido.

No FounderLens atual, toda progressão de fase passa pelo founder. Orquestração
explícita em código.

---

## A convergência dos três frameworks

Você estudou três frameworks. Eles chegaram ao mesmo lugar por caminhos diferentes:

```
LangChain (2022) → Chain → LCEL → LangGraph
  percebeu que pipeline linear não basta → adicionou grafo de estados

CrewAI (2023) → Crew/Process → Flows
  percebeu que LLM-orchestrator não é confiável → adicionou controle explícito

ADK (2024) → árvore de composição + evaluation
  nasceu já com controle explícito + avaliação nativa
```

Todos convergiram para: **orquestração determinística em código, LLM nas folhas,
avaliação sistemática de qualidade**.

Isso é exatamente o que a spec do FounderLens já desenhava — sem saber que era
o estado da arte.

---

## Padrão a extrair pro FounderLens

### 1. Árvore de composição = sua estrutura já em código

O pipeline `Discover → Define → Audit → Develop → Deliver` é um `SequentialAgent`.
Dentro do Midpoint Audit, `market_sizing` e `validation_auditor` podem ser um
`ParallelAgent` (rodam com os mesmos dados, sem dependência entre si).
Dentro do Deliver, as waves do squad são `SequentialAgent([ParallelAgent([...]), ParallelAgent([...]), ...])`.

Você não precisa do ADK para implementar isso — a estrutura de dados (lista de agentes,
sequencial vs paralelo) pode existir em Swift puro. O padrão é o objeto, não o framework.

### 2. Agent-as-Tool = auditor com superpoderes

O `validation_auditor` deve ter `benchmark_tool` como uma de suas ferramentas.
Quando avaliar a dimensão Winnable, o auditor chama o benchmark automaticamente —
sem o orquestrador precisar decidir isso. Cada auditor tem autonomia dentro de sua
responsabilidade única.

### 3. session.state = project_state.json em memória

O estado da sessão deve ser um dicionário com schema definido. Cada agente lê as
chaves que precisa, escreve as chaves que produz, nunca sobrescreve o que não é
sua responsabilidade. Isso é o que o `project_state.json` já é — só formalizar
quem pode escrever o quê.

### 4. EvalSet desde o início

Para cada agente do FounderLens, defina pelo menos 3 casos de teste antes de
lançar. Para o `validation_auditor`: projeto forte (score >80), médio (60-80),
fraco (<60). Para o `benchmark_agent`: produto com 3 concorrentes claros, produto
com mercado difuso. Rodar os evals antes de alterar qualquer prompt é o que
separa manutenção de regressão acidental.

### 5. LoopAgent = retry com condição em código

Retry não é "tentou de novo com esperança". É "tentou de novo com critério claro
de quando parar". Para qualquer agente que pode falhar (benchmark com dados ruins,
validation com output malformado), o retry deve ter: critério de sucesso em código,
máximo de iterações, e logging de por que cada tentativa falhou.

---

## Mapeamento ao FounderLens

| Conceito ADK | No FounderLens | Implementação |
|---|---|---|
| `LlmAgent` | Cada agente da spec | Anthropic API + system prompt em 3 blocos |
| `SequentialAgent` | Pipeline Discover → Define → Audit → Develop → Deliver | Orquestração explícita em código |
| `ParallelAgent` | Wave execution do squad / Market Sizing + Validation em paralelo | `asyncio.gather()` por wave |
| `LoopAgent` | Retry com critério de saída | Loop explícito com `max_iter` + verificação em código |
| `Agent-as-Tool` | `validation_auditor` chama `benchmark_agent` como tool | `AgentTool(agent=...)` ou sub-chamada explícita |
| `output_key` | Escrita estruturada no `project_state.json` | Campo definido por schema, escrito só pelo agente dono |
| `session.state` | `project_state.json` | Dicionário em memória (ou arquivo JSON em disco) |
| `session.events` | Histórico de mensagens | Array de turns, nunca modificado diretamente |
| `EvalSet` | Suite de testes de qualidade por agente | Casos de referência por fase |
| `tool_trajectory_avg_score` | Auditoria de sequência de tools usadas | Log estruturado com `callbacks` |
| `response_match_score` | Output dentro do schema esperado | Validação Pydantic antes de escrever no state |
| `before_agent_callback` | Log de início por agente com timestamp | `log_evento()` no ponto de entrada |
| `transfer_to_agent` | ❌ não usar para progressão de fase | Progressão controlada pelo founder via UI |

---

## Ponto de checagem

1. Qual a diferença entre `Agent-as-Tool` e `allow_delegation=True` do CrewAI?
   Por que Agent-as-Tool é mais previsível em produção?

2. Como `session.state` difere do histórico de mensagens (`session.events`)?
   O `project_state.json` do FounderLens é mais parecido com qual dos dois, e por quê?

3. Como você expressaria a wave execution do [[prompt-generator-squad]] como árvore
   de composição ADK? Desenhe a estrutura (`SequentialAgent`, `ParallelAgent`, etc.)

4. Você tem um `validation_auditor` funcionando. O founder reclama que o score
   está alto demais para projetos fracos. Sem evaluation, como você saberia se a
   correção do prompt resolveu o problema sem criar novos? Com `EvalSet`, como você
   saberia?

5. Em qual situação `transfer_to_agent` seria a escolha certa no FounderLens?
   (Pense em um cenário onde a decisão de avançar de fase genuinamente dependeria
   de julgamento sobre conteúdo, não de input do founder.)

---

## Próximo

→ **[06 — Multica + OpenSpec: o que a competição construiu](./06-multica-openspec-map.md)**
Dois projetos open-source que já implementaram partes do que o FounderLens quer
construir. O que cada um fez, onde falharam, e o que o FounderLens faz diferente.
Inclui análise do módulo M7 (Prompts) do OpenSpec como competidor direto do Deliver.

## Relacionados

- [[react-pattern]] — `LlmAgent` roda este loop internamente
- [[loop-explicito-vs-abstraido]] — `LoopAgent` é a versão multi-agente da mesma decisão
- [[state-memoria]] — `session.state` = estado de longa duração; `session.events` = curta
- [[max-iterations]] — `max_iterations` do `LoopAgent` é o mesmo guardrail
- [[callbacks]] — `before_agent_callback` é onde você mede custo por agente
- [[tool-description-as-prompt]] — docstring da tool ADK tem a mesma função
- [[prompt-generator-squad]] — a wave execution que `ParallelAgent` + `SequentialAgent` modela
- [[validation-auditor]] — candidato principal a Agent-as-Tool + EvalSet
- [[benchmark-agent]] — candidato a ser chamado como `AgentTool` pelo auditor
- [[MOC-agentes]] — mapa completo do grafo

---

*Última atualização: junho 2026*
