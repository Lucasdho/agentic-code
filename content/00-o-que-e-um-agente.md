---
tags: [fundamentos, agente, react, módulo]
aliases: [O que é um agente, agente vs LLM, loop de decisão]
módulo: "00"
próximo: "[[01-function-calling]]"
---

# 00 — O que é um agente
**Fundamentos · v1.0 · junho 2026**

---

## Conceito central

Uma chamada LLM normal é **linear e stateless**:

```
você → [prompt] → LLM → [resposta] → fim
```

Um agente é um **loop com decisão**:

```
objetivo
  → LLM raciocina: "o que preciso fazer agora?"
  → decide usar uma ferramenta
  → seu código executa a ferramenta
  → resultado volta pro LLM
  → LLM raciocina: "já tenho o suficiente?"
    → se não: próximo passo
    → se sim: resposta final
```

O LLM não executa nada. Ele **decide o que pedir**. Seu código executa.
Você é o executor. O LLM é o raciocínio.

---

## Por que "agência"

O nome não é por acaso. Agência = capacidade de agir no mundo com base em objetivos,
não apenas responder a um estímulo.

A diferença prática:

| Chamada normal | Agente |
|---|---|
| Você formula a pergunta exata | Você define o objetivo |
| Uma resposta, fim | N rodadas até concluir |
| LLM não sabe o resultado de ações | LLM observa e adapta |
| Stateless | Estado persiste entre rodadas |
| Determinístico (dado o prompt) | Emergente (depende do que encontra) |

---

## O padrão ReAct

ReAct = **Re**ason + **Act**. Publicado pelo Google em 2022, é o padrão por baixo de
quase todo agente que você vai encontrar nos frameworks.

O ciclo:

```
Thought: preciso saber os preços da AirHelp para comparar com o modelo do founder
Action: busca_web("AirHelp pricing model 2026")
Observation: "AirHelp cobra 35% de success fee, sem pagamento antecipado..."
Thought: tenho o preço. Agora preciso da Resolvvi
Action: busca_web("Resolvvi pricing Brazil 2026")
Observation: "Resolvvi cobra 35% success fee, análise gratuita..."
Thought: tenho os dois principais. Há gap no in-the-moment que nenhum cobre.
Action: [nenhuma — tenho o suficiente]
Final Answer: síntese do benchmark com o gap identificado
```

O LLM produz `Thought` e `Action` como texto estruturado.
Seu código lê a `Action`, executa, devolve a `Observation`.
Repete até o LLM decidir que tem o suficiente para uma `Final Answer`.

---

## Como funciona na prática (estrutura de código)

```python
# Pseudocódigo — agnóstico de framework

def run_agent(objetivo, ferramentas):
    historico = [{"role": "user", "content": objetivo}]
    
    while True:
        resposta = llm.call(historico, ferramentas_disponíveis=ferramentas)
        
        if resposta.tipo == "resposta_final":
            return resposta.conteudo          # loop termina
        
        if resposta.tipo == "uso_de_ferramenta":
            resultado = executar_ferramenta(
                nome=resposta.ferramenta,
                args=resposta.argumentos
            )
            historico.append(resposta)         # o que o LLM "pediu"
            historico.append(resultado)        # o que aconteceu
            # continua o loop
```

Três elementos essenciais:
1. **Histórico** — o LLM precisa ver o que já aconteceu para raciocinar o próximo passo
2. **Ferramentas disponíveis** — o que o LLM pode pedir para ser executado
3. **Condição de parada** — quando o LLM decide que terminou (ou você força um limite)

---

## Onde o FounderLens já tem esse padrão

### Benchmark Agent (Discover, ingrediente 10)

A spec diz: *"Grounded, never invented. Competitor features and prices come from
web-search via the user's key."*

O que isso significa em código de agente:

