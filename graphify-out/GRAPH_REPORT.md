# Graph Report - .  (2026-08-09)

## Corpus Check
- Corpus is ~29,721 words - fits in a single context window. You may not need a graph.

## Summary
- 142 nodes · 253 edges · 14 communities (12 shown, 2 thin omitted)
- Extraction: 81% EXTRACTED · 18% INFERRED · 1% AMBIGUOUS · INFERRED: 45 edges (avg confidence: 0.83)
- Token cost: 248,527 input · 0 output

## Community Hubs (Navigation)
- Pipeline de Comandos Spec Kit
- Heurísticas de Análisis y Clarificación
- Principios de Arquitectura de Eva
- Plantillas y Convenciones del Repo
- Checklists y Trazabilidad
- common.ps1: Resolución de Rutas
- Gobernanza y Presupuestos de Eva
- Creación de Features y Branch Naming
- Identidad del Producto Eva
- Alcance y Versionado Constitucional
- Sesión de Clarificaciones

## God Nodes (most connected - your core abstractions)
1. `Project Constitution (.specify/memory/constitution.md)` - 14 edges
2. `Restricciones que condicionan toda decisión técnica` - 14 edges
3. `speckit-implement Command` - 13 edges
4. `speckit-analyze Command` - 12 edges
5. `speckit-converge Command` - 11 edges
6. `Feature Specification (spec.md)` - 11 edges
7. `Extension Hooks System (.specify/extensions.yml)` - 10 edges
8. `Task List (tasks.md)` - 10 edges
9. `Flujo de Desarrollo y Puertas de Calidad` - 10 edges
10. `speckit-plan Command` - 9 edges

## Surprising Connections (you probably didn't know these)
- `Restricciones que condicionan toda decisión técnica` --semantically_similar_to--> `Restricciones Técnicas y Presupuestos (tabla de límites duros)`  [INFERRED] [semantically similar]
  CLAUDE.md → .specify/memory/constitution.md
- `Flujo de trabajo Spec Kit con integración claude y separador '-'` --semantically_similar_to--> `Flujo de Desarrollo y Puertas de Calidad`  [INFERRED] [semantically similar]
  CLAUDE.md → .specify/memory/constitution.md
- `Qué es Eva (resumen en CLAUDE.md)` --semantically_similar_to--> `Eva (asistente de voz local para Omarchy)`  [INFERRED] [semantically similar]
  CLAUDE.md → .specify/memory/constitution.md
- `Flujo de trabajo Spec Kit con integración claude y separador '-'` --conceptually_related_to--> `Workflow 'Full SDD Cycle' (speckit)`  [AMBIGUOUS]
  CLAUDE.md → .specify/workflows/speckit/workflow.yml
- `Estado actual: no hay código de aplicación` --references--> `Constitución de Eva (v1.0.0)`  [EXTRACTED]
  CLAUDE.md → .specify/memory/constitution.md

## Import Cycles
- None detected.

