# PROJECT BRIEF — Motor de Fricción Neta para Inversionistas Residentes Fiscales en Colombia

> **Documento de entrada para Claude Code.**
> Léelo completo antes de escribir una sola línea de código.
> No implementes nada en la Fase 0 ni en la Fase 1. Son fases de análisis y especificación.

---

## 0. Instrucciones de proceso (LEER PRIMERO)

Este proyecto se construye en tres fases secuenciales. **No saltes fases.**

### Fase 0 — Análisis de infraestructura con BMAD

1. Verifica si el framework **BMAD-METHOD** está disponible en el entorno.
   - Si no lo está, consulta la documentación oficial vigente antes de instalar. **No asumas los comandos ni la estructura de agentes desde memoria** — el proyecto evoluciona rápido y los nombres de agentes/comandos pueden haber cambiado. Verifica primero, ejecuta después.
   - Si no logras instalarlo o los comandos no coinciden con lo esperado, **detente y repórtamelo**. No improvises un sustituto silenciosamente.
2. Ejecuta el flujo de planeación de BMAD con los agentes de *Analyst*, *Product Manager* y *Architect* sobre este brief, con un objetivo concreto:
   - **Decidir la infraestructura.** Yo no tengo criterio formado sobre esto y no quiero que se decida por defecto.
   - Debe producirse un documento de arquitectura que justifique explícitamente:
     - Local-first vs. cloud (y si cloud: AWS vs. Azure vs. VPS simple).
     - Base de datos (relacional vs. time-series vs. archivos Parquet).
     - Monolito vs. servicios.
     - Cómo se cachean y versionan los datos de mercado.
     - Costo mensual estimado en tres escenarios: 1 usuario (yo), 10 clientes, 100 clientes.
3. **Sesgo explícito que debes aplicar:** este es un MVP para un solo operador. Penaliza la sobre-ingeniería. Si la respuesta correcta es "un contenedor Docker, Postgres y archivos Parquet en disco", dilo. No propongas Kubernetes.

**Entregable Fase 0:** `docs/00-architecture-decision.md` con las decisiones y sus trade-offs.

### Fase 1 — Especificación con spec-kit (SDD)

1. Verifica la disponibilidad de **GitHub Spec Kit** (`specify`). Igual que arriba: **consulta la documentación oficial vigente para los comandos exactos**, no los infieras de memoria.
2. Inicializa el proyecto y ejecuta el ciclo de Spec-Driven Development completo:
   - Definir la **constitución** del proyecto (principios no negociables — usa la sección 2 de este brief como base).
   - **Especificar** el *qué* y el *porqué* a partir de las secciones 3 a 8 de este brief.
   - **Planear** el *cómo*, alimentando el plan con las decisiones de infraestructura de la Fase 0.
   - Descomponer en **tareas** ejecutables.
3. **Antes de implementar, muéstrame la especificación y el plan.** Quiero revisarlos.

**Entregable Fase 1:** especificación, plan técnico y backlog de tareas versionados en el repo.

### Fase 2 — Implementación

Solo después de mi aprobación explícita del plan.

---

## 1. Contexto y objetivo

Soy ingeniero financiero y científico de datos en Medellín, Colombia. Construyo una herramienta de **análisis** (no de asesoría) para producir memorandos de inversión defendibles para clientes que son **residentes fiscales en Colombia**.

**El problema que resuelve:** todo el mundo compara vehículos de inversión por su retorno bruto. Nadie los compara por su **retorno neto real** después de:

- Comisión de gestión del fondo (TER)
- Retención en la fuente en el país de origen
- Comisiones de bróker y custodia
- Spread cambiario de entrada y salida
- Impuesto de renta colombiano al momento de realizar

La diferencia entre bruto y neto puede superar los 200 puntos básicos anuales. Ese es el producto.

**El objetivo del MVP:** dado un conjunto de vehículos candidatos, un monto, un horizonte y un perfil fiscal, calcular y comparar el **retorno neto esperado en COP** y generar un memorando explicando el resultado.

---

## 2. Principios no negociables (base para la constitución del proyecto)

