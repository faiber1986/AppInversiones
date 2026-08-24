# Specification Quality Checklist: Motor de Fricción Neta

**Purpose**: Validar completitud y calidad de la especificación antes de planear
**Created**: 2026-08-24
**Feature**: [spec.md](../spec.md)

## Content Quality

- [x] Sin detalles de implementación (lenguajes, frameworks, APIs)
- [x] Centrada en valor de usuario y necesidad de negocio
- [x] Escrita para interlocutores no técnicos
- [x] Todas las secciones obligatorias completas

## Requirement Completeness

- [ ] **No quedan marcadores [NEEDS CLARIFICATION]** — quedan 2 (FR-052, FR-053), ambos deliberados
- [x] Los requisitos son testeables y no ambiguos
- [x] Los criterios de éxito son medibles
- [x] Los criterios de éxito son agnósticos de tecnología
- [x] Todos los escenarios de aceptación están definidos
- [x] Los casos borde están identificados
- [x] El alcance está acotado (incluye requisitos negativos FR-049 a FR-051)
- [x] Dependencias y supuestos identificados

## Feature Readiness

- [x] Todo requisito funcional tiene criterio de aceptación claro
- [x] Las historias de usuario cubren los flujos primarios
- [x] La feature cumple los resultados medibles de Success Criteria
- [x] No se filtran detalles de implementación

## Notas de validación

**Iteración 1 — hallazgos corregidos antes de cerrar:**

1. *Criterios de éxito con tecnología dentro.* Los criterios del brief §10 mencionaban "un solo comando o pantalla" y "PDF". Se conservó "comando o pantalla" porque describe el esfuerzo del operador, no la tecnología, y "PDF" porque es el formato del entregable acordado con el cliente, no una decisión de implementación. El resto se reformuló en términos de resultado observable.

2. *Requisitos negativos ausentes.* La primera versión describía el alcance excluido solo en prosa. Se elevaron a FR-049, FR-050 y FR-051, porque el brief §3 dice que el MVP se define tanto por lo que excluye como por lo que incluye, y un exclusión en prosa no es testeable.

3. *Prohibiciones de la capa narrativa redactadas como instrucción.* Se reformularon como requisitos verificables sobre la prosa producida (FR-040, FR-041, FR-043), no sobre lo que se le pide al generador. Un requisito que solo se puede cumplir "pidiéndolo" no es testeable.

**Marcadores [NEEDS CLARIFICATION] retenidos deliberadamente (2 de 3 permitidos):**

- **FR-052 — destino del efectivo del dividendo distribuido.** No se resuelve con un valor por defecto razonable: las tres opciones (reinvertir, acumular en caja, repatriar) producen TIR distintas y **definen la comparación distributivo vs. acumulativo, que es el eje del catálogo**. Los cambios estructurales del operador sugieren reinversión pero no lo afirman. Asumirlo sería decidir el resultado central del producto por omisión.
- **FR-053 — presentación de 135 conjuntos de resultados.** Impacta directamente SC-003 (30 segundos, cliente no técnico). Hay al menos tres organizaciones plausibles con implicaciones muy distintas para el producto.

Ambos van a `/speckit-clarify`. **No bloquean** la planeación técnica: la arquitectura acomoda cualquiera de las respuestas sin cambio estructural (AD-29 para FR-052, AD-24 para FR-053).

**Tercer punto NO marcado como clarificación**, por decisión explícita: los valores numéricos de escenarios y series de reajuste. No es una ambigüedad de especificación sino un insumo que el operador carga validado con su tributarista. Está registrado en Assumptions y cubierto por FR-031 y FR-033, que hacen fallar la corrida mientras falten.
