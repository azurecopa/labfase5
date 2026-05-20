# @po Validation — Story 1.5 (Workshop Guide VM)

> **Validator:** Pax (@po) · **Date:** 2026-05-20 · **Verdict:** ✅ **GO (9.5/10)**
> **Story:** [`docs/stories/1.5.story.md`](../stories/1.5.story.md)
> **Source model:** [`docs/GUIA-EVENTO.md`](../GUIA-EVENTO.md) (PaaS, 486 linhas)

## Veredito consolidado

| Item | Valor |
|---|---|
| **Score** | **9.5 / 10** |
| **Confidence** | **High** |
| **Verdict** | **GO** |
| **Status transition** | Draft → **Ready** |
| **Próximo agente** | `@dev *develop 1.5` |

## 10-Point Checklist

| # | Critério | Resultado |
|---|---|---|
| 1 | Título claro/objetivo | ✅ |
| 2 | Story bem formada (As a/I want/So that) | ✅ |
| 3 | AC testáveis (11 ACs c/ thresholds e mapping) | ✅ |
| 4 | Tasks executáveis (16 tasks T-1..T-16) | ✅ |
| 5 | Dependências mapeadas (6 source docs) | ✅ |
| 6 | Estimativa de complexidade | ✅ |
| 7 | Valor de negócio explícito (paridade c/ guia PaaS) | ✅ |
| 8 | Riscos documentados (5 c/ mitigação) | ✅ |
| 9 | DoD / smoke claros (T-16 + Testing) | ✅ |
| 10 | Alinhamento c/ EPIC-001 (extends scope, justificado) | ✅ |

## Anti-Hallucination (Art. IV)

Verificações:

| Reference | Status |
|---|---|
| Story 1.1 (playbook source) | ✅ Existe, Ready, validada anteriormente |
| GUIA-EVENTO.md (modelo de estilo, 486 linhas) | ✅ Existe |
| DEPLOY.md (topologia + NSG) | ✅ Existe |
| DEPLOY_IIS_SIMPLIFICADO.md (passos IIS) | ✅ Existe |
| DEPLOY_AZURE.md (guia detalhado fallback) | ✅ Existe |
| fifa2026-api/.env.example (template literal) | ✅ Existe |
| audit 2026-05-15 (contagens 104/49/17) | ✅ Existe |
| Custo VM B2s ≈ $30/mês | ✅ Standard Azure pricing |
| Jump host pattern vs Bastion $87/mo | ✅ Real |
| Bacpac stale (D3 owner-owned) | ✅ Alinhado com audit |

**Sem invenção detectada.**

## Reforço de decisões @sm

| Decisão @sm | Posição @po |
|---|---|
| Escopo: SÓ Story 1.1 (sem 1.2-1.4) | **REFORÇO** — full journey teria 1500+ linhas, dilui foco "deploy VM" |
| Jump host via vm-front como default (gratuito) | **REFORÇO** — Bastion $87/mês inflaciona custo do aluno; jump host alinha c/ constraint workshop |
| Bacpac stale callout ⚠️ Fase 1 | **REFORÇO** — D3 é owner-owned, callout encaminha aluno ao bacpac correto do fork |
| Tamanho alvo 400-700 linhas | **OK** — range razoável (PaaS=486, VM um pouco maior por causa de RDP/IIS/SQL install) |

## Issues registradas

### Should-Fix opcional (não-bloqueador)

- **S-1 — AC-2 conta "7 fases" mas lista 8** (Fase 0 a Fase 7 = 8). Cosmético. @dev ajusta contagem ao escrever — "8 fases" se incluir Pós-jogo, ou renumera para 7 (fundindo Final + Pós-jogo).

### Sem issues HIGH/CRITICAL.

## Handoff

```yaml
next_agent: @dev
next_command: *develop 1.5
condition: Story Ready (GO 9.5/10)
alternatives:
  - agent: @sm, command: *draft, condition: rework (não aplicável)
```

Branch `feature/1.5-vm-workshop-guide` preparada por @sm — @dev commita direto.
