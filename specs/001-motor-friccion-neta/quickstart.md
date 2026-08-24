# Quickstart — validación de extremo a extremo

**Fase 1 de `/speckit-plan`** · Escenarios ejecutables que prueban que el sistema funciona.

> Nada de esto corre todavía: no existe código. Este documento define **cómo se validará**, y es el
> contrato contra el que se escribirán las tareas.

---

## Prerrequisitos

```bash
uv sync                        # instala dependencias del lock
uv run playwright install chromium   # ~100-150 MB, una sola vez — uv sync NO lo resuelve
```

> El segundo comando es un paso propio y debe quedar en el README. Es la tercera flecha que sale de
> la máquina, y ocurre solo en el setup (`AD-12`).

---

## V-1 — La compuerta de calidad existe y muerde

**Se valida primero, antes de cualquier cálculo financiero.** El principio VII no puede nacer sin
mecanismo de cumplimiento (`AD-25`).

```bash
uv run ruff check .
uv run pytest tests/test_arquitectura.py -v
uv run pytest --cov=motor.fiscal --cov=motor.friccion
```

**Esperado:** los tres pasan. Y la prueba de que la compuerta *muerde*: introducir
`import httpx2` en cualquier módulo de `motor/` debe hacer fallar `test_arquitectura.py`.

| Verificación | AD |
|---|---|
| Imports prohibidos en `motor/` | `AD-1` |
| `float` en cualquier parte de `motor/` | `AD-45` |
| Literal numérico normativo en `fiscal/` o `cumplimiento/` | `AD-19` |

---

## V-2 — El sistema falla en vez de inventar

**El escenario más importante del MVP.** Con la configuración tributaria en `TODO`, el sistema
**no debe producir ninguna cifra**.

```bash
uv run python -m app.cli comparar --perfil perfiles/ejemplo.yaml --horizonte 10
```

**Esperado:** `ParametroTributarioFaltante`, nombrando qué parámetro y qué archivo.
**Fallo de la prueba:** que devuelva un número.

Variantes que deben fallar igual de explícitamente:

| Caso | Excepción |
|---|---|
| Año simulado sin vigencia en el escenario B | `VigenciaNoCubierta` |
| Parámetro sin bloque `procedencia` | `ProcedenciaNoDeclarada` |
| `art_73` sobre vehículo `sin_clasificar` | `ElegibilidadNoClasificada` |
| Serie de reajuste con un año faltante | `VigenciaNoCubierta` |
| Vehículo sin `retorno_esperado_base` | `ConvencionTerNoDeclarada` |

---

## V-3 — Casos calculados a mano (principio VII)

```bash
uv run pytest tests/fiscal tests/friccion -v
```

Cada caso lleva su aritmética documentada en el propio test. **Un caso derivado de correr el código
no cuenta.** Cobertura mínima:

| Caso | Qué prueba | AD |
|---|---|---|
| Posición de un solo lote, horizonte sobre el umbral | Ruta feliz de ganancia ocasional | `AD-29`, `AD-44` |
| Distributivo que reinvierte, horizonte apenas sobre el umbral | Fragmentación: parte a cédula general | `AD-29`, `AD-37` |
| Retorno cero en USD con devaluación positiva | Impuesto 100 % por devaluación | `AD-28` |
| Tenencia exactamente igual al umbral | Regla del borde | `AD-42` |
| Tres lotes pequeños de la misma cédula | Tarificación sobre el agregado, no por lote | `AD-44` |
| Venta parcial | El remanente hereda reloj y TRM | `AD-40` |
| ETF `neto_de_ter` | El TER **no** se resta dos veces | `AD-6` |
| CDT y TES | Impuesto anual sin dividendo | `AD-5` |
| Mismo dividendo en los tres `DestinoDividendo` | El spread difiere por modo | `AD-51` |

---

## V-4 — Las nueve celdas, ninguna omitida

```bash
uv run python -m app.cli comparar --perfil perfiles/no-activo-fijo.yaml
```

**Esperado:** nueve `ResultadoCelda`. Seis vienen `NoDisponible(PERFIL_NO_ACTIVO_FIJO)` y tres
`Disponible`. **El memorando se emite igual** — este es el caso que antes dejaba a los perfiles no
activo fijo sin ningún entregable (`AD-48`).

| Verificación | AD |
|---|---|
| Ninguna celda se omite ni se sustituye por otro modo | `AD-24`, `AD-32` |
| El ordenamiento ignora las `NoDisponible`, no las trata como cero | `AD-48`, `AD-26` |
| Cada vehículo produce 18 libros, no uno | `AD-41` |

---

## V-5 — Trazabilidad: toda cifra se descompone

```bash
uv run python -m app.cli auditar --corrida <id> --concepto retencion_origen
```

**Esperado:** los asientos que componen el total, cada uno con año, fórmula, inputs y tasa aplicada.
La suma de los asientos iguala el total mostrado (`AD-8`).

**Verificación adicional:** los asientos de `naturaleza: saldo` (valor patrimonial) **no** entran en
ningún total agregado (`AD-46`).

---

## V-6 — Los guards impiden el memorando defectuoso

```bash
uv run pytest tests/render/test_guards.py -v
```

Cada guard con casos positivos **y negativos**:

| Guard | Caso negativo que debe hacer fallar el render |
|---|---|
| G-NUM | Prosa con una cifra que no está en el JSON |
| G-PROC | Prosa que presenta como hecho un parámetro `supuesto_no_verificado` |
| G-CUMP | Prosa con lenguaje recomendatorio |
| G-EST | Memorando sin sección de abogado del diablo |

---

## V-7 — Memorando en PDF, con disclaimers, en menos de dos minutos

```bash
time uv run python -m app.cli memorando --corrida <id> --salida output/memo.pdf
```

**Esperado:** PDF generado en menos de 120 s (SC-006), conteniendo los cuatro elementos legales del
brief §9, la fecha de vigencia y el estado de procedencia de los parámetros usados.

---

## V-8 — Cambiar una tarifa no toca código (SC-008)

1. Correr una comparación y anotar el retorno neto.
2. Editar **un solo valor** en `config/tributario/escenario-a.yaml`.
3. Volver a correr.

**Esperado:** el resultado cambia y **ningún archivo `.py` fue modificado**. Automatizado como test.

---

## V-9 — Reproducibilidad

Dos corridas con los mismos inputs y el mismo `snapshot_id` de TRM producen **bytes idénticos**
(`AD-11`, convención de determinismo). El núcleo no lee reloj, entorno ni aleatoriedad.

---

## Orden de validación

`V-1` va primero: sin compuerta, ninguna de las demás tiene con qué hacerse cumplir.
`V-2` va segundo: probar que el sistema **se niega a inventar** antes de probar que calcula bien.
