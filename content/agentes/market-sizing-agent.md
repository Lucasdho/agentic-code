---
tags: [agente, founderlens, midpoint-audit, produto]
aliases: [Market Sizing Agent, agente de tamanho de mercado, TAM SAM SOM]
fase: Midpoint Audit (paralelo ao Validation Auditor)
status: especificado (não implementado)
---

# Market Sizing Agent

> **Onde:** Midpoint Audit — roda em paralelo (ou em sequência) com o [[validation-auditor]]
> **O que faz:** busca dados reais de tamanho de mercado para fundamentar o score "Worth it" da rubrica R-W-W

## Por que existe separado do Validation Auditor

O Validation Auditor precisa de dados de market sizing para pontuar o "W" (Worth it).
Esses dados precisam ser **reais e fundamentados** — não inventados pelo LLM.

Separar em agente próprio permite:
1. Reutilizar o resultado em outros contextos (pitch deck, business plan)
2. Correr em paralelo com o Validation Auditor (se a arquitetura suportar)
3. Ter um sistema de logging separado para auditar a qualidade das fontes

## Ferramentas

- `busca_web(query)` — Brave Search para dados de mercado reais

## Queries típicas

```python
queries = [
    f"TAM mercado {setor} {pais} {ano}",
    f"CAGR crescimento {setor} previsão {ano+3}",
    f"número de usuários {produto_similar} {pais}",
    f"receita {concorrente_principal} {ano} relatório"
]
```

## Output esperado

```json
{
  "tam": {
    "valor": "R$ 180M",
    "fonte": "ANAC Relatório Anual 2025",
    "confiança": "alta"
  },
  "sam": {
    "valor": "R$ 45M",
    "justificativa": "20% do TAM — passageiros com voos domésticos frequentes + smartphone",
    "confiança": "estimada"
  },
  "som": {
    "valor": "R$ 9M",
    "justificativa": "20% do SAM nos primeiros 3 anos com distribuição App Store",
    "confiança": "estimada"
  },
  "cagr": "12% a.a.",
  "fontes": ["ANAC 2025", "IBGE pesquisa mobilidade", "Techcrunch Resolvvi funding"]
}
```

A chave aqui é `"confiança"`: o agente distingue dados reais encontrados (`"alta"`)
de estimativas calculadas (`"estimada"`). Nunca apresenta estimativa como fato.

## Padrões que usa

- [[react-pattern]] — loop de pesquisa
- [[tool-description-as-prompt]] — crucial: a descrição deve deixar claro que é para dados de mercado reais
- [[callbacks]] — monitorar qualidade das fontes ao longo do tempo

## Relacionado

- [[validation-auditor]] — usa o output deste agente para pontuar "Worth it"
- [[benchmark-agent]] — agente irmão (pesquisa concorrentes, não tamanho)
