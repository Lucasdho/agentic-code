---
tags: [plano, índice, meta]
aliases: [Plano de estudos, roadmap de aprendizado]
---

# FounderLens — Plano de Estudos: Código Agentico
**Living doc · v1.0 · junho 2026**

Guia de estudo de código agentico ancorado no FounderLens. Cada módulo tem um doc próprio.
A ordem importa — cada módulo constrói sobre o anterior.

---

## Contexto: o que o FounderLens precisa de agentes

O FounderLens não é um chatbot. É um pipeline estruturado onde agentes executam tarefas
específicas em momentos específicos, sempre com o founder no controle das decisões.

**Agentes já identificados na spec:**

| Agente | Onde aparece | O que faz |
|---|---|---|
| Benchmark Agent | Discover, ingrediente 10 | Busca web → extrai features/preços/gaps dos top 3 concorrentes |
| Market Sizing Agent | Midpoint Audit | Busca dados de mercado reais (não inventa TAM) |
| Validation Auditor | Midpoint Audit | Lê o contexto completo → aplica rubrica R-W-W → retorna score + veredito |
| Development Auditor | Dev Audit | Lê artefatos → aplica 3 lentes (Coerência/Testabilidade/Completude) → patch cirúrgico |
| Prompt Generator Squad | Deliver M7 | Gera prompts por história do backlog em wave execution (paralelo, dependency-aware) |

**O que você já fez manualmente com agentes (sem saber):**
Os três subagentes Haiku (product strategist + UX designer + senior engineer) que leram
todas as specs e geraram a product strategy — você orquestrou isso na mão, no Claude.
O salto que este estudo te dá: código fazendo essa orquestração automaticamente.

---

## Decisão arquitetural em aberto (não mudar a spec antes dos estudos)

Três caminhos possíveis identificados na conversa. Decidir após estudar:

### Opção A — Pure BYOK (spec atual)
- Usuário traz chave Anthropic própria
- Sem servidor
- **Problema**: mata conversão de não-técnicos

### Opção B — Server-side (nova direção preferida)
- FounderLens tem servidor próprio
- Você paga Anthropic, cobra usuário (uso ou assinatura)
- BYOK vira desconto opcional
- **Habilita**: Python frameworks server-side (LangChain, CrewAI, ADK)
- **Requer**: infraestrutura (Supabase? Cloudflare Workers? Railway?)

### Opção C — Apple Foundation Models + Private Cloud Compute
- Tarefas leves: on-device (grátis, privado)
- Tarefas pesadas: Apple Private Cloud Compute
- **Limitação atual**: modelos Apple são menos capazes que Claude para raciocínio complexo
- **Upside**: se Apple lançar modelos mais potentes (WWDC 2026 — checar anúncios)
- **Modelo híbrido**: Foundation Models para UI/tarefas rápidas + Claude no servidor para pipeline

**Recomendação de estudo**: aprender os padrões agnósticos de framework primeiro.
A decisão de onde rodar (Swift nativo vs servidor Python vs híbrido) fica mais clara
depois de entender como os agentes funcionam de verdade.

---

## Mapa de estudos

```
founderlens-specs/learning/
│
├── PLANO-DE-ESTUDOS.md          ← este arquivo
│
├── FUNDAMENTOS (estudar primeiro)
│   ├── 00-o-que-e-um-agente.md     ✅
│   ├── 01-function-calling.md      ✅
│   └── 02-react-loop.md            ✅
│
├── FRAMEWORKS COMO PROFESSORES DE PADRÃO
│   ├── 03-langchain-patterns.md    ✅
│   ├── 04-crewai-patterns.md       ✅
│   └── 05-google-adk-patterns.md   ✅
│
├── REFERÊNCIAS DE PRODUTO E PLATAFORMA
│   ├── 06-multica-openspec-map.md  ✅
│   └── 07-apple-foundation-models.md   ⏳ (incorporado em 08)
│
├── ARQUITETURA  ← próximo passo crítico antes de codar
│   └── 08-arquitetura-server-side.md   ⏳
│
└── SÍNTESE  ← guia de implementação por onde começar
    └── 09-founderlens-agent-map.md     ⏳
```

**Progresso: 7/10 módulos. Faltam 2 lições antes de codar.**

---

## Módulo por módulo

### FUNDAMENTOS

**00 — O que é um agente**
O salto conceitual: de chamada LLM linear para loop com decisão. O padrão ReAct
(Reason + Act). Como o FounderLens já tem esse padrão na spec sem chamar de agente.

**01 — Function calling**
O mecanismo central de todo agente: o LLM produz JSON de intenção, seu código executa.
Exemplos em Anthropic API e Apple Foundation Models. O Benchmark Agent do FounderLens
como caso concreto.

