---
tags: [agente, founderlens, develop, produto]
aliases: [Development Auditor, auditor de desenvolvimento, Dev Audit]
fase: Dev Audit (fim do Develop)
status: especificado (não implementado)
---

# Development Auditor

> **Onde:** Dev Audit — executado quando o founder completa o Develop
> **O que faz:** lê os artefatos de spec gerados → aplica 3 lentes → retorna diagnóstico + patches cirúrgicos

## Responsabilidade única

Revisar a qualidade e coerência dos artefatos de desenvolvimento (specs de dados, backend, frontend).
**Sem busca web.** **Sem scoring de mercado.** Só leitura e análise interna.

## Ferramentas

- `ler_artefato(nome: str)` — lê uma spec específica do projeto

**Nota:** este agente **não usa** `busca_web`. Toda informação está nos artefatos.
Isso o torna mais barato e mais rápido que os outros auditores.

## As 3 lentes

| Lente | Pergunta | O que detecta |
|---|---|---|
| **Coerência** | As specs se contradizem? | Campo em dados vs uso no backend, modelo de preço vs feature set |
| **Testabilidade** | Cada requisito tem critério de teste claro? | "Funcionar bem" sem definição de "bem" |
| **Completude** | Faltam casos de borda? | Happy path definido mas error states ausentes |

## Trace de execução típica

```
iter 1: ler_artefato("spec-dados")     → modelo de dados, entidades, relações
iter 2: ler_artefato("spec-backend")   → endpoints, lógica, regras de negócio
iter 3: ler_artefato("spec-frontend")  → telas, fluxos, estados de UI
iter 4: end_turn → diagnóstico das 3 lentes + lista de patches
```

Apenas 4 iterações. Barato. Rápido. Sem busca web.

## Output esperado

```json
{
  "coerencia": {
    "score": 82,
    "issues": [
      "spec-backend define campo 'valor_compensacao' como Float, mas spec-dados usa Decimal"
    ]
  },
  "testabilidade": {
    "score": 65,
    "issues": [
      "Critério de sucesso do claim não está definido (aprovado? pago? submetido?)"
    ]
  },
  "completude": {
    "score": 71,
    "issues": [
      "Fluxo de cancelamento de voo define happy path mas não define o que acontece se ANAC API cair"
    ]
  },
  "patches": [
    { "artefato": "spec-dados", "linha": 34, "sugestao": "Mudar Float para Decimal(10,2)" }
  ]
}
```

## Posição no pipeline

```
Develop phase → founder completa specs com agentes
→ [Development Auditor] ← lê todos os artefatos
→ patches cirúrgicos → founder revisa e aplica
→ Deliver phase ([[prompt-generator-squad]])
```

## Padrões que usa

- [[react-pattern]] — loop de leitura sequencial de artefatos
- [[state-memoria]] — lê artefatos do project_state
- [[callbacks]] — monitorar quais tipos de issue aparecem mais (para melhorar os prompts do Develop)

## Docs de referência

- [[00-o-que-e-um-agente]] — agente sem busca externa (só ferramentas de leitura)
- [[04-crewai-patterns]] — como orquestrar auditors em squad
