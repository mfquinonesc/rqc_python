# Requisiciones masivas — Archivos planos SIESA

Contexto de dominio para trabajar con el proceso de **requisiciones internas** de Reymon
sobre SIESA HITECH (ERP8): cómo se arman los archivos planos de importación, el script
Python de producción que los genera desde base de datos, y el generador alterno que los
arma directamente desde el Excel `Requisiciones.xlsm`. Todo lo de aquí fue verificado
byte a byte contra archivos reales (no son suposiciones).

## 1. Qué es un "archivo plano" en este contexto

Es un archivo de texto de **ancho fijo** (sin delimitadores como comas o pipes) que SIESA
usa para importar masivamente requisiciones internas. Cada línea es un "registro"; cada
campo dentro de la línea ocupa siempre el mismo número de caracteres, sin importar el
contenido:

- Campos **numéricos (`Num`)**: se recortan y se rellenan con **ceros a la izquierda**.
- Campos **alfanuméricos (`Alf`)**: se recortan y se rellenan con **espacios a la derecha**.

No hay separadores entre campos: la posición (byte inicial/final) es lo único que le dice
al importador de SIESA qué campo está leyendo.

## 2. Estructura del archivo `REQUISICIONES_MASIVOS-*.txt`

El archivo completo = 4 tipos de registro concatenados, en este orden:

1. **INICIO** (tipo `0000`) — 1 línea, marca el inicio del lote.
2. **Encabezado** (tipo `0440`) — 1 línea por cada documento/requisición.
3. **Detalle** (tipo `0441`) — 1 línea por cada ítem/movimiento de cada documento.
4. **FIN** (tipo `9999`) — 1 línea, marca el cierre del lote.

Todos los registros (incluidos INICIO y FIN) comparten el mismo prefijo común de 18
caracteres:

| Campo | Ancho | Notas |
|---|---|---|
| `F_NUMERO-REG` | 7 | **Consecutivo global de línea en todo el archivo** (1, 2, 3… hasta la última línea). No es por sección, es del archivo completo. |
| `F_TIPO-REG` | 4 | `0000`=Inicio, `0440`=Encabezado, `0441`=Detalle, `9999`=Fin |
| `F_SUBTIPO-REG` | 2 | Normalmente `00` |
| `F_VERSION-REG` | 2 | Versión de la estructura del registro (ver §4 — clave para detectar cambios de layout) |
| `F_CIA` | 3 | Compañía (`001`) |

Ejemplo real: la primera línea de todo archivo es `000000100000001001` (registro #1,
tipo Inicio, subtipo 00, versión 01, compañía 001) y la última es
`{N:07d}99990001001` donde `N` = número total de líneas del archivo.

## 3. Layout de Encabezado (440) y Detalle (441) — versión "antigua" (v02)

Documentado en las hojas `Encabezado` y `Detalle` del libro `Requisiciones.xlsm`
("ARCHIVOS PLANOS PARA IMPORTACIÓN DE REQUISICIONES A PARTIR DE LA VERSIÓN 1.13.9.30").
Total encabezado = **375** caracteres, total detalle = **2534** caracteres.

**Encabezado (440) — campos con datos reales:**

| Campo | Inicio | Ancho | Tipo |
|---|---|---|---|
| `f440_id_co` | 20 | 3 | Alf |
| `f440_id_tipo_docto` | 23 | 3 | Alf (`RQS`/`RQI`) |
| `f440_consec_docto` | 26 | 8 | Num |
| `f440_fecha` | 34 | 8 | Alf (AAAAMMDD) |
| `f440_id_solicitante` | 57 | 5 | Alf |
| `f440_fecha_entrega` | 62 | 8 | Alf |
| `f440_id_clase_docto` | 73 | 3 | Num (75=consumo, 76=tránsito) |
| `f440_notas` | 78 | 255 | Alf |
| `f440_id_concepto` | 333 | 3 | Num (607=transferencia) |
| `f440_id_bodega_salida` | 336 | 5 | Alf |
| `f440_id_bodega_entrada` | 341 | 5 | Alf |
| `f440_referencia` | 346 | 20 | Alf |
| `f440_id_ubicacion_ent` | 366 | 10 | Alf |

**Detalle (441) — campos con datos reales:**

