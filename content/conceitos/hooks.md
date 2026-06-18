---
tags: [conceito, claude-code, automação, extensibilidade]
aliases: [hooks, event hooks, SessionStart, PreToolUse, PostToolUse]
---

# Hooks — automação em pontos de evento do harness

> Hooks são scripts shell que o [[harness]] executa automaticamente em eventos específicos da sessão.
> São o mecanismo de extensão do Claude Code — sem modificar o LLM, você muda o comportamento.

## Os 4 hooks principais

```
SessionStart    → quando uma sessão começa
PreToolUse      → antes de qualquer chamada de ferramenta
PostToolUse     → depois que uma ferramenta retorna
Stop            → quando o agente termina uma resposta
```

## Configuração (settings.json)

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "cat /Users/eu/.claude/skills/context-injector.sh"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo '[AUDIT] Bash chamado'"
          }
        ]
      }
    ]
  }
}
```

## Casos de uso reais

| Hook | Uso típico |
|---|---|
| `SessionStart` | Injetar contexto (AGENTS.md, memória do projeto, data atual) |
| `PreToolUse` | Validar antes de executar (bloquear rm -rf, logar chamadas) |
| `PostToolUse` | Disparar CI, notificar Slack, salvar logs |
| `Stop` | Resumir sessão, atualizar memória persistente, enviar métricas |

## Hooks vs Callbacks

Hooks (Claude Code) e [[callbacks]] (LangChain) resolvem o mesmo problema:
**observabilidade e extensibilidade do loop do agente**.

| | Hooks (Claude Code) | Callbacks (LangChain) |
|---|---|---|
| Linguagem | Shell / qualquer executável | Python |
| Onde configurar | `settings.json` | `BaseCallbackHandler` |
| Escopo | Eventos da sessão inteira | Eventos de cada chain/tool |
| Portabilidade | Só Claude Code | Qualquer harness LangChain |

## Por que são poderosos

Hooks rodam **fora do LLM**. O modelo não sabe que existem. Isso significa:
- Zero tokens gastos com o comportamento do hook
- Executam mesmo que o LLM falhe
- Podem bloquear ações perigosas antes do LLM agir

## Relacionado

- [[harness]] — quem dispara os hooks
- [[callbacks]] — o equivalente em Python/LangChain
- [[skills]] — outra forma de extensão, via contexto no prompt
- [[progressive-disclosure]] — hooks que carregam contexto sob demanda
