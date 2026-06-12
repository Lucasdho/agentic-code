---
tags: [agente, founderlens, deliver, squad, produto]
aliases: [Prompt Generator Squad, squad de prompts, Deliver M7, wave execution]
fase: Deliver — Módulo M7
status: especificado (visão v2/v3)
---

# Prompt Generator Squad

> **Onde:** Deliver, Módulo M7 — o ato final do Double Diamond
> **O que faz:** gera a biblioteca completa de prompts spec-driven para cada história do backlog, em wave execution paralela com dependency awareness

## O que torna isso um squad (não um agente único)

Cada prompt card é uma unidade de trabalho independente.
Um squad pode gerar múltiplos cards em paralelo.
Um agente único seria sequencial — mais lento sem necessidade.

```
                    ┌─ Prompt Writer A → card: autenticação
                    ├─ Prompt Writer B → card: listagem de voos
Orquestrador ───────┤─ Prompt Writer C → card: submissão de claim
                    └─ Prompt Writer D → card: dashboard founder
                    
                    (paralelo, não sequencial)
```

## Wave execution

Respeita dependências: cards que precisam de outros rodam depois.

```
Wave 1: [setup db] [auth] [design system]         ← sem dependências
Wave 2: [api de voos] [api de claims]             ← dependem de db
Wave 3: [tela de voo] [tela de claim]             ← dependem de api
Wave 4: [integração end-to-end] [testes]          ← dependem de tudo
```

Cada wave só começa quando todos os cards da wave anterior estão prontos.

## O que cada Prompt Writer Agent gera

O "prompt card" completo da spec (definido na interaction-spec):

```markdown
## Card: Submissão de Claim

**Contexto herdado:** [header do projeto — persona, plataforma, não-objetivos]

**Tarefa:** Implementar o endpoint POST /claims que recebe os dados do voo
cancelado, valida contra as regras da ANAC, e submete para o parceiro jurídico.

**Não-objetivos agora:** pagamento ao usuário, interface de acompanhamento.

**Guards:** Parar e perguntar se o cálculo de compensação envolver valores
acima de R$2000. Verificar compliance OAB antes de qualquer comunicação ao usuário.

**Definição de pronto:** endpoint retorna 200 com claim_id, dados persistidos
no banco, notificação de confirmação enviada. Testável com curl sem frontend.
```

## Ferramentas do Orquestrador

- `ler_artefato(nome)` — lê specs do Develop para gerar prompts fiéis
- `ler_contexto_projeto()` — lê o header do projeto (herdado por todos os cards)
- `verificar_dependencias(card_id)` — checa se dependências estão resolvidas

## Ferramentas de cada Prompt Writer

- `ler_artefato(nome)` — lê a spec da história específica
- `gerar_prompt_card(historia, contexto)` — produz o card formatado

## Padrões que usa

- [[react-pattern]] — cada Prompt Writer é um agente
- [[state-memoria]] — todos leem do mesmo project_state
- [[loop-explicito-vs-abstraido]] — o orquestrador gerencia o loop de waves
- [[callbacks]] — monitorar quantos cards gerados, taxa de aprovação do founder

## Por que é o módulo mais complexo do FounderLens

1. **Paralelismo** — múltiplos agentes rodando ao mesmo tempo
2. **Dependências** — o grafo de dependências entre cards precisa ser resolvido
3. **Qualidade** — o output é o que o founder vai usar para construir o produto
4. **Custo** — muitas chamadas LLM em paralelo = custo alto se não otimizado

## Relacionado

- [[04-crewai-patterns]] — o modelo de squads que inspira isso
- [[05-google-adk-patterns]] — wave execution e dependency management
- [[development-auditor]] — roda antes, garante que os artefatos estão prontos
- [[PLANO-DE-ESTUDOS]] — módulos que ensinam os padrões para implementar isso