1. **El LLM no calcula nada.** Todos los números salen de código Python determinístico, testeable y auditable. El LLM solo lee resultados ya calculados y redacta narrativa. Cualquier diseño donde un modelo genere cifras es un defecto crítico.
2. **Todo cálculo es trazable.** Cada número del output debe poder descomponerse en sus inputs y en la fórmula aplicada. Si no se puede auditar, no se muestra.
3. **Los parámetros tributarios son configuración, no código.** Tarifas, UVT, umbrales y plazos viven en un archivo de configuración versionado con fecha de vigencia. La normativa colombiana cambia con cada reforma; el código no puede requerir un refactor por eso.
4. **La herramienta no recomienda.** Produce comparativos y escenarios. Nunca emite un "compre X". Esto es una restricción regulatoria, no estética.
5. **Sin dependencias de datos de pago en el MVP.** Debe funcionar con fuentes gratuitas; el diseño debe permitir cambiar el proveedor de datos sin tocar la lógica de negocio.
6. **Cobertura de tests obligatoria en el módulo fiscal y de fricción.** Ningún cálculo financiero se mergea sin test unitario con caso conocido.

---

## 3. Alcance del MVP

### Sí incluye

- Comparación de **10 a 15 vehículos** cargados manualmente en un catálogo (no hay descubrimiento automático de universo).
- Motor de fricción neta completo (sección 5).
- Motor de escenarios normativos (sección 6).
- Generación de memorando en Markdown y exportación a PDF.
- Interfaz para operar los cálculos y ajustar supuestos.

### No incluye (explícitamente fuera de alcance)

- Valoración de empresas (DCF, múltiplos) — **fase posterior**
- Opciones, futuros y derivados — **fase posterior**
- Optimización de portafolio / frontera eficiente — **fase posterior**
- Conexión con brókeres o ejecución de órdenes — **nunca**
- Multi-tenant, autenticación, facturación — **fase posterior**
- Cualquier funcionalidad de recomendación automática

Si durante la especificación aparece la tentación de meter alguno de estos, la respuesta es no. El MVP se define tanto por lo que excluye como por lo que incluye.

---

## 4. Catálogo de vehículos a soportar

El sistema debe modelar estos tipos, cada uno con su perfil de fricción propio:

| Tipo | Domicilio | Fricción característica a modelar |
|---|---|---|
| ETF de renta variable | EE.UU. | Retención 30% sobre dividendos (sin tratado CO–EEUU); exposición a *estate tax* de EE.UU. sobre activos con *situs* estadounidense por encima de US$60.000 |
| ETF UCITS distributivo | Irlanda | Retención reducida a nivel del fondo por tratado Irlanda–EE.UU.; sin *situs* estadounidense |
| ETF UCITS acumulativo (`Acc`) | Irlanda | Igual que el anterior, pero sin distribución → diferimiento del hecho gravable colombiano hasta la venta |
| Acción individual local | Colombia | Régimen de dividendos del art. 242 ET |
| Fondo de Inversión Colectiva (FIC) | Colombia | Comisión de administración; tratamiento de transparencia fiscal |
| CDT / renta fija local | Colombia | Rendimientos financieros; **verificar si el componente inflacionario sigue vigente** (ver sección 6) |
| TES | Colombia | Rendimientos y ganancia en enajenación |
| Mercado Global Colombiano (BVC) | Colombia | Acceso local a activos externos; comisiones más altas y liquidez menor |

Cada vehículo del catálogo se define en un archivo de datos con: ticker, ISIN, domicilio, TER, dividend yield histórico, tipo de distribución, moneda, mercado de negociación, y bróker(s) por los que es accesible desde Colombia.

---

## 5. El motor de fricción neta — especificación funcional

Este es el corazón del sistema. Debe implementarse como un módulo puro, sin dependencias de UI ni de red.

### Inputs

- Monto inicial en COP
- Horizonte en años
- Vehículo (del catálogo)
- Bróker (comisión de compra/venta, custodia anual, spread FX)
- Supuestos de mercado: retorno total esperado anual, dividend yield
- Tipo de cambio inicial y devaluación anual esperada del COP
- Perfil fiscal: tarifa marginal del contribuyente en cédula general
- Escenario normativo (ver sección 6)

