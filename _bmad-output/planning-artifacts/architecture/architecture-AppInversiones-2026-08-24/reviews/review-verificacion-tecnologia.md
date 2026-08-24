# Reviewer Gate — Lente: Verificación de Tecnología

- **Artefacto revisado:** `ARCHITECTURE-SPINE.md` (Motor de Fricción Neta), `status: draft`, `updated: 2026-08-24`
- **Fecha de la revisión:** 2026-08-24
- **Entorno destino declarado:** Windows 11, Python 3.12 fijado en `.python-version`
- **Método:** consulta directa a la API JSON de PyPI (`https://pypi.org/pypi/<paquete>/json`) para versión, `requires_python` y lista de wheels; lectura del código fuente publicado de Playwright y del changelog del SDK `anthropic`; documentación oficial vigente de WeasyPrint y Playwright. Ninguna afirmación de esta revisión proviene de memoria.

## Veredicto

**CONCERNS.** El stack es real, vive, y corre en Windows 11 + Python 3.12 sin compilar nada. Pero la frase *"Verificado en vivo contra la API JSON de PyPI el 2026-08-24. Ninguna versión citada de memoria"* es **demostrablemente falsa** para al menos una fila, el stack está **incompleto** frente a lo que las propias ADs exigen, y `anthropic 1.0.0` es un **major con breaking changes publicado cuatro días antes** de la fecha del spine, adoptado sin mencionarlo.

---

## Tabla de verificación paquete por paquete

Todas las versiones de la columna "versión real hoy" se leyeron de `.info.version` en PyPI el 2026-08-24. La columna "wheel Windows" indica si existe un wheel instalable en Windows para CPython 3.12 (`cp312-…-win_amd64.whl`, o un wheel `py3-none-any` puro).

| Paquete | Versión que afirma el spine | Versión real hoy en PyPI | Soporta py3.12 | Wheel Windows (cp312) | Veredicto |
|---|---|---|---|---|---|
| Python | 3.12 | — | — | — | OK — 3.12 está soportado por todo el stack; pin conservador y válido |
| **uv** | **0.10.5** | **0.12.5** | Sí (`>=3.8`) | Sí (`uv-0.12.5-py3-none-win_amd64.whl`) | **FALLO — desactualizado.** 0.10.5 se publicó el **2026-02-24**, seis meses antes de la fecha del spine |
| pydantic | 2.13.4 | 2.13.4 | Sí (`>=3.9`, clasificador 3.12) | Sí (`py3-none-any`) | OK — coincidencia exacta |
| streamlit | 1.62.0 | 1.62.0 | Sí (`>=3.10`, clasificador 3.12) | Sí (`py3-none-any`) | OK — coincidencia exacta |
| matplotlib | 3.11.1 | 3.11.1 | Sí (`>=3.11`, clasificador 3.12) | Sí (`matplotlib-3.11.1-cp312-cp312-win_amd64.whl`) | OK — coincidencia exacta |
| pyarrow | 25.0.1 | 25.0.1 | Sí (`>=3.10`, clasificador 3.12) | Sí (`pyarrow-25.0.1-cp312-cp312-win_amd64.whl`) | OK — con nota (ver H-4) |
| jinja2 | 3.1.6 | 3.1.6 | Sí (`>=3.7`) | Sí (`py3-none-any`) | OK — coincidencia exacta |
| playwright | 1.62.0 | 1.62.0 | Sí (`>=3.10`, clasificador 3.12) | Sí (`playwright-1.62.0-py3-none-win_amd64.whl`) | OK como paquete — **paso de setup no documentado** (ver M-1) |
| **anthropic** | **1.0.0** | 1.0.0 | Sí (`>=3.10`, clasificador 3.12) | Sí (`py3-none-any`) | **RIESGO — major con breaking changes de hace 4 días** (ver H-3) |
| pytest | 9.1.1 | 9.1.1 | Sí (`>=3.10`, clasificador 3.12) | Sí (`py3-none-any`) | OK — coincidencia exacta |
| ruff | 0.16.4 | 0.16.4 | Sí (`>=3.7`) | Sí (`ruff-0.16.4-py3-none-win_amd64.whl`) | OK — coincidencia exacta |
| Modelo LLM `claude-opus-5` | — | Existe y es vigente | — | — | OK — **el identificador es real** (ver V-3) |
| **(ausente)** parser YAML | **no está en la tabla** | `pyyaml` 6.0.3 | Sí | Sí (`pyyaml-6.0.3-cp312-cp312-win_amd64.whl`) | **FALTA — exigido por AD-3, AD-7 y la convención "Configuración"** (ver H-2) |
| **(ausente)** cliente HTTP | **no está en la tabla** | `httpx` 0.28.1 / `httpx2` 2.12.0 | Sí | Sí (puros) | **FALTA — exigido por AD-11, y nombrado explícitamente por AD-1 como prohibido en `motor/`** (ver H-2) |

