# Specification Quality Checklist: Eva — Asistente de Voz Local para Escritorio

**Purpose**: Validate specification completeness and quality before proceeding to planning
**Created**: 2026-08-09
**Feature**: [spec.md](../spec.md)

## Content Quality

- [x] No implementation details (languages, frameworks, APIs)
- [x] Focused on user value and business needs
- [x] Written for non-technical stakeholders
- [x] All mandatory sections completed

## Requirement Completeness

- [ ] No [NEEDS CLARIFICATION] markers remain — **diferido a propósito**, ver Notas
- [x] Requirements are testable and unambiguous
- [x] Success criteria are measurable
- [x] Success criteria are technology-agnostic (no implementation details)
- [x] All acceptance scenarios are defined
- [x] Edge cases are identified
- [x] Scope is clearly bounded
- [x] Dependencies and assumptions identified

## Feature Readiness

- [x] All functional requirements have clear acceptance criteria
- [x] User scenarios cover primary flows
- [x] Feature meets measurable outcomes defined in Success Criteria
- [x] No implementation details leak into specification

## Governance

- [ ] La especificación cumple la constitución del proyecto — **NO**, ver Notas

## Notas

**Marcadores de clarificación pendientes (intencional).** El usuario pidió explícitamente
"ambigüedades a marcar explícitamente para resolver en la fase de aclaración". Se identificaron 7
y ninguna se descartó: AMB-01, AMB-02 y AMB-03 quedan abiertas como bloqueantes (AMB-01 con
marcador inline en FR-028); AMB-04 a AMB-07 quedan resueltas con un supuesto documentado y
fundamentado, revisable en `/speckit-clarify`. Este ítem se marca incompleto para reflejar el
estado real, no porque falte trabajo: resolverlas acá duplicaría la fase de aclaración que la
constitución exige.

**Conflicto constitucional sin resolver.** La US4 y los FR-034 a FR-039 (interfaz gráfica mínima)
contradicen el Principio X de la constitución, que prohíbe overlays y widgets en fase 1. La
especificación documenta el conflicto y las dos vías de salida (enmendar la constitución con bump
MAJOR, o recortar US4 a una feature posterior). **Este ítem bloquea `/speckit-plan`**: el
plan-template incluye un gate de Constitution Check que va a fallar mientras el conflicto siga
abierto.

**Verificado sin conflicto.** Los presupuestos de recursos y latencia que pide el usuario son más
estrictos que los constitucionales (<1% vs <3% de núcleo, <150 MB vs <250 MB en reposo, ≤1,5 GB vs
≤3 GB por turno), y un límite más estricto satisface el Principio VI sin necesitar excepción
documentada.

**Dependencia externa señalada.** NFR-014 a NFR-016 y SC-002 a SC-004 dependen de un conjunto de
frases de referencia en español rioplatense que todavía no existe y que debe proveer el usuario.
Sin ese conjunto esos criterios no son verificables, y el Principio XIV lo exige igual para admitir
herramientas al registry.

- Items marked incomplete require spec updates before `/speckit-clarify` or `/speckit-plan`
