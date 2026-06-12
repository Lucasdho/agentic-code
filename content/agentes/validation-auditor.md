---
tags: [agente, founderlens, midpoint-audit, produto]
aliases: [Validation Auditor, auditor de validação, R-W-W]
fase: Midpoint Audit (entre Define e Develop)
status: especificado (não implementado)
---

# Validation Auditor

> **Onde:** Midpoint Audit — executado quando o founder completa Discover + Define
> **O que faz:** lê o contexto completo do projeto → aplica a rubrica R-W-W → retorna score 0-100 + veredito

## Responsabilidade única

Avaliar se o projeto está pronto para entrar no Develop.
**Não pesquisa concorrentes** (isso é o [[benchmark-agent]]).
**Não revisa artefatos de desenvolvimento** (isso é o [[development-auditor]]).

## Ferramentas

- `ler_contexto_projeto()` — lê o `project_state.json` completo
- `busca_web(query)` — para fundamentar dados de market sizing com fontes reais

## A rubrica R-W-W

**Real — Worth it — Winnable**

| Dimensão | Pergunta central | Peso |
|---|---|---|
| **Real** | O problema existe de forma comprovável? | 33% |
| **Worth it** | O mercado é grande o suficiente para valer? | 33% |
| **Winnable** | O founder pode ganhar contra os concorrentes? | 34% |

Score abaixo de 60: recomenda revisitar Discover/Define antes de prosseguir.
Score acima de 80: proceed com confiança.

## Trace de execução típica

```
iter 1: ler_contexto_projeto() → lê todo o state
iter 2: busca_web("TAM mercado apps passageiros aereos Brasil") → fundamenta o W
iter 3: end_turn → retorna score + veredito + 3 riscos principais
```

## Output esperado

```json
{
  "score_total": 74,
  "dimensoes": {
    "real": { "score": 85, "justificativa": "Dor comprovada por volume de reclamações ANAC" },
    "worth_it": { "score": 70, "justificativa": "TAM estimado R$180M, crescimento 12% a.a." },
    "winnable": { "score": 68, "justificativa": "Gap claro, mas AirHelp pode replicar in-moment feature" }
  },
  "veredicto": "proceed_with_caution",
  "riscos": ["dependência regulatória OAB", "custo de aquisição no aeroporto", "...]
}
```

## Posição no pipeline

```
[[benchmark-agent]] → salva dados
Define phase → founder define posicionamento
→ [Validation Auditor] ← lê benchmark + todo o contexto
→ score + veredito → founder decide: proceed ou revisar
→ [[development-auditor]] (depois do Develop)
```

## Padrões que usa

- [[react-pattern]] — loop de leitura + busca + avaliação
- [[state-memoria]] — lê long-term memory (project_state.json)
- [[callbacks]] — monitorar qualidade do scoring ao longo do tempo

## Docs de referência

- [[00-o-que-e-um-agente]] — por que isso é um agente (múltiplas ferramentas, adaptativo)
- [[04-crewai-patterns]] — quando virar parte de um squad