### Secuencia de cálculo (debe implementarse en este orden)

1. **Conversión de entrada.** COP → USD aplicando spread cambiario de compra. Registrar el tipo de cambio de entrada: es la base de costo fiscal.
2. **Costo de entrada.** Restar comisión de compra del bróker.
3. **Evolución anual**, año por año (no fórmula cerrada — el detalle anual debe ser inspeccionable):
   - Apreciación de capital = retorno total esperado − dividend yield
   - Dividendo bruto del año
   - Retención en origen sobre el dividendo, según domicilio del vehículo
   - Dividendo neto: reinvertido si el vehículo es acumulativo; si es distributivo, se registra como **ingreso gravable en Colombia en ese año fiscal**
   - Restar TER sobre el valor del portafolio
   - Restar custodia del bróker
4. **Impuesto anual** (solo vehículos distributivos): el dividendo neto recibido se convierte a COP al tipo de cambio del año y se grava según el régimen aplicable. Este es el costo del que nadie habla.
5. **Salida.** Valor final en USD → COP al tipo de cambio final; restar comisión de venta y spread cambiario de salida.
6. **Impuesto de realización.** Base gravable = valor de venta en COP − costo fiscal en COP.

   > **Punto crítico que el modelo debe hacer visible:** la base gravable se calcula **en pesos**. Si el COP se devalúa, se genera ganancia gravable en Colombia aunque el retorno en dólares haya sido cero o negativo. El sistema debe mostrar explícitamente cuánto del impuesto pagado corresponde a devaluación y no a rentabilidad real.

   Clasificación:
   - Si el período de tenencia ≥ umbral de ganancia ocasional → tarifa de ganancia ocasional
   - Si es menor → cédula general a la tarifa marginal del contribuyente

7. **Resultado.** Reportar por separado:
   - Retorno bruto acumulado
   - Retorno neto acumulado en COP
   - **Desglose de fricción**: cuánto se perdió por TER, por retención en origen, por comisiones, por spread FX y por impuesto colombiano — en COP y en puntos básicos anualizados
   - TIR neta anualizada

### Output visual requerido

Un gráfico de cascada (waterfall) que vaya del retorno bruto al retorno neto, mostrando cada capa de fricción como una barra. Este gráfico es el activo comercial del producto: es lo que se le muestra al cliente.

---

## 6. Motor de escenarios normativos

La normativa tributaria colombiana está en movimiento. El sistema debe correr **todo cálculo en tres escenarios simultáneos**, definidos en configuración:

- **Escenario A — Régimen vigente.** El marco actualmente aplicable.
- **Escenario B — Reforma aprobada íntegra.** Refleja el proyecto de reforma radicado el 20 de julio de 2026, cuyas medidas de renta aplicarían desde el año gravable 2027 de ser aprobado.
- **Escenario C — Reforma parcial o hundida.** Un punto intermedio parametrizable.

Variables que cambian entre escenarios y que deben estar parametrizadas:

- Tarifa y **umbral de tenencia** para ganancia ocasional (el proyecto propone elevarlo de 2 a 4 años)
- Tratamiento del **componente inflacionario** de rendimientos financieros (el proyecto propone eliminar su carácter de ingreso no constitutivo de renta)
- Tarifas y rangos de la tabla progresiva de renta
- Tratamiento de dividendos de residentes
- Umbrales y tarifas del impuesto al patrimonio

**Requisito de honestidad del sistema:** ninguna comparación puede presentarse en un solo escenario. Si un vehículo gana en A pero pierde en B, el memorando debe decirlo en la primera página.

> **Nota para el agente:** los parámetros numéricos concretos de cada escenario los cargo yo, validados con mi contador tributarista. Tu trabajo es construir la estructura que los recibe, no investigar ni inventar las tarifas. Deja los valores como `TODO` en el archivo de configuración con comentarios que indiquen qué debe llenarse y con qué fuente normativa.

---

## 7. Módulo de cumplimiento y alertas

