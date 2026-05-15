# .githooks — Git hooks versionados

Hooks rastreáveis no repo (vs `.git/hooks/` que é local e não versionado).

## Instalação (uma vez por clone)

```bash
git config core.hooksPath .githooks
```

No Windows, garanta permissão de execução (git-bash respeita o shebang):

```bash
git update-index --chmod=+x .githooks/commit-msg
```

## Hooks

| Hook | Função |
|------|--------|
| `commit-msg` | **AIOX Constitution Art. III** — bloqueia commits de código (`feat/fix/perf/ui/refactor`) sem `[Story X.Y]`, `[EPIC-XXX]` ou o escape `[no-story]`. Tipos `docs/chore/ci/test/style/build/revert` são isentos. |

### Por que

30 commits feitos direto na `main` em 2026-05-08 sem story associada
violaram o Art. III e forçaram a consolidação retroativa da Story 0.10
(ver `docs/qa/gates/2026-05-15-story-0.10-qa-gate.md`). Este gate impede
a recorrência de forma determinística.

### Desabilitar temporariamente (não recomendado)

```bash
git -c core.hooksPath=/dev/null commit ...   # bypass pontual
```
