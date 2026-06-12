---
tags: [conceito, fundamentos, segurança]
aliases: [max_iterations, limite de iterações, safety mechanism]
---

# max_iterations — mecanismo de segurança

> `max_iterations` não é um limite de qualidade. É um **orçamento de segurança**.
> Define o custo máximo e o tempo máximo que um agente pode consumir.

## Por que é necessário

Sem limite, um agente pode:
- Ficar em loop infinito (LLM chama ferramenta → resultado ruim → chama de novo → ...)
- Consumir centenas de dólares de API
- Bloquear o usuário esperando

## Como calcular o limite certo

O limite deve ser **generoso o suficiente para o caso de uso** — não mínimo, não infinito.

**Fórmula:**
```
max_iterations = (chamadas_esperadas × margem_de_erro) + buffer
```

**Exemplo — Benchmark Agent:**
- 3 concorrentes × ~2 buscas cada = 6 chamadas esperadas
- Margem para queries ruins, retries = +2
- **max_iterations = 8**

## O que acontece quando o limite é atingido

Depende de como você implementa:

```python
# Opção A: falha silenciosa (ruim)
return None

# Opção B: falha informativa (bom)
return {
    "sucesso": False,
    "erro": "Limite de iterações atingido",
    "progresso": historico,   # o que foi coletado até agora
    "iteracoes": MAX_ITERACOES
}

# Opção C: resultado parcial (às vezes melhor)
# Salva o que tem e avisa o usuário que o benchmark pode estar incompleto
```

## Relação com custo

Cada iteração = uma chamada LLM + possível chamada de ferramenta.
Custo por iteração varia por modelo:

| Modelo | Custo médio/iter | Limite razoável |
|--------|-----------------|----------------|
| claude-haiku | ~$0.001 | 20-30 |
| claude-sonnet | ~$0.01 | 10-15 |
| claude-opus | ~$0.05 | 6-10 |

## Relacionado

- [[react-pattern]] — o loop que o limite controla
- [[callbacks]] — como monitorar iterações em tempo real
- [[benchmark-agent]] — limite de 8 para o caso concreto
- [[02-react-loop]] — implementação
