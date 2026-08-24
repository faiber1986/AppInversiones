# Contrato — Guards de la capa narrativa

`adaptadores/render/` · `AD-9`, `AD-23`, `AD-35`, `AD-49`

Los guards son **código con test propio**, no instrucciones de prompt. Un principio que solo vive en el
system prompt no es un principio: es una esperanza.

**No hay ruta degradada.** Si un guard falla, no hay memorando.

---

## G-NUM — Guard numérico (`AD-9`)

Extrae todo token numérico de la prosa, lo canoniza, y falla si alguno no existe en el JSON del motor.

### Tabla de normalización — es parte del contrato

| Forma en la prosa | Canoniza a |
|---|---|
| `1.234.567,89` (separadores colombianos) | `1234567.89` |
| `COP 1.234.567` / `$1.234.567` | `1234567` |
| `USD 1,234.56` (separadores anglosajones) | `1234.56` |
| `1,2 millones` / `1,2 M` | `1200000` |
| `3,45 %` | `0.0345` |
| `345 pb` / `345 puntos básicos` | `0.0345` |

### Por qué la tabla es contrato y no detalle de implementación

Sin ella el guard tiene dos modos de fallo, ambos inaceptables:

- **Normaliza de menos** → marca como inventada toda cifra bien formateada, y **nunca se emite ningún memorando**. El sistema queda inutilizable.
- **Normaliza de más** → dos cifras distintas colapsan a la misma y **deja pasar una cifra inventada**, que es exactamente lo que el principio I existe para impedir.

### Tests obligatorios

Casos **positivos** (deben pasar) y **negativos** (deben hacer fallar el render).
Un guard sin casos negativos no está probado: solo demuestra que no estorba.

---

## G-PROC — Guard de procedencia (`AD-35`, `AD-49`)

Consulta el `frozenset[ParametroId]` que cada asiento propaga, y falla si la prosa presenta como
**hecho establecido** un resultado que consumió un parámetro `supuesto_no_verificado`.

Sin ese conjunto propagado el guard no puede distinguir qué afirmación es defendible: fallaría todo
o no fallaría nada.

---

## G-CUMP — Guard de cumplimiento no numérico (`AD-23`)

El guard numérico dejaría pasar intacta la frase *«recomiendo el ETF irlandés»*, porque no contiene
ninguna cifra. Estas verificaciones cubren las otras tres prohibiciones del brief §8:

| Verificación | Falla el render cuando la prosa |
|---|---|
| Lenguaje recomendatorio | Emite un juicio de compra o venta |
| Retorno futuro | Estima un retorno que no está en el JSON |
| Interpretación normativa | Interpreta norma fuera de los textos configurados (`AD-27`) |

---

## G-EST — Guard estructural (`AD-23`, `AD-51`)

| Verificación | Falla cuando |
|---|---|
| Sección de abogado del diablo | Falta en el memorando — el brief §8 la exige en todos |
| Divergencia entre celdas | Un vehículo invierte su orden entre celdas y no aparece en la primera página |
| Modo de dividendo declarado | El memorando no dice qué `DestinoDividendo` se usó |
| Disclaimers y vigencia | Falta cualquiera de los cuatro elementos legales del brief §9 |

---

## Orden de ejecución

```
G-NUM  →  G-PROC  →  G-CUMP  →  G-EST
```

El primero que falla detiene el pipeline y **no se emite artefacto**.
El error nombra **el guard, la frase y el motivo** — nunca un fallo genérico.
