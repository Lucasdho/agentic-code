---
tags: [agente, founderlens, discover, produto]
aliases: [Benchmark Agent, agente de benchmark, concorrentes]
fase: Discover — ingrediente 10
status: especificado (não implementado)
---

# Benchmark Agent

> **Onde:** Discover, ingrediente 10
> **O que faz:** pesquisa os top 3 concorrentes do founder, extrai features/preço/gap, sintetiza o que nenhum cobre

## Responsabilidade única

Recebe o contexto do founder → pesquisa → retorna JSON estruturado com benchmark.
**Não faz scoring. Não faz auditoria.** Só o benchmark.

## Ferramentas

- `busca_web(query: str)` — Brave Search API (ou Serper)

## Parâmetros de execução

```python
modelo = "claude-opus-4-5"  # raciocínio pesado
max_iterations = 8           # 3 concorrentes × ~2 buscas + margem
max_tokens = 4096
```

## Output esperado (JSON)

```json
{
  "concorrentes": [
    {
      "nome": "AirHelp",
      "features_principais": ["tracking de voo", "submissão automática de claim"],
      "modelo_preco": "35% success fee",
      "por_que_ganharam": "12 anos de mercado, 3M+ passageiros, marca global"
    }
  ],
  "gap_identificado": "Todos atuam APÓS o fato — nenhum ajuda no momento do cancelamento",
  "sinal_regulatorio": "OAB notificou Resolvvi e LiberFly em 2020 por publicidade irregular"
}
```

## Trace de execução típica (app de voo)

```
iter 1: busca_web("apps direitos passageiros cancelamento voo Brasil 2026")
iter 2: busca_web("AirHelp pricing model features 2026")
iter 3: busca_web("Resolvvi app passageiros Brasil")
iter 4: busca_web("LiberFly express modelo preço")
iter 5: end_turn → retorna benchmark completo
```

5 iterações. Zero invenção. Resultado fundamentado.

## Passagem de contexto

O output é salvo no `project_state.json`:
```json
"discover": {
  "benchmark": { ... },
  "benchmark_status": "concluido",
  "benchmark_iteracoes": 5
}
```

O [[validation-auditor]] lê o benchmark daqui para fazer o scoring.

## Padrões que usa

- [[react-pattern]] — loop de pesquisa iterativo
- [[tool-use-id]] — para o parallel tool use (se buscar 2 concorrentes ao mesmo tempo)
- [[max-iterations]] — limite de 8
- [[state-memoria]] — lê contexto do founder, salva resultado
- [[callbacks]] — monitorar tokens por sessão

## Docs de referência

- [[02-react-loop]] — implementação Python completa
- [[01-function-calling]] — protocolo de ferramenta
