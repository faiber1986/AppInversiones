# Implementation Plan: [FEATURE]

**Branch**: `[###-feature-name]` | **Date**: [DATE] | **Spec**: [link]

**Input**: Feature specification from `/specs/[###-feature-name]/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command. See `.specify/templates/plan-template.md` for the execution workflow.

## Summary

[Extract from feature spec: primary requirement + technical approach from research]

## Technical Context

<!--
  ACTION REQUIRED: Replace the content in this section with the technical details
  for the project. The structure here is presented in advisory capacity to guide
  the iteration process.
-->

**Language/Version**: [e.g., Python 3.11, Swift 5.9, Rust 1.75 or NEEDS CLARIFICATION]

**Primary Dependencies**: [e.g., FastAPI, UIKit, LLVM or NEEDS CLARIFICATION]

**Storage**: [if applicable, e.g., PostgreSQL, CoreData, files or N/A]

**Testing**: [e.g., pytest, XCTest, cargo test or NEEDS CLARIFICATION]

**Target Platform**: [e.g., Linux server, iOS 15+, WASM or NEEDS CLARIFICATION]

**Project Type**: [e.g., library/cli/web-service/mobile-app/compiler/desktop-app or NEEDS CLARIFICATION]

**Performance Goals**: [domain-specific, e.g., 1000 req/s, 10k lines/sec, 60 fps or NEEDS CLARIFICATION]

**Constraints**: [domain-specific, e.g., <200ms p95, <100MB memory, offline-capable or NEEDS CLARIFICATION]

**Scale/Scope**: [domain-specific, e.g., 10k users, 1M LOC, 50 screens or NEEDS CLARIFICATION]

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Compuertas derivadas de `.specify/memory/constitution.md` v1.0.0. Cada una se responde
SÍ / NO / N/A con justificación. Un NO en una compuerta NO NEGOCIABLE bloquea el plan.

| # | Compuerta | Principio | ¿Pasa? |
|---|---|---|---|
| G1 | Ninguna cifra del output la genera el LLM; todas vienen del motor determinístico, y hay guard de código que lo verifica | I (NO NEG.) | |
| G2 | Todo total se deriva del libro de asientos; ninguna cifra se calcula por vía paralela | II (NO NEG.) | |
| G3 | Ningún parámetro tributario está en código; ninguno tiene valor por defecto numérico | III (NO NEG.) | |
| G4 | Todo parámetro tributario declara procedencia, y el estado `supuesto_no_verificado` se propaga hasta el artefacto final | IV (NO NEG.) | |
| G5 | Ninguna ruta puede emitir lenguaje recomendatorio; hay guard de código y verificación de sección de abogado del diablo | V (NO NEG.) | |
| G6 | Ninguna dependencia de datos es de pago; el proveedor está tras `FuenteMercado` | VI | |
| G7 | Todo cálculo nuevo en `motor/fiscal/` o `motor/friccion/` trae test con caso calculado a mano y aritmética documentada | VII (NO NEG.) | |
| G8 | Toda ruta de salida pasa por el `Renderer` único que emite disclaimers y fecha de vigencia | VIII | |
| G9 | Todo resultado se emite en la matriz 3 escenarios × 3 modos; las celdas no disponibles llevan su razón y no se sustituyen | IX | |
| G10 | `motor/` no importa E/S, red ni UI; no lee reloj, entorno ni aleatoriedad; `tests/test_arquitectura.py` lo verifica | X | |
| G11 | La solución es la más aburrida que satisface los principios; la complejidad añadida está justificada en la tabla de abajo | Gobernanza | |

**Contrato técnico:** los 51 ADs de
`_bmad-output/planning-artifacts/architecture/architecture-AppInversiones-2026-08-24/ARCHITECTURE-SPINE.md`
implementan estos principios. Un diseño que contradiga un AD exige enmendar el AD primero,
no eludirlo.

## Project Structure

### Documentation (this feature)

```text
specs/[###-feature]/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/           # Phase 1 output (/speckit-plan command)
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)
<!--
  ACTION REQUIRED: Replace the placeholder tree below with the concrete layout
  for this feature. Delete unused options and expand the chosen structure with
  real paths (e.g., apps/admin, packages/something). The delivered plan must
  not include Option labels.
-->

```text
# [REMOVE IF UNUSED] Option 1: Single project (DEFAULT)
src/
├── models/
├── services/
├── cli/
└── lib/

tests/
├── contract/
├── integration/
└── unit/

# [REMOVE IF UNUSED] Option 2: Web application (when "frontend" + "backend" detected)
backend/
├── src/
│   ├── models/
│   ├── services/
│   └── api/
└── tests/

frontend/
├── src/
│   ├── components/
│   ├── pages/
│   └── services/
└── tests/

# [REMOVE IF UNUSED] Option 3: Mobile + API (when "iOS/Android" detected)
api/
└── [same as backend above]

ios/ or android/
└── [platform-specific structure: feature modules, UI flows, platform tests]
```

**Structure Decision**: [Document the selected structure and reference the real
directories captured above]

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |
