---
tags: [frameworks, langchain, vocabulário, módulo]
aliases: [LangChain patterns, Chain, Memory, Callbacks LangChain]
módulo: "03"
anterior: "[[02-react-loop]]"
próximo: "04-crewai-patterns"
---

# 03 — LangChain: o vocabulário base
**Frameworks como professores de padrão · v1.0 · junho 2026**

---

## Por que estudar LangChain se não vou usá-lo diretamente?

Porque todo tutorial, artigo, Stack Overflow, vídeo e post de blog sobre agentes usa
a terminologia do LangChain. Se você não sabe o que é uma `Chain`, um `Callback`, ou
`ConversationBufferMemory`, você vai ficar confuso na metade de qualquer leitura técnica.

LangChain é o vocabulário compartilhado do ecossistema de agentes — não uma dependência
do FounderLens, mas uma língua que você precisa falar para navegar o campo.

Segundo ponto: os docs de frameworks mais recentes (CrewAI, ADK) assumem que você já
conhece LangChain. São camadas sobre esse vocabulário.

---

## O que é LangChain

Um framework Python (e TypeScript) para construir aplicações com LLMs. Lançado em 2022,
tornou-se o padrão de fato antes de qualquer alternativa existir.

Cinco abstrações centrais:

```
Chain   → sequência de operações
Tool    → ferramenta que o LLM pode usar (igual ao que você já conhece)
Memory  → estado que persiste entre turnos
Callback → hook que dispara em cada evento do loop
Agent   → o loop completo encapsulado
```

Você já conhece `Tool` e `Agent` por outros nomes. Agora vai aprender como o LangChain
os formaliza — e o que o LangChain adiciona que você não viu ainda.

---

## Chain — sequência de operações

A abstração fundamental do LangChain é que tudo é uma `Chain`: uma sequência de passos
que transforma input em output.

A sintaxe moderna (LCEL — LangChain Expression Language) usa o operador `|` para compor:

```python
from langchain_anthropic import ChatAnthropic
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

# Define os componentes
prompt = ChatPromptTemplate.from_messages([
    ("system", "Você é um analista de mercado."),
    ("user", "{pergunta}")
])

llm = ChatAnthropic(model="claude-opus-4-5")
parser = StrOutputParser()

# Compõe a chain com |
chain = prompt | llm | parser

# Executa
resultado = chain.invoke({"pergunta": "Quais são os maiores concorrentes de apps de direitos de passageiros?"})
```

**O que `|` faz:** o output de um componente vira input do próximo.
`prompt.invoke(vars)` → formata o prompt → passa pro `llm` → raw response → `parser` extrai o texto.

**Chains dentro de chains:**

```python
# Sub-chain que enriquece contexto
enrich_chain = busca_contexto_prompt | llm | parser

# Chain principal que usa a sub-chain
main_chain = enrich_chain | analise_prompt | llm | json_parser
```

**Padrão a extrair pro FounderLens:** cada fase do Double Diamond é uma Chain.
`Discover = perguntas_chain | benchmark_agent | handoff_chain`
O output de uma fase é o input estruturado da próxima.

---

## Tool — o decorator formal

LangChain tem um decorator `@tool` que transforma uma função Python numa ferramenta
que o LLM pode usar. Compare com o que você escreveu no doc 01 e 02:

**O que você escreveu (Anthropic API direto):**

```python
FERRAMENTAS = [{
    "name": "busca_web",
    "description": "Busca informações atuais na web...",
    "input_schema": {
        "type": "object",
        "properties": {"query": {"type": "string"}},
        "required": ["query"]
    }
}]
```

**O equivalente em LangChain:**

```python
from langchain_core.tools import tool

@tool
def busca_web(query: str) -> str:
    """Busca informações atuais na web sobre concorrentes e mercado.
    
    Use para: features de concorrentes, preços, dados de adoção.
    NÃO use para: informações já disponíveis no contexto.
    
    Args:
        query: Termos de busca específicos
    """
    # implementação real aqui
    return resultados_formatados

# LangChain extrai automaticamente:
# - nome: "busca_web"  
# - descrição: o docstring
# - schema: dos type hints
print(busca_web.name)         # "busca_web"
print(busca_web.description)  # o docstring completo
print(busca_web.args_schema)  # schema JSON derivado dos type hints
```

**O que mudou:** em vez de escrever o dicionário JSON manualmente, o decorator
lê o docstring e os type hints e monta o schema automaticamente.

O **princípio é idêntico** ao que você já aprendeu: a descrição é um prompt que o
LLM lê para decidir quando e como usar a ferramenta. Só a sintaxe é mais ergonômica.

---

## Memory — os três tipos

Memory no LangChain é qualquer coisa que persiste estado entre turnos.
Três tipos que você vai ver em todos os tutoriais:

### Tipo 1: ConversationBufferMemory

O histórico completo de mensagens — exatamente o `historico` que você escreveu
no Benchmark Agent.

```python
from langchain.memory import ConversationBufferMemory

memory = ConversationBufferMemory()
memory.save_context(
    {"input": "Pesquise AirHelp"},
    {"output": "AirHelp cobra 35% success fee..."}
)

# Na próxima chamada, o histórico completo é passado pro LLM
print(memory.load_memory_variables({}))
# {"history": "Human: Pesquise AirHelp\nAI: AirHelp cobra 35%..."}
```

**Problema**: cresce indefinidamente. Numa sessão longa de Benchmark Agent,
cada busca adiciona ao histórico — o contexto fica caro.

### Tipo 2: ConversationSummaryMemory

Em vez de guardar mensagens brutas, resume periodicamente:

```python
from langchain.memory import ConversationSummaryMemory

memory = ConversationSummaryMemory(llm=ChatAnthropic())

# Após 10 turnos, faz uma chamada LLM para resumir os anteriores
# "O agente pesquisou AirHelp (35% fee), Resolvvi (35% fee), LiberFly (express R$1000/48h)"
# Usa esse resumo como contexto, não os turnos originais
```

**Tradeoff**: menor custo, mas você perde detalhes granulares que o LLM pode precisar.

### Tipo 3: VectorStoreMemory

Recupera memórias relevantes por similaridade semântica, não por ordem cronológica:

```python
from langchain.memory import VectorStoreRetrieverMemory
from langchain_community.vectorstores import FAISS

# Guarda memórias como embeddings
# Quando você precisa de contexto, recupera as mais semanticamente similares
# "Qual foi o resultado de AirHelp?" → recupera AQUELE trecho, não o histórico inteiro
```

**Uso típico**: base de conhecimento grande onde você não quer passar tudo pro LLM,
só o que é relevante para a pergunta atual.

---

### Como o FounderLens usa cada tipo

| Tipo | Equivalente no FounderLens | Onde fica |
|---|---|---|
| Buffer | `historico` dentro de uma execução de agente | RAM (descartado ao terminar) |
| Summary | Resumo de fase para handoff | `project_state.json` → campo `discover_summary` |
| VectorStore | Base de prompts do Deliver para busca semântica | Supabase pgvector (v2) |

O `project_state.json` que você viu no doc 02 é o análogo de "long-term memory" —
não é nenhum dos tipos LangChain formalmente, mas cumpre o mesmo papel:
persistir contexto relevante além de uma única execução.

---

## Callbacks — o hook de observabilidade

Este é o conceito novo que os docs anteriores ainda não tinham.

Você respondeu no checklist do doc 02: *"registro interno de erros e falhas para
melhoria contínua do produto"*. Callbacks são exatamente o mecanismo para isso.

**O que é:** um objeto que o LangChain chama automaticamente em cada evento do loop.

```python
from langchain.callbacks.base import BaseCallbackHandler

class FounderLensObserver(BaseCallbackHandler):
    
    def on_llm_start(self, serialized, prompts, **kwargs):
        """Disparado quando o LLM começa a processar."""
        print(f"[{timestamp()}] LLM chamado. Tokens estimados: {len(prompts[0])//4}")
    
    def on_tool_start(self, serialized, input_str, **kwargs):
        """Disparado quando uma ferramenta é chamada."""
        tool_name = serialized.get("name")
        print(f"[{timestamp()}] Ferramenta: {tool_name} | Input: {input_str[:100]}")
    
    def on_tool_end(self, output, **kwargs):
        """Disparado quando uma ferramenta retorna."""
        print(f"[{timestamp()}] Resultado: {output[:200]}")
    
    def on_chain_end(self, outputs, **kwargs):
        """Disparado quando a chain termina."""
        print(f"[{timestamp()}] Chain concluída: {outputs}")
    
    def on_agent_finish(self, finish, **kwargs):
        """Disparado quando o agente retorna a resposta final."""
        iteracoes = finish.return_values.get("iterations", "?")
        print(f"[{timestamp()}] Agente terminou em {iteracoes} iterações")

# Usando o callback
agent = AgentExecutor(agent=agente, tools=ferramentas, callbacks=[FounderLensObserver()])
agent.invoke({"input": contexto_founder})
```

**O que você captura com isso:**

```
[10:23:01] LLM chamado. Tokens estimados: 312
[10:23:02] Ferramenta: busca_web | Input: apps direitos passageiros cancelamento voo Brasil
[10:23:04] Resultado: AirHelp, Resolvvi, LiberFly aparecem nos resultados...
[10:23:04] LLM chamado. Tokens estimados: 1847
[10:23:06] Ferramenta: busca_web | Input: AirHelp pricing model features 2026
[10:23:08] Resultado: 35% success fee, app de tracking, 12 anos de mercado...
[10:23:09] LLM chamado. Tokens estimados: 3201
[10:23:15] Agente terminou em 5 iterações
```

**Por que isso importa pro FounderLens:**

1. **Custo por sessão**: você vê exatamente quantos tokens cada agente consome.
   Se o Benchmark Agent está usando 12k tokens por sessão, você sabe onde otimizar.

2. **Debugging de produto**: quando um founder diz "o benchmark estava errado",
   você tem o log completo de o que foi buscado, o que foi encontrado, o que foi
   concluído. Sem callbacks, você não sabe o que aconteceu.

3. **Melhoria contínua**: após 100 sessões, você consegue ver padrões — quais
   queries de busca produzem resultados ruins, quais founders chegam no limite
   de iterações sem conclusão, etc.

O loop explícito que você escreveu no doc 02 pode implementar exatamente esse padrão
sem LangChain — basta chamar sua função de log nos mesmos pontos:

```python
# Equivalente sem LangChain — o que você já escreveu, com logging
for i in range(MAX_ITERACOES):
    
    log_evento("llm_start", {"iteracao": i, "tokens_historico": contar_tokens(historico)})
    
    response = client.messages.create(...)
    
    if response.stop_reason == "tool_use":
        for block in response.content:
            if block.type == "tool_use":
                log_evento("tool_start", {"ferramenta": block.name, "input": block.input})
                resultado = await executar_ferramenta(block.name, block.input)
                log_evento("tool_end", {"resultado_preview": resultado[:200]})
    
    if response.stop_reason == "end_turn":
        log_evento("agent_finish", {"iteracoes": i+1, "sucesso": True})
        return ...
```

O LangChain formaliza esse padrão. Você o implementa da mesma forma, só escrevendo
os hooks manualmente nos pontos certos.

---

## Agent — o loop encapsulado

Em LangChain, `AgentExecutor` é o loop ReAct que você escreveu no doc 02 — encapsulado
num objeto:

```python
from langchain.agents import AgentExecutor, create_tool_calling_agent
from langchain_core.prompts import ChatPromptTemplate

# Define o prompt do agente
prompt = ChatPromptTemplate.from_messages([
    ("system", SYSTEM_PROMPT),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}")  # onde o LangChain coloca o histórico de tool calls
])

# Cria o agente
agent = create_tool_calling_agent(llm, tools, prompt)

# Encapsula no executor (= o loop)
executor = AgentExecutor(
    agent=agent,
    tools=tools,
    max_iterations=8,               # seu MAX_ITERACOES
    handle_parsing_errors=True,     # não quebra em JSON inválido
    callbacks=[FounderLensObserver()],
    verbose=True                    # imprime o trace no console
)

# Executa
resultado = executor.invoke({"input": contexto_founder})
```

**O que `AgentExecutor` faz internamente:**

```
enquanto não terminou e iterações < max_iterations:
    1. chama o LLM com o histórico
    2. se tool_use: executa, adiciona ao scratchpad, continua
    3. se end_turn: retorna
    4. dispara callbacks em cada passo
```

