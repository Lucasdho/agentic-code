---
tags: [conceito, claude-code, extensibilidade, prompts, reutilização]
aliases: [skills, skill system, habilidades, prompts reutilizáveis]
---

# Skills — instruções reutilizáveis carregadas sob demanda

> **Skills** são arquivos Markdown com instruções especializadas que o agente carrega quando precisa.
> São o mecanismo de extensão via contexto — ao contrário dos [[hooks]] (que rodam código), skills ensinam o LLM como agir.

## O que é uma skill

Uma skill é um arquivo `SKILL.md` (ou similar) que contém:
- Instruções detalhadas sobre como executar uma tarefa específica
- Checklists, exemplos, padrões a seguir
- Referências a outros arquivos ou ferramentas

```markdown
# SKILL.md — swift-dev

Use when writing any Swift or iOS code.

## Rules
1. Use Swift Testing (#expect), not XCTest
2. Prefer native SwiftUI over third-party libs
3. Always check Context.md first

## Checklist
- [ ] Read existing code before editing
- [ ] Run build after changes
```

## Como o agente usa skills

O agente invoca uma skill explicitamente (`Skill(name: "swift-dev")`).
O [[harness]] carrega o conteúdo do arquivo e injeta no contexto.

```
Usuário: "adiciona um botão de salvar"
Agente: → invoca skill "swift-dev"
Harness: → carrega /skills/swift-dev/SKILL.md → injeta no contexto
Agente: → agora sabe as convenções do projeto antes de escrever código
```

## Skills vs System Prompt

| | System Prompt | Skill |
|---|---|---|
| Quando carrega | Sempre, toda sessão | Só quando invocado |
| Tamanho | Limitado (tokens custam) | Pode ser extenso |
| Especialização | Geral | Específica por domínio |
| Manutenção | Arquivo único | Modular, um arquivo por skill |

## Progressive Disclosure aplicada a skills

Skills implementam [[progressive-disclosure]]: o contexto relevante só é carregado quando necessário. Isso evita encher o context window com instruções de Swift quando você está fazendo uma tarefa de banco de dados.

```
Sessão de debug de banco:
→ swiftdata-pro skill carregada ✅
→ swift-testing-pro NÃO carregada ✅ (economiza tokens)

Sessão de escrita de testes:
→ swift-testing-pro skill carregada ✅
→ swiftdata-pro NÃO carregada ✅
```

## Skill como roteador

Uma skill pode redirecionar para sub-skills. Ex: `swift-dev` é um roteador que detecta o tipo de tarefa e carrega o guia especializado:

```
swift-dev (master router)
├── swiftui-pro          ← criando views
├── swift-testing-pro    ← escrevendo testes
├── swiftdata-pro        ← banco de dados
├── swift-concurrency-pro ← async/await
└── ios-debugger-agent   ← depurando
```

## Onde ficam as skills

```
~/.claude/skills/          ← skills globais (qualquer projeto)
.claude/skills/            ← skills locais (só este projeto)
/mnt/skills/public/        ← skills compartilhadas (plataforma)
```

## Relacionado

- [[hooks]] — extensão via código, não via contexto
- [[harness]] — quem carrega e injeta as skills
- [[progressive-disclosure]] — o princípio por trás do carregamento sob demanda
- [[tool-description-as-prompt]] — skills são "prompts de nível acima" do tool description