```
Objetivo: "Identifique os top 3 concorrentes de [ideia do founder],
           extraia features, preço e por que ganharam, sintetize o gap"

Ferramenta disponível: busca_web(query)

Rodada 1:
  Thought: vou buscar o mercado geral primeiro
  Action: busca_web("flight cancellation passenger rights app Brazil 2026")
  Observation: [resultados de busca]

Rodada 2:
  Thought: AirHelp apareceu. Vou detalhar
  Action: busca_web("AirHelp pricing model features review")
  Observation: [35% success fee, 3M+ passengers, 12 anos...]

Rodada 3:
  Thought: preciso do segundo concorrente
  Action: busca_web("Resolvvi app passageiros direitos Brasil")
  Observation: [35% success fee, foco em Brasil, R$130M recuperados...]

Rodada 4:
  Thought: tenho os 3 principais. Vou sintetizar o gap
  Final Answer: [benchmark completo com gap identificado]
```

Isso não é uma pergunta para o LLM. É um loop de agente. 3-4 iterações,
resultados reais da web, síntese fundamentada.

### Validation Auditor (Midpoint Audit)

```
Objetivo: "Avalie este projeto com a rubrica R-W-W. Score 0-100."

Ferramentas: busca_web (para market sizing), ler_contexto_do_projeto

Rodada 1: lê o contexto completo do projeto
Rodada 2: busca dados de mercado reais para fundamentar o score
Rodada 3: aplica rubrica, retorna score + veredito
```

### Development Auditor

Agente especializado — **sem ferramentas de busca**. Só leitura:

```
Objetivo: "Revise os artefatos do Develop. Aplique as 3 lentes."

Ferramenta: ler_artefato(nome)

Rodada 1: lê spec de dados
Rodada 2: lê spec de backend
Rodada 3: lê spec de frontend
Rodada 4: retorna diagnóstico com patches cirúrgicos
```

---

## O que NÃO é um agente no FounderLens

Para clareza: nem tudo no FounderLens é um agente.

- **O loop de perguntas** (Discover/Define): uma chamada LLM por resposta do usuário.
  O app envia as respostas acumuladas + próxima pergunta. Simples, stateless, linear.
  Não precisa ser agente.

- **A geração de chips de resposta**: uma chamada LLM. Rápida, determinística.

A regra prática: **se a tarefa requer buscar informação externa, ler múltiplos
artefatos, ou tomar decisões adaptativas com base em resultados — é agente.**
Se é "dado este contexto, gere este output" — é uma chamada simples.

---

## Estado e memória

Um agente precisa de estado entre rodadas. Como isso é implementado:

**Memória de curto prazo** — o `historico` de mensagens da sessão atual.
O LLM vê tudo que aconteceu naquela execução. Mais contexto = mais tokens = mais custo.

**Memória de longo prazo** — o que persiste além de uma execução.
No FounderLens: o `project_state.json` (GSD Core pattern) ou SwiftData/Supabase.
Quando o usuário retoma uma sessão, o agente lê o estado salvo.

**Memória de trabalho** — arquivos intermediários durante execução.
No padrão GSD Core: `CONTEXT.md`, `STATE.md`. No FounderLens: o handoff entre fases.

---

## Ponto de checagem

Antes de seguir para o doc 01, você deve conseguir responder:

1. Qual a diferença entre uma chamada LLM e um agente em termos de estrutura de código?
2. O que é o loop ReAct e quais são suas três partes?
3. Por que o Benchmark Agent do FounderLens precisa ser um agente e não uma chamada simples?
4. O que é "estado" num agente e como o FounderLens já resolve isso na spec?

---

## Próximo

→ **[01 — Function Calling](./01-function-calling.md)**
Como o LLM "usa" uma ferramenta na prática. O mecanismo JSON. Exemplos em Anthropic API
e Apple Foundation Models.

## Relacionados

- [[react-pattern]] — o padrão que define o loop
- [[state-memoria]] — memória de curto e longo prazo
- [[loop-explicito-vs-abstraido]] — por que escrever o loop
- [[benchmark-agent]] — o primeiro agente concreto do FounderLens
- [[validation-auditor]] — exemplo de agente sem busca web
- [[development-auditor]] — agente com só ferramentas de leitura
- [[MOC-agentes]] — mapa completo do grafo


---

*Última atualização: junho 2026*