| Campo | Inicio | Ancho | Tipo |
|---|---|---|---|
| `f441_consec_docto` | 25 | 8 | Num |
| `f441_nro_registro` | 33 | 10 | Num |
| `f441_referencia_item` | 50 | 50 | Alf |
| `f441_codigo_barras` | 100 | 20 | Alf |
| `f441_id_ext1_detalle` (TALLA) | 120 | 20 | Alf |
| `f441_id_ext2_detalle` (COLOR) | 140 | 20 | Alf |
| `f441_id_bodega` | 160 | 5 | Alf |
| `f441_id_concepto` | 165 | 3 | Num |
| `f441_id_motivo` | 168 | 2 | Alf |
| `f441_id_unidad_medida` | 170 | 4 | Alf |
| `f441_cant_base` | 174 | 20 | Num — **15 enteros + PUNTO literal + 4 decimales**, ej. `000000000000078.0000` |
| `f441_cant_2` | 194 | 20 | Num |
| `f441_fecha_entrega` | 214 | 8 | Alf |
| `f441_notas` | 260 | 255 | Alf |
| `f441_desc_varible` | 515 | 2000 | Alf |
| `f441_id_un_movto` | 2515 | 20 | Alf |

⚠️ Nota: en los datos reales de tela (`MTR`), el campo "TALLA" (posición 120) suele traer
el **nombre del color** (ej. `"NEGRO"`) y el campo "COLOR" (posición 140) queda vacío —
confirmado en producción, pero vale la pena validarlo con el equipo funcional/SIESA si
se toca ese mapeo.

## 4. Versión "nueva" (v04) — qué cambió

Se detecta por `F_VERSION-REG = "04"` en vez de `"02"`. Encabezado pasa de 375 a **450**
caracteres, detalle pasa de 2534 a **2564**. Todo lo anterior a estas posiciones se
mantiene igual; lo nuevo se agrega al final:

**Encabezado (440) v04 — campos agregados después de `f440_id_ubicacion_ent` (que termina en 375):**

| Campo | Inicio | Fin | Ancho | Tipo |
|---|---|---|---|---|
| `idCarge` | 376 | 385 | 10 | Alf (vacío en los datos vistos) |
| `opnum` (número de OP/pedido, ej. `"001-PS-12107"`) | 386 | 435 | 50 | Alf |
| `idProyecto` | 436 | 450 | 15 | Alf (vacío en los datos vistos) |

**Detalle (441) v04 — campos agregados después de `f441_id_un_movto` (que termina en 2534):**

| Campo | Inicio | Fin | Ancho | Tipo |
|---|---|---|---|---|
| `precioUnit` | 2535 | 2554 | 20 | Num (siempre `0` en los datos vistos) |
| `UbicacionSalida` | 2555 | 2564 | 10 | Alf (vacío en los datos vistos) |

También cambió la cardinalidad: en v02, un solo encabezado agrupaba varias OP-PS
(usando `f440_notas` como texto libre con la lista separada por comas, ej.
`"001-PS -12106, 001-PS -12107"`, y `f440_referencia` como `"3054-3052"`). En v04, cada
OP-PS generaba su propio encabezado independiente (ver §6 para la modificación que
consolida esto de vuelta a un solo encabezado).

## 5. `script_mass_requisitions.py` — generador de producción (Django/docflow)

Genera el `.txt` a partir de datos de base de datos (`execute_procedure_sp_find_requisition_detail`),
no del Excel. Flujo: `main()` → `get_requisition_detail()` (arma `listReqDetails` filtrando
las OP-PS que ya no estén en `HistoryMassRequisitions` y que sí tengan detalle) →
`gen_txt_requisition()` (arma el texto de ancho fijo con f-strings y lo escribe a disco).

**Riesgos/bugs identificados en la versión original (antes de nuestros ajustes):**

- **Requiere Python 3.12+**: usa comillas dobles anidadas dentro de un f-string delimitado
  también con comillas dobles (`f"{"RQI" if ... else "RQS"}"`). En Python ≤3.11 esto es
  `SyntaxError` y el módulo ni siquiera importa. Solución: usar comillas simples adentro,
  o precalcular `tipoDocto` en una variable antes del f-string.
- **Crash con `None`**: `dataJsonRequisition.get('notes')` / `.get('idSol')` sin valor por
  defecto — si vienen `null`, `f"{None:255}"` lanza `TypeError`. Usar `.get('notes', '') or ''`.
- **Sin validación de longitud**: a diferencia de la macro VBA original (que sí paraba con
  un mensaje claro si un campo no cabía), este script no trunca ni valida. Un dato más
  largo que su campo (`notes`, `ref`, `opnum`, `TALLA`, etc.) desplaza todos los campos
  siguientes de esa línea sin avisar.
- **Cantidades se truncan a enteros**: `round_quantity()` hace `int(...)`, y el `.0000`
  final está fijo en el texto — los decimales reales nunca se envían salvo el redondeo a
  múltiplos configurado para materiales `101`/`102`.
