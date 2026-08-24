# Contrato — Esquema YAML de configuración tributaria

`config/tributario/` y `config/reajuste/` · `AD-3`, `AD-7`, `AD-31`, `AD-35`, `AD-43`

Validado con `pydantic` en el borde (`adaptadores/config/`). Dentro del núcleo circulan objetos validados, nunca `dict` crudos.

---

## Regla del centinela

Todo valor numérico admite la cadena literal `TODO` en lugar de un número. El brief §6 lo exige.

- Cargar un `TODO` **no falla**: produce `valor = None` y permite inspeccionar de una pasada todo lo que está pendiente.
- **Consumir** un `valor = None` levanta `ParametroTributarioFaltante` y la corrida falla.
- **Nunca existe un valor por defecto numérico.**

## Bloque de procedencia — obligatorio

Todo parámetro lo lleva. Su ausencia levanta `ProcedenciaNoDeclarada`.

```yaml
procedencia:
  fuente: "TODO — referencia normativa pendiente"
  fecha_vigencia: "TODO"
  estado: supuesto_no_verificado    # verificado_profesional | supuesto_no_verificado
```

> Todo parámetro derivado del material actual nace `supuesto_no_verificado`: ese material no está
> firmado por un profesional y contiene una contradicción interna entre dos de sus documentos.

---

## `config/tributario/escenario-{a,b,c}.yaml`

```yaml
escenario: A
vigencia_desde: "TODO"

ganancia_ocasional:
  tarifa: TODO
  umbral_tenencia_anios: TODO
  procedencia: { fuente: "TODO", fecha_vigencia: "TODO", estado: supuesto_no_verificado }

cedula_general:
  tabla_progresiva:
    - { desde_uvt: TODO, hasta_uvt: TODO, tarifa_marginal: TODO }
  procedencia: { fuente: "TODO", fecha_vigencia: "TODO", estado: supuesto_no_verificado }

componente_inflacionario:
  es_ingreso_no_constitutivo: TODO
  procedencia: { fuente: "TODO", fecha_vigencia: "TODO", estado: supuesto_no_verificado }

dividendos_residentes:
  tarifa: TODO
  procedencia: { fuente: "TODO", fecha_vigencia: "TODO", estado: supuesto_no_verificado }

impuesto_patrimonio:
  umbral_uvt: TODO
  tarifa: TODO
  procedencia: { fuente: "TODO", fecha_vigencia: "TODO", estado: supuesto_no_verificado }

retencion_origen:
  EEUU:     { tarifa_dividendos: TODO, procedencia: { fuente: "TODO", fecha_vigencia: "TODO", estado: supuesto_no_verificado } }
  IRLANDA:  { tarifa_dividendos: TODO, procedencia: { fuente: "TODO", fecha_vigencia: "TODO", estado: supuesto_no_verificado } }
  COLOMBIA: { tarifa_dividendos: TODO, procedencia: { fuente: "TODO", fecha_vigencia: "TODO", estado: supuesto_no_verificado } }

uvt:
  - { anio_gravable: TODO, valor: TODO, procedencia: { fuente: "TODO", fecha_vigencia: "TODO", estado: supuesto_no_verificado } }
```

---

## `config/reajuste/art-70.yaml`

```yaml
modo: art_70
ventana: TODO          # anual | acumulada  <- DECISIÓN NORMATIVA, escalada al tributarista
campo_indice: TODO     # qué campo del lote indexa la serie
entradas:
  - { anio_gravable: TODO, porcentaje: TODO, procedencia: { fuente: "TODO", fecha_vigencia: "TODO", estado: supuesto_no_verificado } }
```

## `config/reajuste/art-73.yaml`

```yaml
modo: art_73
ventana: TODO
campo_indice: TODO     # anio_fiscal de reconocimiento | año de la operación
entradas:
  - { anio_adquisicion: TODO, factor: TODO, procedencia: { fuente: "TODO", fecha_vigencia: "TODO", estado: supuesto_no_verificado } }
```

### Invariantes de las series

- Una entrada por año, **sin huecos**. Un año ausente levanta `VigenciaNoCubierta`.
- **Nunca se interpola, extrapola ni hereda del año vecino.**
- `ventana` y `campo_indice` son obligatorios y **jamás se infieren**.

> `ventana` y `campo_indice` no son detalles de formato: son **decisiones normativas**. El art. 70 no
> dice por sí solo si su porcentaje aplica al año en curso o de forma acumulada sobre los años de
> tenencia, y la diferencia mueve el costo fiscal en decenas de millones de COP sobre un caso
> realista. Se dejaron sin resolver deliberadamente — interpretarlas violaría el §8 del brief.

---

## `config/alertas.yaml` — `AD-27`

```yaml
formulario_160:
  texto_norma: "TODO"
  advertencia_validacion: "Requiere validación por contador o abogado tributarista."
  procedencia: { fuente: "TODO", fecha_vigencia: "TODO", estado: supuesto_no_verificado }
```

Una alerta sin texto configurado **falla**; no se degrada a un identificador crudo.

---

## `config/brokers.yaml`

Comisión de compra, comisión de venta, custodia anual y spread cambiario por bróker.
**No lleva procedencia**: son datos comerciales, no juicios normativos.
