---
tags: [MOC, índice, hub]
aliases: [Mapa de Agentes, índice de conhecimento, hub]
---

# MOC — Código Agentico & FounderLens

> **Map of Content** — o nó central do grafo de conhecimento.
> Tudo conecta aqui. Comece aqui se estiver desorientado.

---

## Plano de estudos

→ [[PLANO-DE-ESTUDOS]] — sequência completa, decisão arquitetural, repos de referência

---

## Fundamentos (estudar em ordem)

```
[[00-o-que-e-um-agente]]
    ↓
[[01-function-calling]]
    ↓
[[02-react-loop]]
    ↓
[[03-langchain-patterns]]
    ↓
[[04-crewai-patterns]]
    ↓
05-google-adk-patterns (pendente)
```

---

## Conceitos atômicos

Cada ideia fundamental tem sua própria nota. Estas são as mais importantes:

| Conceito | O que é |
|---|---|
| [[react-pattern]] | Reason+Act — o ciclo de todo agente |
| [[tool-use-id]] | O ID que liga Action a Observation |
| [[max-iterations]] | Orçamento de segurança do loop |
| [[callbacks]] | Observabilidade — ver o que acontece dentro |
| [[state-memoria]] | Curto prazo (historico) vs longo prazo (project_state) |
| [[tool-description-as-prompt]] | A descrição é um prompt para o LLM |
| [[loop-explicito-vs-abstraido]] | Por que escrever o loop, não abstrair |

---

## Agentes do FounderLens

Os agentes que vão existir no produto, com suas specs:

```
Double Diamond
│
├── Discover
│   └── [[benchmark-agent]]          ← pesquisa concorrentes
│
├── Midpoint Audit
│   ├── [[market-sizing-agent]]      ← TAM/SAM/SOM reais
│   └── [[validation-auditor]]       ← score R-W-W
│
├── Develop
│   └── (agentes de spec — a definir)
│
├── Dev Audit
│   └── [[development-auditor]]      ← 3 lentes: coerência/testabilidade/completude
│
└── Deliver
    └── [[prompt-generator-squad]]   ← wave execution, cards por história
```

---

## Decisão arquitetural em aberto

Três opções (não decidir antes de estudar):

| Opção | Resumo | Status |
|---|---|---|
| A — Pure BYOK | Usuário traz chave Anthropic | Mata conversão de não-técnicos |
| B — Server-side | FounderLens tem servidor | Preferida, habilita frameworks Python |
| C — Apple FM híbrido | On-device + Claude no servidor | Depende de evolução da Apple |

→ Decisão final: [[09-founderlens-agent-map]] (após estudar todos os módulos)

---

## Referências de produto e plataforma (pendentes)

- 06-multica-openspec-map (pendente)
- [[07-apple-foundation-models]] (pendente)
- 08-arquitetura-server-side (pendente)

---

## Grafo de dependências entre conceitos

```
[[react-pattern]]
    ├── usa → [[tool-use-id]]
    ├── controlado por → [[max-iterations]]
    ├── visível via → [[callbacks]]
    └── implementado como → [[loop-explicito-vs-abstraido]]

[[loop-explicito-vs-abstraido]]
    └── facilita → [[callbacks]]

[[state-memoria]]
    ├── curto prazo = historico (no loop)
    └── longo prazo = project_state.json (no produto)

[[tool-description-as-prompt]]
    └── afeta cada → ferramenta de cada agente
```

---

*Última atualização: junho 2026*
