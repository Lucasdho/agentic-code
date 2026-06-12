---
tags: [conceito, fundamentos, padrão]
aliases: [ReAct, Reason+Act, Razão+Ação]
---

# ReAct Pattern

> **ReAct = Reason + Act**
> Publicado pelo Google em 2022. É o padrão por baixo de quase todo agente moderno.

## O ciclo

```
Thought  → o LLM raciocina internamente
Action   → o LLM pede para executar algo (uma ferramenta)
Observation → seu código executa e devolve o resultado
```

Repete até o LLM ter o suficiente para uma **Final Answer**.

## Por que importa

Sem ReAct, o LLM produz uma resposta baseada só no que já sabe.
Com ReAct, o LLM pode **buscar, observar, adaptar** — respondendo com dados reais.

## O que o LLM produz

O LLM nunca executa nada. Ele produz texto estruturado descrevendo:
- O que quer fazer (`tool_use`)
- Com quais argumentos

Seu código lê isso e executa.

## Relacionado

- [[loop-explicito-vs-abstraido]] — onde o loop ReAct fica no código
- [[tool-use-id]] — o mecanismo que liga Action a Observation
- [[max-iterations]] — quando o loop para
- [[00-o-que-e-um-agente]] — introdução ao padrão
- [[02-react-loop]] — implementação completa
