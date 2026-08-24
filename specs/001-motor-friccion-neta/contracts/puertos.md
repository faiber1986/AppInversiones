# Contrato — Puertos del núcleo

`motor/puertos/` · Protocolos que el núcleo declara y los adaptadores implementan.
La dirección de dependencia apunta hacia adentro (`AD-1`): el núcleo nunca conoce a su implementador.

---

## `RepositorioConfig` — `AD-7`, `AD-35`, `AD-43`

```
obtener_config(escenario, anio_fiscal) -> ConfigTributaria
obtener_serie_reajuste(modo) -> SerieReajuste
```

**Contrato**

- `obtener_config` levanta `VigenciaNoCubierta` si el año no tiene configuración vigente en ese escenario. **Nunca hereda de otro escenario.**
- `obtener_serie_reajuste` devuelve la serie con su `ventana` y `campo_indice` declarados. Si falta cualquiera de los dos, levanta `ParametroTributarioFaltante`.
- Todo parámetro devuelto porta `Procedencia`. Su ausencia levanta `ProcedenciaNoDeclarada`.
- Un parámetro con centinela `TODO` se devuelve con `valor = None`; el fallo ocurre **al consumirlo, no al cargarlo**, para que el operador pueda inspeccionar todo lo que falta de una sola pasada.

---

## `RepositorioCatalogo` — `AD-5`, `AD-6`, `AD-32`, `AD-50`

```
obtener_vehiculo(vehiculo_id) -> Vehiculo
listar_vehiculos() -> Sequence[Vehiculo]
obtener_perfil(perfil_id) -> PerfilCliente
```

**Contrato**

- Todo campo obligatorio ausente hace fallar la carga. No hay valores por defecto.
- `forma_juridica_emisor`, `evento_ingreso_anual` y `elegibilidad_art_73` portan `Procedencia` (`AD-50`): son juicios normativos, no datos de mercado.

---

## `FuenteMercado` — `AD-11`, `AD-16`

```
obtener_trm(desde_anio: int, hasta_anio: int) -> SerieTRM
```

**Contrato**

- El rango va en **años fiscales**, no en fechas: el dominio no conoce calendario (`AD-42`).
- Devuelve la serie más el `snapshot_id` que la produjo. Toda corrida registra ese id en su libro.
- El adaptador persiste cada descarga como Parquet inmutable direccionado por contenido. **Un snapshot nunca se sobrescribe.**
- La red se toca solo para refrescar. **Una corrida se sirve del caché y nunca depende de que el portal esté arriba.**
- Implementación del MVP: `datos.gov.co/resource/32sa-8pi3.json` vía `httpx2`, sin credencial (ver `research.md` R-1).

---

## `RedactorNarrativo` — `AD-9`, `AD-23`

```
redactar(seccion: SeccionMemorando, datos: Mapping) -> str
```

**Contrato**

- Recibe **datos ya calculados**, nunca una petición de cálculo.
- El núcleo **no llama a este puerto**. Lo invoca `adaptadores/render/`, que aplica los guards sobre lo devuelto.
- El adaptador no decide si la prosa se emite. Esa decisión pertenece a los guards.