**Dependencias transitivas de streamlit 1.62.0 verificadas para Windows + cp312** (ninguna obliga a compilar): `numpy 2.5.2` ✔, `pandas 3.0.5` ✔, `pillow 12.3.0` ✔, `httptools 0.8.0` ✔, `websockets` ✔, `protobuf` (puro) ✔, `altair`/`pydeck` (puros) ✔, `watchdog` ✔. `jiter 0.16.0` (transitiva de `anthropic`) ✔.

---

## Hallazgos

### H-1 — ALTO — La afirmación de verificación del spine es falsa; `uv` está seis meses desactualizado

El encabezado de la sección Stack dice literalmente:

> "Verificado en vivo contra la API JSON de PyPI el 2026-08-24. Ninguna versión citada de memoria."

`uv` está declarado en **0.10.5**. La versión vigente hoy es **0.12.5**. Historial real de PyPI:

```
0.10.5   2026-02-24
0.10.12  2026-03-19   <- última de la línea 0.10.x
0.12.0   2026-07-28
0.12.5   2026-08-14   <- vigente
```

`0.10.5` es de hace **seis meses** y ni siquiera es la última de su propia línea menor. Una verificación en vivo contra PyPI el 2026-08-24 no puede producir ese número. La fila fue citada de memoria.

**Por qué importa más allá de la fila:** el resto de la tabla resultó correcto — pero eso lo verifiqué yo, no el spine. La afirmación de provenance del encabezado es lo que permite a las fases río abajo confiar en la tabla sin re-verificar. Al quedar falsificada, **toda la tabla pierde su garantía de origen**, y el defecto que la sección pretendía prevenir (versiones alucinadas) ya ocurrió una vez dentro de ella.

**Acción:** corregir a `uv 0.12.5` y reescribir la frase del encabezado a algo verificable y con alcance honesto — p. ej. "Versiones consultadas contra la API JSON de PyPI el 2026-08-24; ver `reviews/review-verificacion-tecnologia.md` para el registro de verificación." O eliminar `uv` de la tabla: es un instalador, no una dependencia en tiempo de ejecución, y fijar su patch no aporta nada.

### H-2 — ALTO — El Stack está incompleto: faltan dos paquetes que las propias ADs vuelven obligatorios

La tabla Stack se presenta como el inventario tecnológico del sistema, pero omite dos dependencias que las ADs exigen por texto explícito:

**(a) No hay parser de YAML.** AD-3 fija "Texto YAML versionado en git" como uno de los dos hogares del estado. AD-7 hace que `adaptadores/config/` cargue configuración tributaria fechada desde YAML. La convención "Configuración" dice: *"Solo YAML, cargado y validado con modelos `pydantic` en el borde."*

**Pydantic no parsea YAML.** Pydantic valida estructuras Python ya deserializadas; el YAML → `dict` lo tiene que hacer otra librería. `pydantic-settings` tiene una fuente YAML, pero tampoco está en la tabla, y a su vez depende de `pyyaml`. El sistema no puede leer un solo archivo de `config/tributario/` con lo que la tabla declara.

