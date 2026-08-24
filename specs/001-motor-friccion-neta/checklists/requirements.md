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

- [x] **No quedan marcadores [NEEDS CLARIFICATION]** — los 2 resueltos por el operador el 2026-08-24
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

**Iteración 2 — los dos marcadores resueltos por el operador el 2026-08-24:**

- **Destino del efectivo del dividendo → `reinvertir` por defecto, pero parámetro de corrida** (FR-052, FR-053). Los tres modos corren para comparar. El operador añadió una precisión que yo no había planteado y que resultó ser el punto fino del asunto: **el spread cambiario depende del modo** (FR-054). En `reinvertir` aplica en cada reinversión, porque lote nuevo es conversión nueva; en `acumular_caja` no aplica hasta la salida; en `repatriar` aplica cada año. Sin esa distinción los tres modos habrían compartido un tratamiento cambiario incorrecto, y el spread es entre 30 y 100 puntos básicos — la magnitud que el brief §1 declara ser el producto.
- **Presentación → celda de referencia + banda de sensibilidad** (FR-056 a FR-059). Se añadió FR-059 para blindar el requisito de honestidad: designar una celda de referencia es un recurso de presentación, **no un filtro**. Las nueve se siguen calculando y quedan disponibles. Sin ese requisito, la celda de referencia habría sido una puerta trasera para volver a presentar un solo escenario, que es justo lo que el brief §6 prohíbe.

**Tercer punto NO marcado como clarificación**, por decisión explícita: los valores numéricos de escenarios y series de reajuste. No es una ambigüedad de especificación sino un insumo que el operador carga validado con su tributarista. Está registrado en Assumptions y cubierto por FR-031 y FR-033, que hacen fallar la corrida mientras falten.