## Hyperedges (group relationships)
- **Spec-Driven Development Lifecycle Pipeline** — _claude_skills_speckit_constitution_skill_speckit_constitution, _claude_skills_speckit_specify_skill_speckit_specify, _claude_skills_speckit_clarify_skill_speckit_clarify, _claude_skills_speckit_plan_skill_speckit_plan, _claude_skills_speckit_tasks_skill_speckit_tasks, _claude_skills_speckit_analyze_skill_speckit_analyze, _claude_skills_speckit_implement_skill_speckit_implement, _claude_skills_speckit_converge_skill_speckit_converge, _claude_skills_speckit_taskstoissues_skill_speckit_taskstoissues, _claude_skills_speckit_checklist_skill_speckit_checklist [EXTRACTED 1.00]
- **Extension Hook Dispatch Protocol (before_*/after_* in .specify/extensions.yml)** — _claude_skills_speckit_analyze_skill_extension_hooks, _claude_skills_speckit_analyze_skill_speckit_analyze, _claude_skills_speckit_checklist_skill_speckit_checklist, _claude_skills_speckit_clarify_skill_speckit_clarify, _claude_skills_speckit_constitution_skill_speckit_constitution, _claude_skills_speckit_converge_skill_speckit_converge, _claude_skills_speckit_implement_skill_speckit_implement, _claude_skills_speckit_plan_skill_speckit_plan, _claude_skills_speckit_specify_skill_speckit_specify, _claude_skills_speckit_tasks_skill_speckit_tasks, _claude_skills_speckit_taskstoissues_skill_speckit_taskstoissues [EXTRACTED 1.00]
- **Core Feature Artifact Set** — _claude_skills_speckit_specify_skill_spec_md, _claude_skills_speckit_plan_skill_plan_md, _claude_skills_speckit_tasks_skill_tasks_md, _claude_skills_speckit_constitution_skill_constitution_md, _claude_skills_speckit_specify_skill_spec_quality_checklist, _claude_skills_speckit_checklist_skill_checklist_file [EXTRACTED 1.00]
- **Frontera de seguridad de ejecución (registry cerrado + confirmación + falla ruidosa + tests)** — _specify_memory_constitution_principio_iii_registry_cerrado, _specify_memory_constitution_principio_xii_confirmacion_acciones_destructivas, _specify_memory_constitution_principio_xvi_falla_ruidosa, _specify_memory_constitution_principio_xiv_tests_obligatorios, _specify_memory_constitution_registry_de_herramientas [INFERRED 0.85]
- **Presupuestos medibles y su verificación por turno** — _specify_memory_constitution_principio_vi_presupuesto_de_recursos, _specify_memory_constitution_principio_vii_latencia_requisito_funcional, _specify_memory_constitution_principio_xv_observabilidad_por_turno, _specify_memory_constitution_restricciones_tecnicas_y_presupuestos [EXTRACTED 1.00]
- **Cadena de artefactos Spec Kit: spec → plan → tasks → checklist orquestada por el workflow** — _specify_templates_spec_template_feature_specification, _specify_templates_plan_template_implementation_plan, _specify_templates_tasks_template_tasks, _specify_templates_checklist_template_checklist, _specify_workflows_speckit_workflow_speckit_workflow [EXTRACTED 1.00]

## Communities (14 total, 2 thin omitted)

### Community 0 - "Pipeline de Comandos Spec Kit"
Cohesion: 0.18
Nodes (30): check-prerequisites.ps1 Prerequisite Script, Constitution Authority Principle, Extension Hooks System (.specify/extensions.yml), speckit-analyze Command, speckit-checklist Command, speckit-clarify Command, Project Constitution (.specify/memory/constitution.md), Placeholder Token Resolution ([ALL_CAPS_IDENTIFIER]) (+22 more)

### Community 1 - "Heurísticas de Análisis y Clarificación"
Cohesion: 0.12
Nodes (17): Detection Passes (Duplication, Ambiguity, Underspecification, Coverage, Inconsistency), Analysis Severity Assignment Heuristic, Dynamic Clarifying Questions (max 5, no pre-baked catalog), Requirement Quality Dimensions (Completeness, Clarity, Consistency, Measurability, Coverage), Ambiguity & Coverage Scan Taxonomy, Five-Question Clarification Quota, Sequential Questioning Loop (one question at a time), Phase N: Convergence Task Section (+9 more)

### Community 2 - "Principios de Arquitectura de Eva"
Cohesion: 0.23
Nodes (17): Capas: dominio / aplicación / infraestructura / interfaz, Flujo de Desarrollo y Puertas de Calidad, II. Núcleo Desacoplado de la Voz, III. Ejecución por Registry Cerrado (NO NEGOCIABLE), IV. Determinismo Antes que Modelo, IX. Separación de Responsabilidades, V. Degradación Elegante, VII. La Latencia es un Requisito Funcional (+9 more)

### Community 3 - "Plantillas y Convenciones del Repo"
Cohesion: 0.20
Nodes (17): I. La Especificación Manda, XVII. Especificación Antes que Código, checklist-template (CHK-NNN por categoría), plan-template (Implementation Plan), spec-template (Feature Specification), Requisitos funcionales FR-NNN con marcador NEEDS CLARIFICATION, Historias de usuario priorizadas e independientemente testeables (P1/P2/P3), Marcador [P] de paralelismo por archivos disjuntos (+9 more)

