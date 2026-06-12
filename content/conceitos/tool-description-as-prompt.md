---
tags: [conceito, fundamentos, prompting, function-calling]
aliases: [descrição de ferramenta, tool description, description as prompt]
---

# Descrição de ferramenta é um prompt

> O campo `description` de uma ferramenta não é documentação para você.
> É um **prompt** que o LLM lê para decidir quando e como usar a ferramenta.

## O que o LLM faz com a descrição

Antes de cada iteração, o LLM recebe:
1. O sistema prompt (o que ele é e o que deve fazer)
2. O histórico de mensagens (o que aconteceu até agora)
3. **A lista de ferramentas com suas descrições**

O LLM lê as descrições e decide:
- Preciso de uma ferramenta agora?
- Qual ferramenta é adequada para o que preciso?
- Quais argumentos devo passar?

## Bom vs ruim

**Ruim — muito vago:**
```json
{
  "name": "busca",
  "description": "Busca coisas na internet"
}
```
O LLM não sabe: busca o quê? Quando? O que não buscar?

**Bom — preciso:**
```json
{
  "name": "busca_web",
  "description": "Busca informações atuais na internet sobre concorrentes, mercado e preços.

Use QUANDO precisar de:
- Features e detalhes de concorrentes específicos
- Preços e modelos de monetização atuais
- Dados de adoção e tamanho de mercado
- Notícias recentes do setor

NÃO use para:
- Informações que já estão no contexto da conversa
- Dados históricos que você já conhece
- Cálculos ou raciocínio que você pode fazer sem busca"
}
```

## Os três componentes de uma boa descrição

1. **O que faz** — uma linha clara
2. **Quando usar** — casos de uso explícitos
3. **Quando NÃO usar** — escopo negativo (tão importante quanto o positivo)

O "NÃO use para" evita que o LLM use a ferramenta desnecessariamente
(= mais tokens, mais custo, mais latência).

## Para o FounderLens

Cada ferramenta precisa de uma descrição que o LLM de análise de mercado
entenda — não você, não o usuário, **o LLM** no contexto do agente:

| Ferramenta | Regra de ouro |
|---|---|
| `busca_web` | Explicitar que é para dados reais que mudam — não para raciocínio |
| `ler_contexto_projeto` | Explicitar quando ler vs quando usar o que já está no histórico |
| `avaliar_rubrica` | Explicitar que recebe critérios + dados, não faz busca |

## Relacionado

- [[01-function-calling]] — protocolo completo onde a descrição aparece
- [[react-pattern]] — o Thought usa as descrições para decidir qual Action tomar
- [[benchmark-agent]] — exemplo de descrição real do `busca_web`