Candidato verificado: `pyyaml 6.0.3`, wheel `cp312-cp312-win_amd64` disponible ✔.

**(b) No hay cliente HTTP.** AD-11 obliga a `adaptadores/mercado/` a descargar datos y persistirlos como snapshot. AD-1 nombra explícitamente `httpx` y `requests` en su lista de prohibiciones para `motor/` — lo que presupone que existen en el proyecto. No aparece ninguno en la tabla.

Esto importa especialmente aquí por H-3: la elección de cliente HTTP ya no es libre, porque `anthropic 1.0.0` arrastra `httpx2`.

**Acción:** añadir ambas filas con versión verificada, o declarar explícitamente en "Deferred" que se eligen en Fase 1 — pero una tabla Stack que no permite arrancar el proceso no es un sustrato de build.

### H-3 — ALTO — `anthropic 1.0.0` es un major con breaking changes publicado cuatro días antes de la fecha del spine

Fechas reales de PyPI:

```
0.125.0   2026-08-19
1.0.0     2026-08-20   <- versión declarada por el spine
```

El spine tiene fecha 2026-08-24. Está fijando un **`.0` de un major que tenía cuatro días de vida**. El CHANGELOG oficial del SDK marca 1.0.0 como:

```
### ⚠ BREAKING CHANGES
* client: upgrade to httpx2 and some minor breaking changes. See MIGRATION.md for details
```

El `requires_dist` real de `anthropic 1.0.0` confirma el cambio de fondo:

```
httpx2<3,>=2.0.0
```

`httpx2` es una **distribución distinta** de `httpx` en PyPI (`httpx2` está en 2.12.0; `httpx` sigue en 0.28.1). Consecuencias que el spine no reconoce:

1. Si `adaptadores/mercado/` (H-2b) elige `httpx` — el default para casi cualquier proyecto Python nuevo — el entorno cargará **dos stacks HTTP simultáneos**, con dos pools de conexión y dos jerarquías de excepciones que `adaptadores/` tendrá que capturar por separado.
2. Existe un `MIGRATION.md` que el spine no menciona; nada indica que se haya leído.
3. Un `.0` de cuatro días no tiene rodaje. AD-2 ("cero infraestructura") y AD-13 ("un solo proceso") apuestan por lo aburrido y estable; adoptar el major más nuevo de PyPI el día que sale contradice esa postura sin justificarlo.

**Acción:** decidir explícitamente y dejar constancia. Opción conservadora: `anthropic 0.125.0` hasta validar la migración a httpx2. Opción deliberada: mantener `1.0.0` **y** añadir `httpx2` a la tabla Stack como cliente HTTP único de `adaptadores/mercado/`, cerrando de paso H-2b — pero eso hay que escribirlo, no dejarlo implícito.

### H-4 — BAJO — `pyarrow 25.0.1` está a un patch de un conflicto de resolución

Verificado: `streamlit 1.62.0` declara

```
pyarrow!=25.0.0,<26,>=7.0
```

`25.0.1` **sí** satisface la restricción, y tiene wheel `pyarrow-25.0.1-cp312-cp312-win_amd64.whl` ✔. La respuesta a la pregunta del mandato es **sí, pyarrow tiene wheels para Windows + cp312**.

Pero la exclusión puntual `!=25.0.0` indica que streamlit tuvo un incidente concreto con esa versión exacta. Fijar `25.0.1` es correcto y deliberado; conviene que el `pyproject.toml` lo fije como `pyarrow>=25.0.1,<26` en vez de `==25.0.1`, para no quedar atrapado entre la exclusión de streamlit y un pin rígido.

### M-1 — MEDIO — AD-12 depende de un paso de setup que el spine no documenta, y que abre una tercera salida de red