**02 — O loop ReAct na prática**
Como os ciclos de raciocínio funcionam em código real. Estado entre iterações.
Quando o loop para. Casos de erro e retry. Exemplo: o Benchmark Agent completo
passando por 3-4 iterações de busca.

---

### FRAMEWORKS COMO PROFESSORES DE PADRÃO

Nenhum desses vai para dentro do FounderLens diretamente. São professores de padrão.
Se a Opção B (servidor) for escolhida, alguns podem virar dependência real server-side.

**03 — LangChain patterns**
O vocabulário base: Chains, Tools, Memory, Callbacks, Agents. Por que LangChain importa:
qualquer tutorial, artigo ou Stack Overflow sobre agentes usa essa terminologia.
Padrões a extrair: Tool definition schema, Memory tipos (Buffer, Summary, Vector),
Callback hooks para observabilidade.

**04 — CrewAI patterns**
O modelo de squads: Agent (com role + backstory + tools), Task (com output esperado),
Crew (orquestra). Process: sequential vs hierarchical vs parallel.
Mapeia diretamente para os auditors do FounderLens e o squad do Deliver.

**05 — Google ADK patterns**
Foco em produção: state machines para agentes longos, handoffs entre agentes,
evaluation loops, retry policies. Mais verboso que CrewAI mas mais robusto para
pipelines críticos. Padrões a extrair: Agent-as-Tool (um agente chama outro como
ferramenta), Session state management.

---

### REFERÊNCIAS DE PRODUTO E PLATAFORMA

**06 — Multica + OpenSpec: o que a competição já construiu**
Multica (19k ⭐): plataforma de agentes como teammates — assign issue → agente executa.
Inspiração para o squad de Deliver no v3.
OpenSpec (52k ⭐): spec-driven development como slash commands. Competidor direto do
módulo M7 (Prompts) do Deliver. O que o FounderLens faz diferente: pipeline
Discover→Define→Develop antes do Deliver + UX nativa + co-founder filosófico (humano decide).

**07 — Apple Foundation Models + Private Cloud Compute**
O que a Apple lançou e onde está hoje. Como o framework `FoundationModels` funciona em
Swift. Tool calling nativo. O que roda on-device vs Private Cloud Compute. Limites atuais
de capacidade de raciocínio. Anúncios do WWDC 2026 (checar antes de escrever este doc).

---

### ARQUITETURA

**08 — Arquitetura server-side para o FounderLens**
Se a Opção B for escolhida: como estruturar o servidor. Candidatos: Supabase Edge
Functions (TypeScript), Cloudflare Workers, Railway (Python). Como os frameworks Python
se encaixam server-side. Como o app iOS vira thin client. Como BYOK coexiste com
servidor como opção de desconto. Custos estimados de API por sessão do pipeline.

---

### SÍNTESE

**09 — FounderLens Agent Map**
Conecta tudo ao produto. Para cada agente identificado na spec: qual padrão usa,
qual ferramenta precisa, onde roda (on-device / servidor / híbrido), como passa
contexto pro próximo. O grafo de dependências dos agentes do Deliver. Este doc
é a entrada para a decisão arquitetural final.

---

## Como usar estes docs

Cada doc segue a estrutura:
1. **Conceito central** — o que é e por que importa
2. **Como funciona** — mecanismo com exemplos de código
3. **Padrão a extrair** — o que levar para o FounderLens
4. **Mapeamento ao produto** — onde aparece na spec concreta

Os docs de frameworks (03-05) têm exemplos em Python porque é onde os frameworks vivem.
Isso é intencional — você está aprendendo padrões, não copiando código.

---

## Referências externas (já na tech-references.md)

- TÂCHES CC Resources — padrões de skills, subagents, context handoff
- GSD Core — STATE.md, CONTEXT.md, wave execution, verifier agent

## Novos links a estudar (adicionados nesta conversa)

| Link | Categoria | Prioridade |
|---|---|---|
| github.com/Fission-AI/OpenSpec | Competidor / inspiração M7 | Alta |
| github.com/multica-ai/multica | Inspiração v3 squad | Média |
| github.com/crewAIInc/crewAI | Framework / padrão CrewAI | Alta |
| github.com/langchain-ai/langchain | Framework / vocabulário base | Alta |
| github.com/google/adk-python | Framework / produção | Média |
| github.com/mergisi/awesome-openclaw-agents | Catálogo de referência | Baixa |
| github.com/github/spec-kit | Competidor (mais pesado) | Baixa |

---

*Última atualização: junho 2026*
