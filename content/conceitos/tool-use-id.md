---
tags: [conceito, fundamentos, function-calling]
aliases: [tool_use_id, ID de ferramenta, link action-observation]
---

# tool_use_id — o ID crítico

> O `tool_use_id` é o campo que liga uma chamada de ferramenta ao seu resultado.
> Sem ele, o LLM não sabe qual resultado pertence a qual chamada.

## Por que existe

O LLM pode fazer **múltiplas chamadas de ferramenta em paralelo** numa mesma rodada.

```json
// LLM pede duas buscas ao mesmo tempo
[
  { "type": "tool_use", "id": "abc123", "name": "busca_web", "input": { "query": "AirHelp pricing" } },
  { "type": "tool_use", "id": "def456", "name": "busca_web", "input": { "query": "Resolvvi features" } }
]
```

Seu código precisa devolver os resultados com o ID correto:

```json
[
  { "type": "tool_result", "tool_use_id": "abc123", "content": "AirHelp cobra 35%..." },
  { "type": "tool_result", "tool_use_id": "def456", "content": "Resolvvi tem análise grátis..." }
]
```

Se os IDs não casarem → o LLM fica confuso, alucina, ou quebra o raciocínio.

## O erro clássico

Tratar só a primeira chamada de ferramenta quando o LLM fez várias.
Resultado: o LLM vê uma resposta sem context de qual chamada era.

## Regra prática

Sempre iterar sobre **todos** os blocos `tool_use` na resposta, não só o primeiro.

```python
for block in response.content:
    if block.type == "tool_use":
        resultado = executar(block.name, block.input)
        tool_results.append({
            "type": "tool_result",
            "tool_use_id": block.id,  # ← sempre usar block.id, não hardcodar
            "content": resultado
        })
```

## Relacionado

- [[react-pattern]] — o ciclo Action → Observation que o ID conecta
- [[01-function-calling]] — protocolo completo de function calling
- [[02-react-loop]] — uso real no Benchmark Agent
