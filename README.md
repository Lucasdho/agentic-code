# Agentic Engineering Notes

Notas públicas de estudo sobre engenharia de agentes de IA.

Documentação do processo de aprendizado: do zero até arquitetura de produção de multi-agent systems.

## 📖 Leia online

👉 **[Abrir o Knowledge Graph](https://lucasdho.github.io/agentic-code)**

O site usa [Quartz v4](https://quartz.jzhao.xyz/) — renderiza wikilinks, grafo de conexões entre conceitos e backlinks automáticos.

## Módulos

| # | Título | Status |
|---|--------|--------|
| 00 | O que é um agente | ✅ |
| 01 | Function calling | ✅ |
| 02 | O loop ReAct na prática | ✅ |
| 03 | Padrões LangChain | ✅ |
| 04 | Padrões CrewAI | 🔜 |
| 05 | Google ADK patterns | 🔜 |
| 06 | Multi-agent open specs | 🔜 |
| 07 | Apple Foundation Models | 🔜 |
| 08 | Arquitetura server-side | 🔜 |
| 09 | Agent map — síntese | 🔜 |

## Estrutura

```
content/
├── MOC-agentes.md          ← hub central do grafo
├── PLANO-DE-ESTUDOS.md
├── 00..09-*.md             ← módulos
├── conceitos/              ← notas atômicas por conceito
└── agentes/                ← specs dos agentes (produto)
```

## Tech stack

- Notas em Markdown com YAML frontmatter
- Wikilinks `[[note]]` entre conceitos
- Gerado com [Quartz v4](https://quartz.jzhao.xyz/)
- Deploy automático via GitHub Actions → GitHub Pages