É exatamente o loop que você escreveu. A diferença: você não precisa escrever o
`while` manualmente — mas você também tem menos visibilidade do que acontece
dentro, a não ser que configure callbacks explicitamente.

**Parallelo com Apple Foundation Models:**
`AgentExecutor` do LangChain ≈ `LanguageModelSession` da Apple.
Ambos abstraem o loop. Ambos reduzem código. Ambos reduzem visibilidade.
Por isso, para o FounderLens, o loop explícito (que você escreveu no doc 02)
continua sendo a abordagem preferida mesmo se você usar LangChain server-side.

---

## Padrão a extrair pro FounderLens

### 1. Callbacks = observabilidade estruturada

Não é opcional. É o que te permite melhorar o produto depois do lançamento.
Implemente desde o início — mesmo que simples (um `print` estruturado com timestamp,
agente, iteração, ferramenta, tokens). Você vai querer esses logs.

### 2. Memory Summary = handoff entre fases

O `ConversationSummaryMemory` formaliza o padrão de "resumir para passar adiante".
No FounderLens, o handoff entre Discover e Define já é isso: um resumo das respostas
do founder, salvo no `project_state.json`, lido pelo agente da fase seguinte.

### 3. Chain composition = pipeline de fases

A sintaxe `|` não importa. O padrão importa: cada fase produz um output estruturado
que é o input da próxima. Se um estágio falha (Benchmark incompleto), o pipeline
para antes de passar pro próximo — não avança com dados ruins.

### 4. Tool docstring como prompt

O `@tool` decorator formaliza o que você já aprendeu: o docstring é o prompt que
o LLM lê para decidir quando usar a ferramenta. Em Swift (Anthropic API), você
escreve o campo `description` no dicionário. Mesmo princípio — só sintaxe diferente.

---

## Mapeamento ao FounderLens

| Conceito LangChain | No FounderLens | Implementação |
|---|---|---|
| `Chain` | Cada fase do Double Diamond | Sequência de agentes que passam `project_state.json` |
| `@tool` | `busca_web`, `ler_contexto`, `avaliar_rubrica` | Dicionário JSON (Anthropic API) ou `Tool` (Apple FM) |
| `BufferMemory` | `historico` dentro de um agente | Array de mensagens na execução |
| `SummaryMemory` | Handoff entre fases | Campo `summary` no `project_state.json` |
| `VectorStoreMemory` | Biblioteca de prompts do Deliver | Supabase pgvector (v2 — não precisa v1) |
| `Callbacks` | Log de observabilidade | Função `log_evento()` chamada nos pontos certos do loop |
| `AgentExecutor` | O loop ReAct | Escrito explicitamente (loop `for i in range(MAX)`) |

---

## Ponto de checagem

1. Qual dos tipos de Memory do LangChain corresponde ao `historico` que você escreveu
   no Benchmark Agent? E qual corresponde ao `project_state.json`?

2. O que é um Callback e por que ele importa para o FounderLens especificamente?
   Que tipo de problema ele resolve que você não consegue resolver sem ele?

3. Por que `AgentExecutor` do LangChain e `LanguageModelSession` da Apple são
   análogos? Qual a limitação que ambos compartilham?

4. Você pode escrever o equivalente de um Callback sem LangChain no seu loop
   explícito do doc 02? Como?

---

## Próximo

→ **[04 — CrewAI patterns](./04-crewai-patterns.md)**
O modelo de squads: Agent com role + backstory + tools, Task com output esperado,
Crew que orquestra. Mapeia diretamente para os auditores do FounderLens e o squad
do Deliver. Aqui os conceitos ficam muito concretos para o produto.

## Relacionados

- [[callbacks]] — o conceito novo deste doc, mapeado ao loop explícito
- [[state-memoria]] — BufferMemory = historico, SummaryMemory = handoff entre fases
- [[loop-explicito-vs-abstraido]] — AgentExecutor vs loop escrito à mão
- [[tool-description-as-prompt]] — @tool decorator formaliza o que você já sabia
- [[react-pattern]] — o que Chain e AgentExecutor abstraem
- [[MOC-agentes]] — mapa completo do grafo


---

*Última atualização: junho 2026*
