---
tags: [conceito, arquitetura, contexto, eficiência, tokens]
aliases: [progressive disclosure, divulgação progressiva, lazy context, deferred tools]
---

# Progressive Disclosure — contexto revelado sob demanda

> **Progressive disclosure** é o princípio de não carregar tudo de uma vez.
> Informação, ferramentas e contexto são revelados **quando necessários** — não antes.

## O problema que resolve

Um agente com acesso a 200 ferramentas, 50 skills e 30 arquivos de contexto vai:
- Usar tokens lendo coisas irrelevantes
- Confundir o LLM com excesso de opções
- Ficar mais lento e mais caro por sessão

Progressive disclosure inverte a lógica: comece com o mínimo, carregue mais quando precisar.

## Onde aparece em Claude Code

### Deferred tools (ferramentas diferidas)

Ferramentas são listadas **pelo nome apenas** no contexto inicial. O schema completo (parâmetros, descrição detalhada) só é carregado quando o agente invoca `ToolSearch`:

```
Contexto inicial:
→ "CronCreate, WebFetch, TaskCreate..." (só nomes)

Agente decide que precisa do WebFetch:
→ ToolSearch("select:WebFetch")
→ Harness injeta o schema completo no contexto
→ Agente pode chamar a ferramenta
```

Isso evita que 150 schemas de ferramentas encham o context window em toda sessão.

### Skills carregadas sob demanda

[[Skills]] só entram no contexto quando explicitamente invocadas:

```
Sessão começa → contexto limpo
Tarefa é de Swift → Skill("swift-dev") invocada
Harness injeta conteúdo do SKILL.md → agente agora sabe as convenções
```

### Memory system (memória seletiva)

O MEMORY.md é um índice leve. Detalhes ficam em arquivos separados:

```
MEMORY.md (sempre carregado, ~200 linhas)
→ [User role](user/role.md)
→ [Feedback testing](feedback/testing.md)

Quando relevante → harness lê o arquivo específico
```

## O princípio em sistemas de agente

Progressive disclosure aparece em vários níveis:

| Nível | Padrão preguiçoso |
|---|---|
| Ferramentas | Listar nomes → carregar schema quando usar |
| Skills | Índice → carregar conteúdo quando invocar |
| Memória | MEMORY.md leve → arquivos detalhados quando relevante |
| Subagentes | Não criar até precisar → fork só quando contexto ficaria pesado |
| Contexto | Não ler arquivo → ler só quando o LLM pede |

## Progressive disclosure vs YAGNI

São o mesmo princípio em contextos diferentes:

| Contexto | Princípio |
|---|---|
| Código | YAGNI — não construa o que não vai usar |
| Contexto de LLM | Progressive disclosure — não carregue o que não vai usar |

## Impacto prático

```
Sessão sem progressive disclosure:
→ 150 tool schemas + 20 skills + 10 arquivos de memória
→ ~80.000 tokens só de contexto
→ Cara, lenta, confusa

Sessão com progressive disclosure:
→ Índice de ferramentas (nomes) + MEMORY.md + CLAUDE.md
→ ~8.000 tokens de contexto base
→ Skills e schemas carregados sob demanda: +2.000 tokens quando necessário
```

## Relacionado

- [[skills]] — implementação de progressive disclosure para instruções
- [[harness]] — quem gerencia o que está no contexto em cada momento
- [[hooks]] — podem disparar carregamento de contexto em SessionStart
- [[state-memoria]] — memória de curto vs longo prazo segue o mesmo princípio
- [[tool-description-as-prompt]] — o que está sempre visível vs carregado sob demanda
