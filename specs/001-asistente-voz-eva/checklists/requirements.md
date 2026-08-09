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

- [x] La especificación cumple la constitución del proyecto (v2.0.0)

## Notas

**Marcadores de clarificación pendientes (intencional).** El usuario pidió explícitamente
"ambigüedades a marcar explícitamente para resolver en la fase de aclaración". Se identificaron 7
y ninguna se descartó: AMB-01, AMB-02 y AMB-03 quedan abiertas como bloqueantes (AMB-01 con
marcador inline en FR-028); AMB-04 a AMB-07 quedan resueltas con un supuesto documentado y
fundamentado, revisable en `/speckit-clarify`. Este ítem se marca incompleto para reflejar el
estado real, no porque falte trabajo: resolverlas acá duplicaría la fase de aclaración que la
constitución exige.

**Conflicto constitucional resuelto.** La US4 y los FR-034 a FR-039 contradecían el Principio X de
la constitución v1.0.0, que prohibía toda interfaz gráfica en fase 1. La enmienda a **v2.0.0**
(bump MAJOR) redefinió el principio para permitir una única superficie mínima de estado bajo cinco
condiciones acumulativas. La spec incorpora una tabla que mapea cada condición a su requisito, y el
FR-053 recoge las exclusiones del principio. El gate de Constitution Check del plan-template ya no
tiene motivo para fallar por este punto.

**Verificado sin conflicto.** Los presupuestos de recursos de esta spec son iguales o más estrictos
que los constitucionales (<1% vs <3% de núcleo, <150 MB vs <250 MB en reposo, ≤1,5 GB en ambos tras
la enmienda), y un límite más estricto satisface el Principio VI sin necesitar excepción
documentada. La remoción del LLM del alcance de fase 1 en la v2.0.0 coincide con FR-013 y con la
sección *Fuera de alcance* de esta spec.

**Dependencia externa señalada.** NFR-014 a NFR-016 y SC-002 a SC-004 dependen de un conjunto de
frases de referencia en español rioplatense que todavía no existe y que debe proveer el usuario.
Sin ese conjunto esos criterios no son verificables, y el Principio XIV lo exige igual para admitir
herramientas al registry.

- Items marked incomplete require spec updates before `/speckit-clarify` or `/speckit-plan`
