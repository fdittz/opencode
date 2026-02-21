# Requirements: OpenCode Fork

**Defined:** 2026-02-21
**Core Value:** Subagentes funcionam corretamente em todas as interfaces (CLI e Desktop) sem consumir cotas do GitHub Copilot

## v1 Requirements

### Bug Fix — Desktop Copilot Quota

- [ ] **BUG-01**: Investigar e documentar a diferença no path de chamada de API entre CLI e Desktop para subagentes
- [ ] **BUG-02**: Subagentes na versão desktop não consomem cota do GitHub Copilot (comportamento idêntico à CLI)

## v2 Requirements

### Upstream Sync

- **SYNC-01**: Processo documentado para fazer merge do upstream `dev` no fork `subs` sem perder patches
- **SYNC-02**: Customizações do fork isoladas e identificáveis

### NFS/Bun Install

- **NFS-01**: Scripts de install funcionam em filesystems NFS
- **NFS-02**: Patches de carregamento de pacotes via Bun mantidos e documentados

### Hardening

- **HARD-01**: Testes para o fetch interceptor do Copilot plugin
- **HARD-02**: Reduzir uso de `any` nas áreas de provider/plugin relevantes ao fork

## Out of Scope

| Feature                                | Reason                  |
| -------------------------------------- | ----------------------- |
| Refatoração geral do upstream          | Não é nosso repositório |
| Novas features além do upstream        | Foco é manutenção       |
| Correção de bugs que não afetam o fork | Reportar upstream       |
| Settings pages incompletas             | Bug do upstream         |
| Permission persistence                 | Bug do upstream         |
| Upstream sync (agora)                  | Deferido para v2        |
| NFS/Bun install (agora)                | Deferido para v2        |

## Traceability

| Requirement | Phase   | Status  |
| ----------- | ------- | ------- |
| BUG-01      | Phase 1 | Pending |
| BUG-02      | Phase 1 | Pending |

**Coverage:**

- v1 requirements: 2 total
- Mapped to phases: 2
- Unmapped: 0 ✓

---

_Requirements defined: 2026-02-21_
_Last updated: 2026-02-21 after roadmap creation_
