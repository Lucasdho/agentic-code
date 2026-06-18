---
tags: [conceito, claude-code, infraestrutura, ambiente]
aliases: [harness, execution harness, ambiente de execução]
---

# Harness — o ambiente de execução do agente

> O **harness** é o container que envolve o LLM e transforma um modelo de linguagem em um agente funcional.
> Ele orquestra ferramentas, gerencia sessões, dispara [[hooks]] e expõe [[skills]].

## O que é

Em Claude Code (e em qualquer agente de produção), o harness é a camada entre o LLM e o sistema operacional. O LLM pede coisas. O harness executa.

```
Usuário
  ↓
Harness ── sessão, contexto, histórico
  ↓
LLM (Claude) ── raciocina, pede ações
  ↓
Harness ── valida, executa, devolve
  ↓
Ferramentas (Bash, Read, Edit, Write, Agent...)
```

## Responsabilidades do harness

| Função | O que faz |
|---|---|
| Context management | Injeta CLAUDE.md, memory, sistema de memória |
| Tool dispatch | Recebe `tool_use` do LLM, executa, retorna `tool_result` |
| Hook execution | Dispara scripts no `settings.json` em cada evento |
| Permission gating | Aprova ou bloqueia chamadas de ferramenta |
| Session state | Mantém histórico e comprime contexto automaticamente |
| Subagent spawning | Cria agentes filhos com contexto herdado ou limpo |

## Por que isso importa para engenharia de agentes

Quando você escreve seu próprio agente, **você é o harness**. O loop ReAct que você codifica no doc [[02-react-loop]] é exatamente isso: receber a intenção do LLM, executar a ferramenta, devolver a observação.

```python
# Você, sendo o harness:
while True:
    response = llm.complete(historico)
    
    if response.stop_reason == "tool_use":
        resultado = executar_ferramenta(response.tool_name, response.tool_input)
        historico.append(tool_result(resultado))  # devolve ao LLM
    else:
        break  # Final Answer
```

## Harness como produto

Claude Code, LangChain, CrewAI, Google ADK — todos são harnesses com diferentes trade-offs:

| Harness | Ponto forte |
|---|---|
| Claude Code | Otimizado para dev, hooks extensíveis |
| LangChain | Ecossistema de integrações |
| CrewAI | Multi-agent com papéis estruturados |
| Google ADK | Integração com infra Google |

## Relacionado

- [[hooks]] — eventos que o harness expõe para extensão
- [[skills]] — instruções que o harness injeta no contexto
- [[react-pattern]] — o padrão que o harness implementa
- [[loop-explicito-vs-abstraido]] — ser o harness vs usar um pronto
- [[callbacks]] — o equivalente em harnesses Python (LangChain)