Verificaciones automáticas sobre cada escenario simulado. No calculan impuestos, solo levantan banderas:

- ¿El total de activos en el exterior supera el umbral de declaración obligatoria? → alerta de Formulario 160
- ¿La operación requiere registro cambiario ante el Banco de la República?
- ¿La posición en activos con *situs* estadounidense supera US$60.000? → alerta de exposición a *estate tax*
- ¿El horizonte planeado deja la posición por debajo del umbral de ganancia ocasional en alguno de los tres escenarios?
- ¿El patrimonio proyectado cruza el umbral de impuesto al patrimonio en alguno de los escenarios?

Cada alerta debe citar la norma o el concepto aplicable como texto configurable, e incluir la advertencia de que requiere validación profesional.

---

## 8. Capa LLM — usos permitidos y prohibidos

**Permitido:**

1. **Redacción del memorando.** Recibe un JSON con todos los resultados calculados y produce un documento narrativo profesional en español.
2. **Abogado del diablo.** Para cada vehículo analizado, generar explícitamente el argumento **en contra**: qué tendría que pasar para que esta sea una mala decisión, qué supuestos son frágiles, qué riesgo no está en el modelo. Esta sección es obligatoria en todo memorando.
3. **Auditoría de supuestos.** Señalar supuestos internamente inconsistentes o fuera de rangos históricos razonables.

**Prohibido:**

- Generar cualquier cifra que no venga del motor de cálculo
- Emitir recomendaciones de compra o venta
- Estimar retornos futuros
- Interpretar normativa tributaria por su cuenta

Implementación: usar la API de Anthropic con los resultados inyectados como contexto estructurado. El prompt del sistema debe prohibir explícitamente la generación de números y exigir que cite el campo del JSON del que toma cada cifra.

---

## 9. Requisitos legales de la interfaz

Todo output —pantalla, PDF, memorando— debe incluir, de forma visible y no descartable:

- Que el documento es un **análisis informativo**, no una recomendación profesional de inversión ni asesoría en los términos de la regulación del mercado de valores colombiano.
- Que quien lo emite no es un asesor inscrito en el RNPMV ni una entidad vigilada por la Superintendencia Financiera de Colombia.
- Que los cálculos tributarios son estimaciones que requieren validación por un contador o abogado tributarista, y que la normativa puede haber cambiado.
- La fecha de vigencia de los parámetros tributarios usados en el cálculo.

Esto no es opcional ni negociable. Trátalo como un requisito funcional bloqueante.

---

## 10. Criterios de aceptación del MVP

El MVP está listo cuando:

1. Puedo cargar 15 vehículos en el catálogo y compararlos en un solo comando o pantalla.
2. Cada comparación corre automáticamente en los tres escenarios normativos.
3. El gráfico de cascada muestra el desglose de fricción de forma que un cliente no técnico lo entienda en 30 segundos.
4. El sistema muestra explícitamente qué porción del impuesto se debe a devaluación del COP y no a rentabilidad.
5. Genero un memorando en PDF, con disclaimers, en menos de dos minutos.
6. Los módulos fiscal y de fricción tienen cobertura de tests con casos calculados a mano.
7. Cambiar una tarifa tributaria requiere editar un archivo de configuración, no código.

---

## 11. Restricciones operativas

- Un solo desarrollador (yo). Optimiza para mantenibilidad, no para escala.
- Debe correr en mi máquina sin costo recurrente de infraestructura en el MVP.
- Python como lenguaje principal. Justifica cualquier desviación.
- El repo debe quedar limpio, con README que explique cómo correr todo desde cero.

---

## 12. Qué espero de ti como agente

- **Cuestiona este brief.** Si algo aquí está mal planteado, sobredimensionado o es innecesario para un MVP, dilo antes de construirlo.
- **No asumas versiones ni comandos de herramientas externas.** Verifica contra documentación vigente y repórtame las discrepancias.
- **No rellenes con datos inventados.** Si falta un parámetro, deja un `TODO` visible y escalámelo. Un número inventado en una herramienta financiera es peor que un campo vacío.
- **Reporta al final de cada fase y espera aprobación.**