AD-12 elige `playwright` + Chromium headless. La documentación oficial vigente de Playwright Python confirma que **`playwright install` es un paso separado y obligatorio** después del `pip install`: el wheel de PyPI trae el driver Node, **no trae los navegadores**.

Consecuencias que el spine no reconoce en ninguna parte:

1. **El Structural Seed dibuja "Chromium headless (playwright)" dentro de la caja de la máquina del operador, y no dice cómo llega ahí.** El texto bajo el diagrama afirma: *"Las dos únicas flechas que salen de la máquina son punteadas y ambas están tras un puerto (AD-9, AD-11)."* Falsa en tiempo de instalación: hay una tercera salida — la descarga del binario de Chromium desde el CDN de Microsoft (~100-150 MB) hacia `%LOCALAPPDATA%\ms-playwright`.
2. **AD-2 dice "cero infraestructura"** pero el sistema adquiere un binario de navegador completo como dependencia implícita fuera del gestor de paquetes. `uv sync` **no** lo instala. Un operador que clone el repo y corra `uv sync` obtendrá un fallo en tiempo de ejecución al primer memorando, no en tiempo de instalación.
3. **El build del navegador está anclado a la versión de playwright.** Subir `playwright` obliga a re-ejecutar `playwright install chromium`; si no se hace, falla en runtime.

**Acción:** documentar `uv run playwright install chromium` como paso de bootstrap explícito en el spine (o en un `Makefile`/`justfile` de arranque), y corregir la afirmación de "las dos únicas flechas" para acotarla a tiempo de ejecución.

### M-2 — MEDIO — El costo mensual probablemente está subestimado; la etiqueta "cota superior" no se sostiene

El precio está **correcto y verificado**: `claude-opus-5` cuesta **5 USD/MTok de entrada y 25 USD/MTok de salida**. La aritmética del spine (3 llamadas × (15k in + 5k out) por memorando) es consistente con esos precios.

El problema es el supuesto de **5k tokens de salida por llamada**. En `claude-opus-5` el *thinking* adaptativo está **encendido por defecto** (a diferencia de Opus 4.8/4.7, donde omitir el parámetro deja el thinking apagado), y `output_config.effort` **también** tiene `high` por defecto. Los tokens de razonamiento **se facturan como tokens de salida**.

Las tres llamadas que el spine describe — redacción, **abogado del diablo**, **auditoría de supuestos** — son precisamente las cargas donde el thinking adaptativo se extiende más. Es probable que la salida real sea un múltiplo de 5k por llamada, salvo que el adaptador fije `effort` explícitamente a `low`/`medium`.

Por lo tanto la frase *"Es una **cota superior**"* es incorrecta tal como está escrita: 5k out/llamada es más bien un **piso** bajo la configuración por defecto del modelo elegido.

**Acción:** o bien fijar `output_config: {effort: "..."}` como decisión explícita en AD-9 (lo que la haría una invariante arquitectónica real y haría la cota defendible), o bien reformular la tabla de costos como estimación con rango y quitar la palabra "cota superior".

### M-3 — MEDIO/BAJO — La convención de Determinismo choca con la ruta de PDF y nadie lo ha comprobado

La convención "Determinismo" afirma: *"Dos corridas con los mismos inputs producen bytes idénticos."* La frase está redactada con sujeto `el núcleo`, así que técnicamente cubre solo `motor/`. Pero AD-11 declara como objetivo explícito *"que un memorando de hace seis meses no se pueda reproducir"* — es decir, reproducibilidad del **artefacto de salida**, no del núcleo.

Un PDF generado por Chromium (`printToPDF`, backend Skia) normalmente lleva metadatos de documento propios. **No pude confirmar en ninguna fuente autoritativa si la salida de `page.pdf()` es byte-idéntica entre corridas**, y Playwright no expone ningún parámetro para fijar esos metadatos. Lo registro como **supuesto no verificado**, no como defecto probado — que es exactamente el problema: el spine lo afirma sin haberlo medido.

