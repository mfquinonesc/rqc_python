# rqc_python — Requisiciones masivas SIESA (archivos planos)

Herramientas en Python para generar los archivos planos de importación masiva de
**requisiciones internas** que usa SIESA HITECH (ERP8) en Reymon, directamente desde el
libro de Excel `Requisiciones.xlsm`, sin necesidad de abrir Excel ni ejecutar macros/VBA.

Todo el conocimiento de dominio (estructura del archivo, layouts de campo, versiones
`v02`/`v04`, riesgos conocidos) está documentado en detalle en [`CLAUDE.md`](CLAUDE.md).
Este README es la puerta de entrada rápida; para el detalle byte a byte de cada campo,
ir a ese archivo.

## ¿Para qué sirve?

SIESA importa requisiciones internas mediante un archivo de texto de **ancho fijo**
(`REQUISICIONES_MASIVOS-*.txt`): sin comas ni pipes, cada campo ocupa siempre el mismo
número de caracteres y la posición byte a byte es lo único que identifica cada dato.

Este proyecto reimplementa en Python puro la macro `GeneradorXml` del libro
`Requisiciones.xlsm` (la que arma esos archivos planos), para poder generarlos:

- Sin tener Excel abierto ni depender de VBA.
- Leyendo la definición de cada campo directamente de las hojas del libro (nombre, tipo,
  ancho), de modo que si el layout cambia (como pasó de `v02` a `v04`) solo hay que
  agregar la columna nueva en el Excel — el generador se adapta solo.

## Contenido del repositorio

| Archivo | Descripción |
|---|---|
| `generar_plano_desde_excel.py` | Generador alterno: lee `Requisiciones.xlsm` y produce el `.txt` de ancho fijo. Ver detalle abajo. |
| `Copy of Requisiciones.xlsm` | Libro de Excel de ejemplo con la definición de campos (hojas `DocInv`, `MovInv`, etc.) y la macro original de referencia. |
| `CLAUDE.md` | Documentación de dominio completa: estructura del archivo plano, layout de campos por versión, y contexto del generador de producción (`script_mass_requisitions.py`, Django/docflow) que **no** forma parte de este repositorio. |
| `requirements.txt` | Dependencias Python (`openpyxl`). |

## Estructura del archivo plano (resumen)

Cada `REQUISICIONES_MASIVOS-*.txt` concatena 4 tipos de registro en orden:

1. **INICIO** (`0000`) — 1 línea, marca el inicio del lote.
2. **Encabezado** (`0440`) — 1 línea por documento/requisición.
3. **Detalle** (`0441`) — 1 línea por ítem/movimiento.
4. **FIN** (`9999`) — 1 línea, cierra el lote.

Todos los registros comparten un prefijo común de 18 caracteres (consecutivo global de
línea, tipo de registro, subtipo, versión y compañía). El detalle completo de cada campo,
su posición y ancho está en `CLAUDE.md` (§2 a §4).

## `generar_plano_desde_excel.py`

Lee la definición de campos directamente de las hojas del Excel con esta convención:

- **Fila 1** (desde columna B) → nombre del campo.
- **Fila 2** → tipo: `Num` (numérico) o cualquier otro valor (`Alf`, alfanumérico).
- **Fila 3** → ancho del campo.
- **Fila 4 en adelante** → datos (una fila de Excel = un registro del `.txt`).

Reglas de relleno:

- **`Num`** → se recorta y se rellena con **ceros a la izquierda**.
- **`Alf`** → se recorta y se rellena con **espacios a la derecha**.

Expone dos funciones:

- **`generar_txt_de_hoja(ruta_xlsm, nombre_hoja, ruta_txt_salida)`** — genérica: sirve
  para cualquier hoja del libro que siga la convención anterior (`TERCEROS`,
  `PROVEEDORES`, `ITEMS`, `CLIENTE`, `RUTAS`, y el resto de los ~35 tipos de plano que
  soportaba la macro original). Un archivo por hoja, respetando cada celda tal cual
  (incluido el consecutivo `F_NUMERO-REG`).
- **`generar_txt_requisiciones(ruta_xlsm, ruta_txt_salida, hoja_encabezado="DocInv", hoja_detalle="MovInv", compania="001")`**
  — específica de Requisiciones: combina encabezado (`DocInv` → `440`) y detalle
  (`MovInv` → `441`) en un solo archivo con `INICIO`/`FIN`, y **recalcula**
  `F_NUMERO-REG` como consecutivo global (los consecutivos de `DocInv` y `MovInv` en el
  libro de ejemplo no continúan entre sí; copiarlos tal cual generaría numeración
  inválida).

⚠️ **Cuidado con campos de cantidad** (`f441_cant_base`, `f441_cant_2`, etc.): la regla
`Num` solo rellena con ceros lo que ya esté escrito en la celda — **no inserta el punto
decimal por su cuenta**. El valor debe escribirse ya formateado como texto completo, ej.
`"000000000000078.0000"` (15 enteros + punto literal + 4 decimales), o armarse con una
fórmula auxiliar en el Excel:

```
="000000000000" & TEXTO(A1,"000") & ".0000"
```

## Instalación

```bash
python -m venv .venv
.venv\Scripts\activate        # Windows
pip install -r requirements.txt
```

## Uso

### Generar el archivo combinado de Requisiciones

```python
from generar_plano_desde_excel import generar_txt_requisiciones

ruta = generar_txt_requisiciones(
    ruta_xlsm="Copy of Requisiciones.xlsm",
    ruta_txt_salida="REQUISICIONES_MASIVOS.txt",
)
print("Archivo generado en:", ruta)
```

O directamente desde la línea de comandos (usa el mismo Excel de ejemplo del repo):

```bash
python generar_plano_desde_excel.py
```

### Generar el plano de cualquier otra hoja (genérico)

```python
from generar_plano_desde_excel import generar_txt_de_hoja

generar_txt_de_hoja("Copy of Requisiciones.xlsm", "DocInv", "DocInv.TXT")
generar_txt_de_hoja("Copy of Requisiciones.xlsm", "MovInv", "MovInv.TXT")
```

Solo hojas con datos reales desde la fila 4 son exportables; hojas de documentación
(`Instrucciones`, `Encabezado`, `Detalle`) lanzan un error claro si se intentan exportar.

## Verificación rápida al tocar este flujo

- Si SIESA rechaza un archivo generado: medir el largo exacto de cada línea (`len(linea)`)
  y compararlo contra los totales esperados (375/2534 para `v02`, 450/2564 para `v04`) —
  un desfase de longitud es la señal más rápida de un campo desbordado o versiones
  mezcladas.
- Confirmar `F_VERSION-REG` esperado (`02` vs `04` u otra futura) antes de generar — el
  ancho total del archivo depende de eso.
- Todo campo de cantidad debe llevar el punto decimal literal ya escrito en la celda.
- Si se agregan campos nuevos al layout, basta con agregar la columna en el Excel con su
  nombre/tipo/ancho — este generador se adapta solo.

## Más contexto

Para el detalle completo de layouts, versiones (`v02`/`v04`), riesgos conocidos del
generador de producción y el historial de decisiones de diseño, ver [`CLAUDE.md`](CLAUDE.md).
