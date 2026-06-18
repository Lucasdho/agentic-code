---
tags: [conceito, git, isolamento, paralelismo, claude-code]
aliases: [worktree, git worktree, isolamento de agente]
---

# Worktree — isolamento para agentes paralelos

> Um **git worktree** é uma cópia de trabalho separada do mesmo repositório, em uma branch dedicada.
> Em contextos agenticos, é o mecanismo que permite múltiplos agentes trabalharem no mesmo repo **sem se sobrescreverem**.

## O problema que resolve

Quando o [[harness]] despacha agentes em paralelo (ex: um refatora autenticação, outro adiciona testes), ambos editariam os mesmos arquivos na mesma working tree. Resultado: conflitos, dados corrompidos, trabalho perdido.

```
Sem worktree:
Agent A edita auth.py ──┐
Agent B edita auth.py ──┘ → CONFLITO

Com worktree:
Agent A → worktree/branch-a/auth.py  ✅
Agent B → worktree/branch-b/auth.py  ✅
```

## Como funciona

```bash
# Git cria uma pasta separada com a branch isolada
git worktree add .claude/worktrees/feature-x -b feature-x

# Agent trabalha em .claude/worktrees/feature-x/
# O repo original em / fica intocado

# Quando termina:
git worktree remove .claude/worktrees/feature-x
```

## Em Claude Code

O harness do Claude Code tem o comando `EnterWorktree` que:
1. Cria `.claude/worktrees/<nome>/` como nova worktree
2. Cria uma branch isolada automaticamente
3. Move a sessão para trabalhar nessa pasta
4. Limpa automaticamente se nenhuma mudança foi feita

```
EnterWorktree(name: "fix-login-bug")
→ cria branch worktree-fix-login-bug
→ sessão passa a operar em .claude/worktrees/fix-login-bug/
→ edições não afetam main até o merge
```

## Quando usar

| Situação | Worktree? |
|---|---|
| Agentes paralelos no mesmo repo | ✅ sempre |
| Sessão background enquanto você edita | ✅ evita conflitos |
| Só lendo/pesquisando código | ❌ não precisa |
| Sessão única sem paralelismo | ❌ YAGNI |

## Worktree vs Branch simples

| | Worktree | Branch simples |
|---|---|---|
| Múltiplos checkouts simultâneos | ✅ | ❌ |
| Troca de contexto sem `git stash` | ✅ | ❌ |
| Overhead de disco | Mínimo | Zero |
| Isolamento para agentes paralelos | ✅ | ❌ |

## Relacionado

- [[harness]] — quem cria e gerencia os worktrees
- [[callbacks]] — observar o que acontece em cada worktree
- [[state-memoria]] — o state de cada agente é isolado na worktree