**Acción:** spike de dos minutos — renderizar el mismo HTML dos veces y comparar SHA-256. Según el resultado: (a) acotar la invariante de determinismo explícitamente a `motor/` y al `LibroDeAsientos`, o (b) añadir un paso de normalización de metadatos en `adaptadores/render/`.

### L-1 — BAJO — `stop_reason: "refusal"` es un modo de fallo no cubierto por el guard de AD-9

AD-9 construye un guard determinístico que falla el render si un token numérico de la prosa no aparece en el JSON de origen. Correcto y bien pensado. Pero en `claude-opus-5` existe un modo de fallo distinto: la API puede devolver **HTTP 200 con `stop_reason: "refusal"`** y contenido vacío o degradado. Eso no es un error HTTP (no lo captura un `except APIStatusError`) y no es una discrepancia numérica (no lo captura el guard de AD-9): se cuela como "el modelo no escribió nada".

Es de baja severidad a esta altitud —es detalle de implementación del adaptador, no una invariante— pero merece una línea en AD-9: el adaptador debe verificar `stop_reason` antes de leer `content`.

---

## Verificaciones que el spine PASA (y que ataqué específicamente)

### V-1 — AD-12: la razón para descartar WeasyPrint es CORRECTA y sigue vigente

Verificado contra la documentación oficial vigente de WeasyPrint (`doc.courtbouillon.org/weasyprint/stable/first_steps.html`, que documenta **WeasyPrint 69.0**, la versión actual en PyPI). Las instrucciones de instalación en Windows siguen diciendo, textualmente:

> "When Python is installed, you have to install Pango and its dependencies. The easiest way to install these libraries is to use MSYS2."
> 1. "Install MSYS2 keeping the default options."
> 2. "After installation, in MSYS2's shell, execute `pacman -S mingw-w64-x86_64-pango`."

**El rechazo de AD-12 se sostiene.** WeasyPrint en Windows sigue exigiendo instalar MSYS2 y Pango fuera de pip — inaceptable para el perfil de operador de AD-2.

**Matiz menor:** AD-12 justifica el rechazo diciendo *"cuya instalación en Windows sigue fallando según los issues abiertos de WeasyPrint"*. La evidencia real y mucho más fuerte no son issues abiertos, sino **el procedimiento de instalación oficialmente documentado**. Vale la pena corregir la cita: la conclusión es correcta pero la fuente invocada es la débil.

### V-2 — AD-12: `page.pdf()` es efectivamente solo-Chromium; verificado en el código fuente

Verificado leyendo el código publicado de Playwright, no la documentación (la página de API pública ya no lleva la nota histórica).

- `packages/playwright-core/src/server/dispatchers/pageDispatcher.ts`:
  ```ts
  async pdf(params, progress) {
    if (!this._page.pdf)
      throw new Error('PDF generation is only supported for Headless Chromium');
  ```
- `packages/playwright-core/src/server/page.ts` solo asigna `this.pdf` cuando el delegate lo define (`if (delegate.pdf) this.pdf = delegate.pdf.bind(delegate)`).
- Únicamente el delegate de Chromium (`chromium/crPage.ts` → `CRPDF`) define `pdf`. Firefox y WebKit no.

**Conclusión: sí, es solo Chromium.** Firefox/WebKit lanzan. AD-12 acierta.

**Matiz técnico a favor del spine:** la mitad "Headless" de ese mensaje de error es **texto obsoleto**. La guarda real es la presencia del delegate, no el modo headless, así que Chromium headed también genera PDF. AD-12 funciona en ambos modos; el margen es mayor del que el spine cree.

### V-3 — `claude-opus-5` EXISTE, es vigente, y el SDK declarado lo soporta

Verificado. `claude-opus-5` es un identificador de modelo real y actual: ventana de contexto de **1M tokens**, precio **5 USD/MTok entrada, 25 USD/MTok salida** — exactamente la base de cálculo que usa el spine. El SDK `anthropic 1.0.0` (publicado 2026-08-20) es posterior al modelo y lo soporta sin problema.

