---
tags: [conceito, fundamentos, estado, memória]
aliases: [state, memória, short-term memory, long-term memory, project_state]
---

# Estado e Memória — dois níveis

> Um agente sem estado é cego. Cada iteração precisaria começar do zero.
> Estado é o que dá continuidade ao raciocínio.

## Os dois níveis

### Memória de curto prazo (dentro de uma execução)

O `historico` — array de mensagens que o LLM vê em cada iteração.

```python
historico = [
    {"role": "user", "content": objetivo_inicial},
    {"role": "assistant", "content": [tool_use_block]},
    {"role": "user", "content": [tool_result_block]},
    # ... cresce a cada rodada
]
```

**Características:**
- Existe só durante a execução do agente
- O LLM vê tudo a cada chamada (= mais tokens = mais custo)
- Descartado quando o agente termina

**Analogia LangChain:** `ConversationBufferMemory`

### Memória de longo prazo (persiste entre sessões)

O `project_state.json` (ou SwiftData / Supabase) — o que sobrevive quando o app fecha.

```json
{
  "founder_id": "uuid",
  "ideia": "app de cancelamento de voo",
  "discover": {
    "status": "concluido",
    "benchmark": { "concorrentes": [...], "gap": "..." },
    "ingredientes": { "dor": "...", "quem_tem": "..." }
  },
  "define": {
    "status": "em_progresso"
  }
}
```

**Características:**
- Persiste entre sessões
- O agente lê no início de cada execução
- Atualiza ao terminar com sucesso

**Analogia LangChain:** `ConversationSummaryMemory` (o que foi destilado) + banco de dados

### Memória de trabalho (arquivos intermediários)

No padrão GSD Core: `CONTEXT.md`, `STATE.md`.
No FounderLens: o handoff entre fases (o que uma fase passa para a próxima).

## O custo do histórico longo

Cada mensagem no `historico` = tokens na chamada.

```
Iteração 1: 500 tokens
Iteração 2: 500 + 800 = 1300 tokens
Iteração 3: 1300 + 600 = 1900 tokens
...
Iteração 8: ~8000 tokens só de contexto
```

Para agentes longos: considere `ConversationSummaryMemory` (resume iterações anteriores)
em vez de guardar tudo bruto.

## Como o FounderLens resolve

| Nível | Implementação | Onde |
|-------|--------------|------|
| Curto prazo | `historico` dentro do agente | RAM (descartado) |
| Longo prazo | `project_state.json` | SwiftData (v1) / Supabase (v2) |
| Trabalho | Handoff JSON entre fases | `project_state.json` → campos por fase |

## Relacionado

- [[react-pattern]] — o histórico é o que dá contexto ao ReAct
- [[benchmark-agent]] — usa os dois níveis
- [[00-o-que-e-um-agente]] — introdução ao estado
- [[03-langchain-patterns]] — Memory Types formais do LangChain
