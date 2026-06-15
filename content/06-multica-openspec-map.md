---
tags: [produto, competição, referência, multica, openspec, módulo]
aliases: [Multica patterns, OpenSpec patterns, competidores, referência de produto]
módulo: "06"
anterior: "[[05-google-adk-patterns]]"
próximo: "07-apple-foundation-models"
---

# 06 — Multica + OpenSpec: o que a competição construiu
**Referências de produto e plataforma · v1.0 · junho 2026**

---

## Por que estudar competidores antes de escrever código

Você pode gastar semanas implementando algo que o mercado já provou que não funciona —
ou que funciona mas com um tradeoff fatal que só aparece em produção. Olhar para o
que outros construíram antes de escrever código é a versão de engenharia de software
do Benchmark Agent: você não inventa TAM, você pesquisa.

Dois projetos resolvem problemas adjacentes ao FounderLens e valem análise detalhada:

- **[Multica](https://github.com/multica-ai/multica)** (19k ⭐) — coordenação entre humanos e agentes de código
- **[OpenSpec](https://github.com/Fission-AI/OpenSpec)** (52k ⭐) — spec antes de código, com slash commands

Nenhum dos dois é um concorrente direto. Ambos são professores de produto.

---

## Multica — agentes como teammates no board

### O que é

Plataforma open-source para transformar coding agents em membros do time.
Você cria uma issue, atribui a um agente como atribuiria a um colega, e o agente
executa: escreve código, reporta blockers, atualiza o status.

```
Desenvolvedor → cria issue "Adicionar autenticação OAuth"
             → atribui ao agente Claude Code
             → agente pega a task, executa no seu machine via daemon
             → posta comentários de progresso no board
             → fecha a issue quando termina
```

### Arquitetura real (do repo)

```
┌──────────────┐     ┌──────────────┐     ┌────────────────┐
│   Next.js    │────>│  Go Backend  │────>│  PostgreSQL 17  │
│   Frontend   │<────│ (Chi + WebSocket)│<──│  (pgvector)    │
└──────────────┘     └──────┬───────┘     └────────────────┘
                            │
                     ┌──────┴───────┐
                     │ Agent Daemon │  ← roda na sua máquina
                     └──────────────┘    código nunca sai do seu ambiente
                     detecta automaticamente: claude, codex, openclaw,
                     opencode, hermes, gemini, pi, cursor-agent
```

O ponto importante de arquitetura: **o código nunca passa pelo servidor Multica**.
O backend só coordena estado de tasks e transmite eventos. Execução é local.
Multica é uma camada de coordenação, não de execução.

### Skills — o que mais importa para o FounderLens

O conceito mais interessante do Multica não é a issue board — é o **sistema de skills**.

```
multica/skills-lock.json  ← como package-lock.json, mas para skills de agentes
```

Cada solução que um agente encontra pode ser transformada em uma skill reutilizável
para o time inteiro. Se o agente descobriu como fazer deploy em staging, essa
sequência vira uma skill versionada. Na próxima issue de deploy, o agente já sabe.

```
Competência                  →  skill reutilizável versionada
"Como fazer migration segura" → migration_safe_v2.skill
"Como testar endpoint X"      → test_endpoint_x.skill
```

O Multica está resolvendo o problema do **compound knowledge**: agentes que ficam
cada vez mais capazes ao longo do tempo no contexto do time.

**Padrão a extrair:** o squad de [[prompt-generator-squad]] do FounderLens deveria
acumular skills da mesma forma. Cada prompt card gerado com sucesso para um tipo de
história (autenticação, CRUD, integração de API) é um template que o squad pode
reutilizar em projetos futuros. A biblioteca de prompts do Deliver v2 não é só
storage — é uma skill library que melhora com uso.

### O que Multica não faz

Multica **assume que você sabe o que construir**. A issue já está escrita. O backlog
já existe. O produto já foi definido. Multica resolve "como executar bem o que já
foi decidido", não "o que decidir".

```
Multica começa aqui:
[Issue criada] → [Agente executa] → [Code merged]

FounderLens precisa do que veio antes:
[Problema do founder] → [Discover] → [Define] → [Valida] → [Develop] → [Issue criada] → ...
```

Para o founder solo que ainda está descobrindo o que construir, Multica não ajuda.
É uma ferramenta para times de engenharia, não para founders pré-produto.

### O que inspirar para o v3

O plano atual do FounderLens tem o Deliver como M7 (geração de prompts). O v3 do
produto poderia usar exatamente o modelo de Multica: o founder "atribui" histórias
do backlog a agentes, que executam de forma autônoma e reportam progresso. O que
Multica faz para engenheiros, o FounderLens poderia fazer para founders não-técnicos.

---

## OpenSpec — spec antes de código, via slash commands

### O que é

Framework open-source (TypeScript, 52k ⭐) que adiciona uma camada de spec entre
a ideia do desenvolvedor e o código. A interação toda acontece via slash commands
dentro do seu AI assistant preferido.

```
/opsx:propose "add dark mode"
→ cria automaticamente:
   openspec/changes/add-dark-mode/
   ├── proposal.md    ← por que estamos fazendo isso
   ├── specs/         ← requisitos e cenários
   ├── design.md      ← abordagem técnica
   └── tasks.md       ← checklist de implementação

/opsx:apply
→ implementa as tasks em sequência

/opsx:archive
→ arquiva em openspec/changes/archive/
→ atualiza as specs do projeto
```

Funciona com 25+ AI assistants (Claude Code, Cursor, Copilot, Gemini...).
A filosofia: **fluido, não rígido — iterativo, não waterfall**.

### Delta specs — a inovação central

A maioria dos sistemas de spec trata cada mudança como independente. OpenSpec introduz
**delta specs**: cada change descreve o que está mudando em relação ao estado atual
das specs do projeto.

Isso é importante para projetos brownfield (código que já existe). Em vez de
re-escrever toda a spec do projeto a cada feature, você descreve o delta:
"assumindo que o sistema de auth já existe, o que muda quando adicionamos OAuth?"

**Padrão a extrair:** o FounderLens também poderia adotar delta specs entre as
fases iterativas. Quando o founder volta ao produto depois do lançamento e quer
expandir uma feature, o input para o Develop não é uma spec nova — é um delta
em relação à spec que já existe no `project_state.json`.

### A estrutura de change folder = o Develop do FounderLens

Olhando a estrutura do OpenSpec, ela é surpreendentemente parecida com o que o
Develop do FounderLens produz:

| OpenSpec change folder | FounderLens Develop |
|---|---|
| `proposal.md` — por que estamos fazendo | Ingrediente 01: problema e oportunidade |
| `specs/` — requisitos e cenários | Interaction specs por história |
| `design.md` — abordagem técnica | Component spec + stack decisions |
| `tasks.md` — checklist de implementação | Backlog do Deliver |

Isso não é coincidência — é o campo convergindo para o mesmo padrão de "o que
você precisa antes de escrever código". A diferença é de ponto de entrada:
OpenSpec começa na ideia do desenvolvedor, FounderLens começa na dor do founder.

### O que OpenSpec não faz

OpenSpec assume que o desenvolvedor já tem a ideia e já sabe que vale a pena.
`/opsx:propose "add dark mode"` pressupõe que dark mode é a coisa certa a construir.
Não há nenhum mecanismo para questionar se dark mode resolve uma dor real, se há
mercado para isso, ou se é prioridade.

```
OpenSpec começa aqui:
"Quero adicionar X" → spec → código

FounderLens começa aqui:
"Tenho um problema de negócio" → Discover → Define → validação → especificação → código
```

OpenSpec é para desenvolvedores que já decidiram o que construir. FounderLens é
para founders que ainda estão descobrindo.

Outra limitação: OpenSpec é **ferramenta de desenvolvedor**. A interação é via
slash commands em IDEs. Um founder não-técnico não usa Cursor nem Claude Code.
O FounderLens precisa de UX nativa — iOS app, não slash commands num terminal.

### Competidores que o OpenSpec cita (e que valem nota)

**[Kiro](https://kiro.dev)** (AWS) — mais poderoso que OpenSpec, tem steering docs,
specs e auto-tasks integrados ao IDE. Mas: locked ao próprio IDE da AWS,
limitado a modelos Claude. OpenSpec escolheu a rota de ser agnóstico de tool.
FounderLens deveria aprender com esse tradeoff: não ficar locked a nenhum assistente.

**[github/spec-kit](https://github.com/github/spec-kit)** — a versão do GitHub,
mais rigorosa com phase gates, mais pesada para configurar. OpenSpec chama de
"thorougher but heavyweight". O FounderLens tem phase gates explícitos
(Discover / Define / Develop / Deliver) — a questão é não deixar esses gates
virarem friction para o founder.

---

## Onde o FounderLens joga diferente

Depois de ver ambos os projetos, o que torna o FounderLens distinto:

### 1. Upstream de tudo: o co-fundador filosófico

Multica e OpenSpec começam quando o desenvolvedor já sabe o que construir.
FounderLens existe para o momento anterior: quando o founder tem uma ideia bruta
e precisa descobrir se vale a pena e como especificar.

```
[Ideia bruta do founder]
        │
   FounderLens        ← não existe nenhum produto aqui
        │
[Project state validado]
        │
   OpenSpec-like      ← Deliver do FounderLens
        │
   Multica-like       ← v3 do FounderLens
        │
   [Produto lançado]
```

### 2. UX para não-técnicos

OpenSpec e Multica são ferramentas de desenvolvedor. Slash commands em IDEs,
CLI com `multica daemon start`, daemon rodando em background — nenhum founder
não-técnico vai usar isso. O FounderLens precisa de UX de produto consumer,
não de ferramenta de dev.

### 3. Humano decide, agente executa

Multica pode executar issues de forma completamente autônoma (set it and forget it).
OpenSpec pode gerar e aplicar specs sem intervenção. FounderLens tem uma posição
diferente: **o founder sempre decide antes de avançar de fase**. O agente prepara
a análise, o founder aprova. Não é automação, é amplificação.

Isso é uma escolha de produto: você pode construir o FounderLens para rodar de
forma totalmente automática, mas o valor do produto está em forçar a reflexão do
founder — não em eliminar o fundador do processo.

### 4. Pipeline completo vs ferramenta isolada

OpenSpec é uma ferramenta para uma fase (spec antes de código).
Multica é uma ferramenta para uma fase (execução de issues).
FounderLens é um **pipeline completo**: da ideia bruta ao backlog spec'd.

Isso é mais complexo de construir, mas é onde está o moat. Qualquer um pode
copiar o Deliver. Ninguém consegue copiar o Discover + Define + Audit + Develop
+ Deliver em um produto coeso de UX nativa.

---

## Padrão a extrair pro FounderLens

### 1. Skills library como compound knowledge

A skill library do Multica é o que o Deliver do FounderLens deveria acumular.
Cada prompt card gerado com sucesso é um template. Projetos futuros se beneficiam
de projetos passados. Isso é o que diferencia um tool de uma plataforma.

### 2. Change folder = artefatos do Develop

A estrutura do OpenSpec (`proposal.md`, `specs/`, `design.md`, `tasks.md`) é o
que o Develop do FounderLens deveria produzir por história do backlog:

```
project_state/
└── develop/
    └── historia-autenticacao/
        ├── interaction-spec.md
        ├── component-spec.md
        └── tasks.md              ← isso vira input para o Deliver
```

### 3. Delta spec para iterações

Versão 1 do produto lança. Founder quer adicionar feature X no v2. O input
para o Develop não é uma spec nova — é um delta em relação ao `project_state.json`
existente. O [[development-auditor]] já sabe o que existe e só auditoria a mudança.

### 4. Agnosticismo de LLM como princípio

OpenSpec rodou de Claude para Codex para Gemini sem reescrever o produto.
FounderLens deve ser agnóstico de modelo. A Anthropic API hoje, mas amanhã
pode ser Apple Foundation Models on-device para tarefas leves. A spec do agente
(role/goal/expected_output) não deve vazar o nome do modelo.

### 5. Não sair do contexto do founder

OpenSpec vive dentro do IDE do desenvolvedor. O founder não tem IDE.
O FounderLens deve viver onde o founder está: iOS, conversação, não pipeline
de ferramentas. Cada tela do FounderLens deve parecer uma conversa com um
co-fundador, não um formulário de spec.

---

## Mapeamento ao FounderLens

| Conceito | Multica | OpenSpec | FounderLens |
|---|---|---|---|
| Ponto de entrada | Issue criada no board | `/opsx:propose <ideia>` | Ideia bruta do founder |
| Quem usa | Time de engenharia | Desenvolvedor solo | Founder (técnico ou não) |
| UX | Web app + CLI daemon | Slash commands em AI tool | iOS app nativo |
| Fase coberta | Execução (Deliver) | Spec (Develop) | Discover → Define → Audit → Develop → Deliver |
| Humano no loop | Opcional (pode ser full auto) | Opcional | Obrigatório (founder decide cada fase) |
| Reuso de conhecimento | Skills system versionado | Delta specs | Biblioteca de prompt cards (v2) |
| Artefato persistente | Issue board + skills-lock.json | change folders + specs/ | `project_state.json` |
| Modelo de agentes | Coding agents como teammates | AI assistant com slash commands | Auditores + Squad de Prompt Writers |

---

## Ponto de checagem

1. Multica e OpenSpec são os únicos dois projetos relevantes que você estudou aqui.
   O que eles têm em comum — e o que essa interseção te diz sobre onde o mercado
   está convergindo?

2. A skills library do Multica e a spec library do OpenSpec são análogos à biblioteca
   de prompts do FounderLens Deliver. Mas há uma diferença fundamental. Qual?

3. Por que o FounderLens não deveria copiar a UX de slash commands do OpenSpec,
   mesmo sendo mais simples de implementar?

4. O Multica tem um `skills-lock.json` que versiona as skills como o npm versiona
   pacotes. Que problema isso resolve? O FounderLens vai precisar de algo análogo
   quando o Deliver acumular prompt cards de múltiplos projetos?

5. O ponto de entrada do OpenSpec é `/opsx:propose "minha ideia"`. O ponto de
   entrada do FounderLens é... o quê? Formule em uma frase como o founder inicia
   o pipeline — e por que isso define o produto inteiro.

---

## Próximo

→ **[07 — Apple Foundation Models + Private Cloud Compute](./07-apple-foundation-models.md)**
O que a Apple lançou, como o framework `FoundationModels` funciona em Swift,
o que roda on-device vs Private Cloud Compute, e o que isso significa para
a decisão arquitetural A/B/C do FounderLens.

## Relacionados

- [[prompt-generator-squad]] — o Deliver do FounderLens, análogo ao que Multica executa
- [[validation-auditor]] — fase que nem Multica nem OpenSpec tocam
- [[05-google-adk-patterns]] — os padrões de agente que implementam o que este doc descreve
- [[PLANO-DE-ESTUDOS]] — a decisão arquitetural que este doc informa
- [[MOC-agentes]] — mapa completo do grafo

---

*Última atualização: junho 2026*
