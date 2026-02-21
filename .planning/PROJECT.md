# OpenCode Fork

## What This Is

Fork mantido do OpenCode com três customizações: suporte a subagentes (Task tool para orquestrar agentes paralelos), ajustes nos scripts de instalação para funcionar com NFS via Bun, e patches para carregamento de pacotes. O objetivo é manter o fork sincronizado com o upstream e corrigir bugs na implementação de subagentes.

## Core Value

Subagentes funcionam corretamente em todas as interfaces (CLI e Desktop) sem consumir cotas do GitHub Copilot — comportamento idêntico ao do upstream para chamadas regulares.

## Requirements

### Validated

- ✓ CLI com suporte a subagentes (Task tool) — existing
- ✓ Subagentes na CLI não consomem cota do Copilot — existing (funciona corretamente)
- ✓ Scripts de install adaptados para NFS/Bun — existing
- ✓ Carregamento de pacotes via Bun com patches para NFS — existing
- ✓ Monorepo TypeScript com Bun runtime — existing
- ✓ Hono HTTP server + SSE para comunicação CLI/Web/Desktop — existing
- ✓ Provider system com 20+ provedores AI incluindo GitHub Copilot — existing
- ✓ SolidJS frontend compartilhado entre web e desktop (Tauri) — existing
- ✓ Agent system com agentes built-in e custom (markdown files) — existing
- ✓ SQLite + Drizzle ORM para armazenamento local — existing

### Active

- [ ] Corrigir bug: subagentes na versão desktop consomem cota do Copilot (na CLI não consomem)
- [ ] Manter fork sincronizado com upstream do OpenCode
- [ ] Preservar e manter scripts de install para NFS/Bun

### Out of Scope

- Novas features além das que existem no upstream — foco é manutenção do fork
- Refatoração do codebase upstream — não é nosso repositório
- Correção de bugs que não afetam as customizações do fork — reportar upstream

## Context

- O projeto é um fork do OpenCode (github.com/anomalyco/opencode)
- Branch atual: `subs` (branch do fork com customizações)
- Branch padrão do upstream: `dev`
- A versão desktop usa Tauri v2 (`packages/desktop/`) que embarca a web app (`packages/app/`)
- O Copilot auth plugin está em `packages/opencode/src/plugin/copilot.ts` — usa fetch interceptor para reescrever headers
- O bug de cotas no desktop sugere que a chamada de API de subagentes no desktop passa headers/auth diferentes da CLI
- O codebase tem ~142 usos de `any`, especialmente no provider/plugin layer — área relevante para o bug
- O Copilot plugin fetch interceptor é classificado como "fragile area" no codebase concerns

## Constraints

- **Compatibilidade upstream**: Mudanças devem ser mínimas e isoladas para facilitar merges futuros com o upstream
- **Runtime**: Bun 1.2+ é o runtime primário — todas as soluções devem funcionar com Bun
- **NFS**: Scripts de install devem funcionar em filesystems NFS (sem operações que NFS não suporta como locks exclusivos)
- **Copilot compliance**: Subagentes não devem consumir cotas segundo as regras do próprio GitHub Copilot

## Key Decisions

| Decision                             | Rationale                                                                             | Outcome   |
| ------------------------------------ | ------------------------------------------------------------------------------------- | --------- |
| Fork ao invés de contribuir upstream | Customizações específicas de workflow (subagentes, NFS) que upstream pode não aceitar | — Pending |
| Branch `subs` para customizações     | Separar mudanças do fork do código upstream                                           | ✓ Good    |
| Manter sync com `dev` do upstream    | Receber bugfixes e features sem reescrever                                            | — Pending |

---

_Last updated: 2026-02-21 after initialization_