- **Bodegas/compañía fijas**: `idBodSalida="MP001"`, `idBodEntrada="MP005"`, `Co="001"`
  están *hardcoded*, no vienen del JSON de entrada.
- **Acople frágil header↔detalle**: el `idConsecDocto` que amarra cada línea 441 con su
  440 depende de que `listReqDetails` conserve el mismo orden que `listRequisitions` en
  `get_requisition_detail()`. Funciona hoy, pero no está documentado en el código.

## 6. Modificación: un solo encabezado aunque haya varias OP-PS

Como casi todos los campos de encabezado (`idSol`, `deliveryDte`, `notes`, `material`) ya
son compartidos por todo `dataJsonRequisition` (no varían por OP-PS), el cambio es
puntual: solo `ref` y `opnum` difieren por ítem.

- `idConsecDocto` pasa a ser **fijo en 1** (un solo documento), tanto para el único
  encabezado como para todos los movimientos, sin importar cuántas OP-PS se agrupen.
- Se escribe **un solo** registro 440 (no uno por OP-PS con detalle).
- `ref` y `opnum` se combinan con una función `merge_field()`: junta valores únicos
  (`-` para `ref`, `, ` para `opnum`) y, si el resultado no cabe en el ancho fijo, lo
  recorta con `...` y **vuelca el listado completo en `notas`** (255 caracteres libres) —
  para no repetir el riesgo de desbordamiento silencioso del §5.

Ya probado con 2 OP-PS y 3 líneas de detalle repartidas entre ellas: sale un encabezado
de 450 caracteres y las 3 líneas de detalle apuntando al mismo `consec_docto = 00000001`.

## 7. `generar_plano_desde_excel.py` — generador alterno desde el xlsm

Reimplementación en Python puro (sin VBA/Excel abierto) de la macro `GeneradorXml` del
libro. Lee la definición de campos directamente de las hojas: **fila 1 = nombre, fila 2 =
tipo (`Num`/otro=`Alf`), fila 3 = ancho, fila 4+ = datos** (columna A solo es un marcador
de fin de datos, no se exporta).

Dos funciones, dos niveles de generalidad:

- **`generar_txt_de_hoja(xlsm, nombre_hoja, salida)`** — genérica de verdad: sirve para
  **cualquier** hoja del libro que siga esa convención (TERCEROS, PROVEEDORES, ITEMS,
  CLIENTE, RUTAS, y el resto de los ~35 tipos de plano que soportaba la macro original),
  un archivo por hoja, respetando cada celda tal cual (incluido `F_NUMERO-REG`).
- **`generar_txt_requisiciones(xlsm, salida)`** — específica de Requisiciones: combina
  `DocInv` (encabezado 440) + `MovInv` (detalle 441) en un solo archivo con INICIO/FIN.
  A diferencia de la anterior, **sí recalcula** `F_NUMERO-REG` como consecutivo global
  (porque en el libro de ejemplo los consecutivos de `DocInv` y `MovInv` no continúan
  entre sí — copiarlos tal cual generaría un archivo con numeración de línea inválida).

⚠️ Cuidado con `f441_cant_base` / `f441_cant_2`: la regla `Num` de este generador (igual
que la de la macro VBA) solo rellena con ceros lo que ya esté escrito en la celda — **no
inserta el punto decimal por su cuenta**. Hay que escribir el valor ya formateado como
texto completo, ej. `"000000000000078.0000"`, o armarlo con una fórmula auxiliar en el
Excel.

En este libro concreto, solo `DocInv` y `MovInv` tienen datos reales exportables;
`Instrucciones`, `Encabezado` y `Detalle` son hojas de documentación (sin datos desde la
fila 4), y el generador lanza un error claro si se intenta exportar alguna de esas.

## 8. Checklist rápido al tocar este flujo

- Si SIESA rechaza un archivo generado: primero medir el largo exacto de cada línea
  (`len(linea)`) y compararlo contra los totales de §3/§4 — un desfase de longitud es la
  señal más rápida de que algún campo se desbordó o de que se mezclaron versiones.
- Antes de generar, confirmar `F_VERSION-REG` esperado (`02` vs `04` u otra futura) — el
  ancho total del archivo depende de eso.
- Cualquier campo de cantidad debe llevar el punto decimal literal ya escrito.
- Si se agregan campos nuevos al layout (como pasó de v02 a v04), solo hay que agregar la
  columna en la hoja de Excel con su nombre/tipo/ancho — el generador desde xlsm se adapta
  solo; el script de producción (`script_mass_requisitions.py`) sí requiere tocar el
  f-string a mano.
