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
[[00-o-que-e-um-agente]]    ✅
    ↓
[[01-function-calling]]     ✅
    ↓
[[02-react-loop]]           ✅
    ↓
[[03-langchain-patterns]]   ✅
    ↓
[[04-crewai-patterns]]      ✅
    ↓
[[05-google-adk-patterns]]  ✅
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

## Decisão arquitetural

**Direção definida pelo founder:** Apple Foundation Models como primário, Claude como fallback.

| Opção | Resumo | Status |
|---|---|---|
| A — Pure BYOK | Usuário traz chave Anthropic | ❌ Mata conversão de não-técnicos |
| B — Server-side | FounderLens tem servidor, paga Claude | ❌ Custo desnecessário se PCC é free |
| C — Apple FM first | PCC (free <2M users) + Claude fallback via LanguageModel protocol | ✅ Direção escolhida |

**Questões abertas (resolver nos vídeos do WWDC):**
- Tool calling funciona no AFM 3 Cloud Pro (PCC)?
- Python SDK tem acesso a PCC ou só on-device?
- Context window do Cloud Pro é suficiente para o project_state completo?

**Status:** ✅ ESTUDOS COMPLETOS (lições 00-09). Próximo: assistir WWDC → confirmar questões abertas → código.
Stack: Apple FM (PCC) para agentes + Supabase (DB/auth/realtime) + Claude como fallback via LanguageModel protocol.
Começar por: Benchmark Agent — mais simples, valida infra, primeiro no pipeline.

---

## Referências de produto e plataforma

- [[06-multica-openspec-map]]   ✅ — competidores e padrões de produto
- [[08-arquitetura-server-side]] ✅ — decisão fechada: Opção B + Apple FM on-device para UI
- [[09-founderlens-agent-map]]  ✅ — mapa completo + ordem de implementação

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