Notas de comportamiento que el spine debería conocer al implementar (no son defectos del spine, sí insumo para Fase 1):
- El *prefill* de mensaje assistant está **eliminado** en Opus 5 (devuelve 400). Si `adaptadores/llm/` pensaba forzar el formato de salida con un prefill, hay que usar structured outputs (`output_config.format`).
- `budget_tokens` está **eliminado** (400). Se controla profundidad con `output_config.effort`.
- El thinking adaptativo está encendido por defecto (ver M-2).

### V-4 — Streamlit 1.62.0 corre en Windows 11 + Python 3.12 sin dependencias nativas extra

Verificado. `streamlit 1.62.0` declara `requires_python: >=3.10` con clasificador explícito para 3.12, y se distribuye como wheel puro `py3-none-any`. Recorrí su árbol de dependencias buscando cualquier extensión C sin wheel para Windows/cp312: **no hay ninguna**. `numpy 2.5.2`, `pandas 3.0.5`, `pyarrow 25.0.1`, `pillow 12.3.0`, `httptools 0.8.0` y `jiter 0.16.0` tienen todos `cp312-cp312-win_amd64.whl`; `altair`, `pydeck`, `protobuf` y `websockets` son puros. `uvloop` está correctamente marcado `sys_platform != "win32"`, así que no se intenta instalar.

**Ningún paquete del stack obliga a compilar desde fuente en Windows.** El hallazgo alto que el mandato buscaba en esa dimensión **no existe**.

### V-5 — Riesgo Streamlit + Playwright sync API: investigado, NO se confirma como defecto

Investigué específicamente si AD-14 (UI Streamlit) y AD-12 (playwright) colisionan. Playwright Python lanza un error duro si su API síncrona se usa dentro de un event loop corriendo — verificado en `playwright/sync_api/_context_manager.py`:

```python
if self._loop.is_running():
    raise Error("""It looks like you are using Playwright Sync API inside the asyncio loop.
Please use the Async API instead.""")
```

Existen reportes históricos de este choque exacto con Streamlit (p. ej. `microsoft/playwright-python#1769`, "[BUG] Playwright with Streamlit not working"). **Sin embargo**, al revisar el código actual de `streamlit/runtime/scriptrunner/script_runner.py` en el repositorio de Streamlit, **no hay ninguna referencia a `asyncio`**: el hilo del script no tiene event loop corriendo, así que la API síncrona de Playwright debería crear el suyo sin conflicto.

**Registro esto como verificado-limpio con reserva**, no como hallazgo: la evidencia apunta a que funciona, pero es exactamente el tipo de integración que se debe probar con un spike antes de comprometer AD-12 + AD-14 en la misma llamada.

---

## Resumen ejecutivo de acciones

| # | Sev. | Acción |
|---|---|---|
| H-1 | Alto | Corregir `uv` a 0.12.5 (o quitarlo de la tabla) y reescribir la afirmación de verificación del encabezado del Stack |
| H-2 | Alto | Añadir parser YAML (`pyyaml 6.0.3`) y cliente HTTP al Stack — sin ellos AD-3, AD-7 y AD-11 no son implementables |
| H-3 | Alto | Decidir explícitamente sobre `anthropic 1.0.0` (major de 4 días, migra a `httpx2`); leer su MIGRATION.md o bajar a 0.125.0 |
| M-1 | Medio | Documentar `playwright install chromium` como paso de bootstrap; corregir "las dos únicas flechas que salen de la máquina" |
| M-2 | Medio | Fijar `effort` en AD-9 o reformular la tabla de costos — "cota superior" no se sostiene con thinking adaptativo por defecto |
| M-3 | Medio/Bajo | Spike de reproducibilidad del PDF; acotar la invariante de determinismo según el resultado |
| H-4 | Bajo | Expresar el pin de pyarrow como `>=25.0.1,<26` |
| L-1 | Bajo | AD-9: verificar `stop_reason` antes de leer `content` |