### Community 4 - "Checklists y Trazabilidad"
Cohesion: 0.14
Nodes (16): Coverage Summary Table & Metrics, Progressive Disclosure Artifact Loading, Strictly Read-Only Operating Constraint, Requirements Inventory (FR-###/SC-### semantic model), Domain Checklist File (FEATURE_DIR/checklists/[domain].md), CHK### Append-Only ID Scheme, Scenario Classification & Coverage (Primary/Alternate/Exception/Recovery/Non-Functional), 80% Traceability Reference Requirement (+8 more)

### Community 5 - "common.ps1: Resolución de Rutas"
Cohesion: 0.22
Nodes (10): Find-SpecifyRoot(), Format-SpecKitCommand(), Get-CurrentBranch(), Get-FeaturePathsEnv(), Get-InvokeSeparator(), Get-Python3Command(), Get-RepoRoot(), Resolve-SpecifyInitDir() (+2 more)

### Community 6 - "Gobernanza y Presupuestos de Eva"
Cohesion: 0.22
Nodes (13): Constitución de Eva (v1.0.0), Governance: enmiendas y versionado semántico de la constitución, VI. Presupuesto de Recursos Explícito y Verificable, XI. Cien por Ciento Local, XV. Observabilidad Mínima por Turno, Restricciones Técnicas y Presupuestos (tabla de límites duros), Sync Impact Report (TEMPLATE → 1.0.0), constitution-template (placeholders de principios y gobierno) (+5 more)

### Community 8 - "Identidad del Producto Eva"
Cohesion: 0.50
Nodes (4): Eva (asistente de voz local para Omarchy), Omarchy (Arch + Hyprland), Qué es Eva (resumen en CLAUDE.md), README — asistente-omarchy (placeholder)

### Community 9 - "Alcance y Versionado Constitucional"
Cohesion: 0.67
Nodes (3): Constitution Scope Guard, Constitution Semantic Versioning Policy (MAJOR/MINOR/PATCH), Sync Impact Report

## Ambiguous Edges - Review These
- `Constitution Semantic Versioning Policy (MAJOR/MINOR/PATCH)` → `Constitution Scope Guard`  [AMBIGUOUS]
  .claude/skills/speckit-constitution/SKILL.md · relation: conceptually_related_to
- `XVII. Especificación Antes que Código` → `Workflow 'Full SDD Cycle' (speckit)`  [AMBIGUOUS]
  .specify/workflows/speckit/workflow.yml · relation: conceptually_related_to
- `Flujo de trabajo Spec Kit con integración claude y separador '-'` → `Workflow 'Full SDD Cycle' (speckit)`  [AMBIGUOUS]
  CLAUDE.md · relation: conceptually_related_to

## Knowledge Gaps
- **10 isolated node(s):** `Scenario Classification & Coverage (Primary/Alternate/Exception/Recovery/Non-Functional)`, `Sync Impact Report`, `Placeholder Token Resolution ([ALL_CAPS_IDENTIFIER])`, `[P] Parallel Task Marker Handling`, `Mark Completed Tasks as [X]` (+5 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **2 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **What is the exact relationship between `Constitution Semantic Versioning Policy (MAJOR/MINOR/PATCH)` and `Constitution Scope Guard`?**
  _Edge tagged AMBIGUOUS (relation: conceptually_related_to) - confidence is low._
- **What is the exact relationship between `XVII. Especificación Antes que Código` and `Workflow 'Full SDD Cycle' (speckit)`?**
  _Edge tagged AMBIGUOUS (relation: conceptually_related_to) - confidence is low._
- **What is the exact relationship between `Flujo de trabajo Spec Kit con integración claude y separador '-'` and `Workflow 'Full SDD Cycle' (speckit)`?**
  _Edge tagged AMBIGUOUS (relation: conceptually_related_to) - confidence is low._
- **Why does `Task List (tasks.md)` connect `Pipeline de Comandos Spec Kit` to `Heurísticas de Análisis y Clarificación`?**
  _High betweenness centrality (0.047) - this node is a cross-community bridge._
- **Why does `Restricciones que condicionan toda decisión técnica` connect `Principios de Arquitectura de Eva` to `Gobernanza y Presupuestos de Eva`?**
  _High betweenness centrality (0.044) - this node is a cross-community bridge._
- **Why does `Project Constitution (.specify/memory/constitution.md)` connect `Pipeline de Comandos Spec Kit` to `Alcance y Versionado Constitucional`?**
  _High betweenness centrality (0.044) - this node is a cross-community bridge._
- **What connects `Scenario Classification & Coverage (Primary/Alternate/Exception/Recovery/Non-Functional)`, `Sync Impact Report`, `Placeholder Token Resolution ([ALL_CAPS_IDENTIFIER])` to the rest of the system?**
  _10 weakly-connected nodes found - possible documentation gaps or missing edges._