---
tags: [conceito, observabilidade, langchain, produção]
aliases: [Callbacks, hooks, observabilidade, logging estruturado]
---

# Callbacks — observabilidade do loop

> Callbacks são funções que disparam **automaticamente** em cada evento do loop do agente.
> São o mecanismo de observabilidade — você vê o que está acontecendo dentro do agente.

## Por que é crítico para o FounderLens

Sem callbacks:
- Um founder diz "o benchmark estava errado" → você não sabe o que aconteceu
- Um agente gasta 12k tokens por sessão → você não sabe onde está o desperdício
- Um agente falha silenciosamente → você não sabe em qual etapa

Com callbacks:
- Log completo de cada busca, cada resultado, cada decisão do LLM
- Tokens consumidos por iteração
- Tempo de cada chamada de ferramenta
- Taxa de sucesso/falha por ferramenta

## Eventos principais

```
on_llm_start      → LLM começa a processar
on_tool_start     → ferramenta é chamada
on_tool_end       → ferramenta retorna
on_llm_end        → LLM termina de responder
on_agent_finish   → agente dá a resposta final
on_chain_error    → erro em qualquer ponto
```

## Implementação sem LangChain (loop explícito)

No loop que você escreveu no doc 02, adicione chamadas de log nos mesmos pontos:

```python
def log_evento(tipo: str, dados: dict):
    entrada = {
        "timestamp": datetime.now().isoformat(),
        "tipo": tipo,
        "sessao_id": sessao_id,
        "agente": "benchmark-agent",
        **dados
    }
    # Salva em arquivo, Supabase, ou apenas print
    print(json.dumps(entrada))

# No loop:
log_evento("llm_start", {"iteracao": i, "tokens_historico": len(str(historico))})
response = client.messages.create(...)
log_evento("llm_end", {"stop_reason": response.stop_reason})

if response.stop_reason == "tool_use":
    log_evento("tool_start", {"ferramenta": block.name, "input": block.input})
    resultado = await executar_ferramenta(...)
    log_evento("tool_end", {"preview": resultado[:200]})
```

## O que você coleta ao longo do tempo

Com 100 sessões logadas:
- Quais queries de busca produzem resultados ruins (iterações extras)
- Qual perfil de founder chega no `max_iterations` sem conclusão
- Quanto custa em média por tipo de ideia (simples vs nicho específico)
- Qual ferramenta tem maior taxa de erro

Isso é o que transforma um produto em produto que melhora com uso.

## Em LangChain

```python
from langchain.callbacks.base import BaseCallbackHandler

class MeuObserver(BaseCallbackHandler):
    def on_tool_start(self, serialized, input_str, **kwargs):
        log_evento("tool_start", {"ferramenta": serialized["name"]})
    
    def on_tool_end(self, output, **kwargs):
        log_evento("tool_end", {"preview": str(output)[:200]})
```

## Relacionado

- [[max-iterations]] — o dado mais importante para monitorar
- [[03-langchain-patterns]] — callbacks formalizados no LangChain
- [[loop-explicito-vs-abstraido]] — por que loop explícito facilita callbacks
- [[benchmark-agent]] — primeiro agente que precisa de observabilidade
