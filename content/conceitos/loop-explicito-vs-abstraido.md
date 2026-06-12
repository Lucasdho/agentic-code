---
tags: [conceito, arquitetura, decisão]
aliases: [loop explícito, loop abstraído, visibilidade do loop, AgentExecutor, LanguageModelSession]
---

# Loop explícito vs abstraído

> Toda abstração de agente (LangChain `AgentExecutor`, Apple `LanguageModelSession`)
> faz a mesma coisa: esconde o loop `while` dentro de um método.
> A questão é: você quer ver o que acontece dentro?

## Loop abstraído

```python
# LangChain
executor = AgentExecutor(agent=agente, tools=ferramentas, max_iterations=8)
resultado = executor.invoke({"input": objetivo})
# O loop acontece dentro — você não vê
```

```swift
// Apple Foundation Models
let resposta = try await session.respond(to: objetivo)
// O loop acontece dentro — você não vê
```

**Vantagens:**
- Menos código para escrever
- Menos chance de bugs no loop em si
- Mais fácil de começar

**Desvantagens:**
- Você não vê as iterações individuais (a não ser que configure callbacks)
- Mais difícil de debugar quando algo dá errado
- Menos controle sobre tratamento de erros específicos

## Loop explícito

```python
# O que você escreveu no doc 02
for i in range(MAX_ITERACOES):
    response = client.messages.create(...)
    
    if response.stop_reason == "end_turn":
        return resultado
    
    if response.stop_reason == "tool_use":
        # você controla cada detalhe aqui
        for block in response.content:
            if block.type == "tool_use":
                resultado = await executar(block.name, block.input)
                # log, métricas, tratamento de erro específico
```

**Vantagens:**
- Visibilidade total de cada iteração
- Você pode logar, medir, tratar erros específicos por ferramenta
- Você pode salvar estado intermediário (checkpoint) a cada N iterações
- Mais fácil de adicionar [[callbacks]] sem depender do framework

**Desvantagens:**
- Mais código para escrever
- Você precisa tratar casos que o framework trataria de graça (parallel tool use, etc.)

## A decisão para o FounderLens

**Loop explícito** — por três razões:

1. **Observabilidade**: você precisa de logs granulares para melhorar o produto
2. **Custo**: você vai cobrar por sessão — precisa saber exatamente o que cada agente consome
3. **Erros específicos**: quando o Benchmark Agent falha, você quer saber *exatamente* o que aconteceu

Frameworks como LangChain e CrewAI podem ser usados como padrão (vocabulário),
mas o loop em produção deve ser explícito.

## Analogia

**Loop abstraído** = câmbio automático. Funciona. Você não vê o que acontece.
**Loop explícito** = câmbio manual. Mais trabalho. Controle total.

Para um app de consumo em produção que você vai cobrar e melhorar: câmbio manual.

## Relacionado

- [[react-pattern]] — o padrão que os dois implementam
- [[callbacks]] — como adicionar observabilidade ao loop explícito
- [[max-iterations]] — o parâmetro que ambos compartilham
- [[02-react-loop]] — o loop explícito na prática
- [[03-langchain-patterns]] — AgentExecutor documentado
- [[07-apple-foundation-models]] — LanguageModelSession da Apple
