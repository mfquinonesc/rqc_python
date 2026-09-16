# Investigación: REQUISICIONES_MASIVOS-10092026 (ANTIGUO vs NUEVO)

> Notas de trabajo para continuar la investigación en el proyecto/código real que genera este archivo.
> Creado: 2026-09-15. Autor de la comparación: Claude (Cowork), a pedido de Francisco (innovacion@reymon.com.co).

## 1. Objetivo

Entender por qué el mismo lote de requisiciones masivas (fecha `20260910`) se generó dos veces con estructuras distintas:

- `REQUISICIONES_MASIVOS-10092026_ANTIGUO.txt`
- `REQUISICIONES_MASIVOS-10092026_NUEVO.txt`

y si el cambio es solo de formato/agrupación o si también cambió el cálculo de cantidades. **Conclusión de esta primera pasada: es lo segundo — cambió el agrupamiento Y las cantidades.** Falta confirmar la causa en el sistema/código fuente real (no se tuvo acceso a él en esta sesión).

## 2. Descripción del formato (ambos archivos)

Archivo plano de ancho fijo, sin delimitadores, usado aparentemente para una carga masiva de requisiciones (posible interfaz de un ERP; no se identificó el sistema exacto todavía). Estructura por tipo de línea:

| Línea(s) | Tipo | Descripción |
|---|---|---|
| 1 | Control/inicio | `000000100000001001` — fija, igual en ambos archivos |
| 2..N-1 (subgrupo A) | Encabezado de requisición (`RQS...`) | Uno por cada número de requisición. Tipo de registro `044`. Trae: nº RQS, fecha, un código de estado/tipo (ver §4), código de bodega/planta `00707510`, referencia de pedido de venta, código de lote de material |
| N..M (subgrupo B) | Detalle / ítem (`RQS...`) | Uno por cada material solicitado. Trae: nº RQS al que pertenece, número de ítem consecutivo **global** (no se reinicia por RQS), código de material, descripción, unidad de medida, cantidad (14 decimales), fecha, bodega |
| Última | Trailer/cierre | `NNNNNNN9999...` — el contador de 7 dígitos coincide con el total de líneas del archivo |

Ambos archivos comparten el mismo "lenguaje" de campos (prefijos `RQS`, contador global de ítem, trailer con conteo), es decir, es el mismo layout/versión de interfaz, no un formato distinto.

## 3. Diferencia estructural principal: cardinalidad del encabezado

- **ANTIGUO**: 1 solo registro de encabezado (línea 2, 375 caracteres) para `RQS00000001`.
  - El campo de pedido de venta viene como texto libre concatenado: `001-PS -12106, 001-PS -12106, 001-PS -12107, 001-PS -12107` (4 menciones, con espacio antes del guion, inconsistente).
  - El campo de lote de material queda truncado: `MP0053054-3052` — se pierde el sufijo `-PLUS` que sí existe en el lote real `MP0053052-PLUS`.
- **NUEVO**: 4 registros de encabezado independientes (líneas 2-5, 450 caracteres cada uno), uno por cada `RQS` (00000001 a 00000004):

  | RQS | Pedido de venta | Lote de material |
  |---|---|---|
  | 00000001 | 001-PS-12107 | MP0053054 |
  | 00000002 | 001-PS-12107 | MP0053054 |
  | 00000003 | 001-PS-12106 | MP0053052-PLUS |
  | 00000004 | 001-PS-12106 | MP0053052-PLUS |

  Cada encabezado tiene su propio campo limpio (sin concatenar, sin truncar). El pedido se movió a un campo dedicado al final del registro (en el antiguo estaba justo después de la bodega).

**Hipótesis a validar en el código real**: el proceso que arma el archivo antiguo probablemente hacía un `GROUP BY`/consolidación indebida de las 4 sub-requisiciones en 1 sola cabecera y concatenaba el campo de referencia como texto (con riesgo de truncamiento por longitud fija de campo). El proceso nuevo genera 1 cabecera por cada combinación (pedido, lote), lo cual es más correcto y no trunca datos.

## 4. Otras diferencias de campos en el encabezado

- Un campo numérico justo después de la fecha cambia de `107` (antiguo) a `100` (nuevo) en todas las líneas de encabezado nuevas. No se determinó su significado (¿código de estado del proceso? ¿versión de layout? ¿tipo de documento?). **Pendiente de confirmar contra el diccionario de datos / copybook del sistema origen.**

## 5. Diferencia estructural en el detalle (ítems)

- Longitud de línea: 2534 caracteres (antiguo) → 2564 caracteres (nuevo), **+30 caracteres**.
- El nuevo trae un bloque adicional al final de cada línea de detalle: `00000000000000000000` (20 ceros) + espacios, que no existe en el antiguo. Aparenta ser un campo nuevo reservado, actualmente sin usar.
- El número de ítem (`item_seq`) sigue siendo consecutivo global en todo el archivo en ambos casos (no se reinicia por RQS): 01→13 en el antiguo, 01→15 en el nuevo.
- Reparto de los ítems entre encabezados en el nuevo:
  - RQS1: 2 ítems (NEGRO y AZUL OSCURO, unidad MTR)
  - RQS2: 6 ítems (CUADROS... y MINIPRINT..., unidad MTR)
  - RQS3: 5 ítems (mismo grupo de materiales que RQS2, otra cantidad)
  - RQS4: 2 ítems (AZUL OSCURO y NEGRO, unidad KLS)
- En el antiguo, los 13 ítems estaban todos bajo el único `RQS00000001`.

## 6. Diferencia de datos: las cantidades NO coinciden entre archivos

Mismos 13 materiales/unidades en ambos archivos, pero las cantidades no son iguales — todas suben en el nuevo, de forma no uniforme:

| Material (código) | Descripción | Unidad | Antiguo | Nuevo | Δ | Δ% |
|---|---|---|---|---|---|---|
| MP010100203900 | AZUL OSCURO | MTR | 227 | 240 | +13 | +5.7% |
| MP010100209100 | NEGRO | MTR | 454 | 460 | +6 | +1.3% |
| MP010101106100 | CUADROS AZ/DENIM 12 | MTR | 318 | 320 | +2 | +0.6% |
| MP010101106300 | CUADROS AZUL OSC 6 | MTR | 78 | 80 | +2 | +2.6% |
| MP010101106500 | CUADROS GRIS/OSC 9 | MTR | 318 | 320 | +2 | +0.6% |
| MP010101106600 | CUADROS NEGRO 1 | MTR | 156 | 160 | +4 | +2.6% |
| MP010101106700 | CUADROS NEGRO 2 | MTR | 318 | 320 | +2 | +0.6% |
| MP010101106900 | CUADROS ROJO 10 | MTR | 318 | 320 | +2 | +0.6% |
| MP010101107000 | CUADROS VERDE 11 | MTR | 78 | 80 | +2 | +2.6% |
| MP010101108900 | MINIPRINT AZ/OSC 8 | MTR | 396 | 400 | +4 | +1.0% |
| MP010101109000 | MINIPRINT NEGRO 4 | MTR | 396 | 400 | +4 | +1.0% |
| MP010119203900 | AZUL OSCURO | KLS | 41 | 60 | +19 | **+46%** |
| MP010119209100 | NEGRO | KLS | 41 | 60 | +19 | **+46%** |

**Total: 3139 (antiguo) → 3220 (nuevo), +81 unidades (+2.6%).**

Puntos llamativos para investigar:
- Los materiales en **KLS** subieron +46%, mucho más que el resto (que subió 0.6%–5.7%). ¿Cambió el factor de consumo, la fórmula/BOM, o el redondeo de unidad para esos dos materiales entre la corrida antigua y la nueva?
- El resto de materiales en MTR sube por montos pequeños y dispares (+2 a +13), no parece un redondeo uniforme a un múltiplo fijo — sugiere recálculo real de necesidad, no solo formato.

## 7. Conclusión de esta fase

1. Ambos archivos corresponden al **mismo evento/lote de negocio**: misma fecha (`20260910`), misma bodega (`00707510`), mismos pedidos de venta (`001-PS-12106`, `001-PS-12107`), mismos materiales y lotes.
2. **No son el mismo dato solo reordenado.** Hay dos cambios simultáneos:
   - Reestructuración: 1 encabezado consolidado (con datos truncados) → 4 encabezados limpios, uno por combinación pedido/lote.
   - Recalculo de cantidades: todas suben, de forma no uniforme (caso KLS +46% resalta).
3. Se agregó un campo reservado (20 ceros) al final de cada línea de detalle en el layout nuevo.

## 8. Próximos pasos (para cuando tenga acceso al proyecto/código real)

- [ ] Ubicar el programa/job/interfaz que genera `REQUISICIONES_MASIVOS-*.txt` (buscar en el repo por: `REQUISICIONES_MASIVOS`, `RQS`, nombre de la interfaz, o el layout/copybook de campos).
- [ ] Confirmar qué representa el campo que cambia de `107` a `100` en el encabezado (buscar catálogo de estados/tipos de documento).
- [ ] Revisar la lógica de agrupación de encabezados: confirmar que el cambio fue una corrección intencional (de 1 registro consolidado con texto truncado a N registros limpios) y no un efecto secundario de otro cambio.
- [ ] Revisar el cálculo de cantidades (MRP / explosión de materiales / factor de desperdicio) para el lote `20260910`, en especial por qué los materiales en KLS subieron 46% mientras el resto subió menos del 6%.
- [ ] Verificar si el campo nuevo de 20 ceros al final del detalle ya tiene un propósito definido en el diseño (campo reservado a futuro, o algo que debía poblarse y quedó en cero por defecto/bug).
- [ ] Si existe una especificación de layout (documento de interfaz, copybook, o mapeo de campos) para esta carga masiva, compararla contra ambos archivos para verificar la posición y tamaño exacto de cada campo (esta investigación usó posiciones inferidas por inspección, no una especificación oficial).

## 9. Cómo continuar esta investigación

Para seguir con esto en el proyecto real necesito acceso a la carpeta/repositorio donde vive el código o los datos de origen (por ejemplo, el job que genera este archivo, la tabla de requisiciones en el sistema, o el documento de especificación del layout). Cuando esa carpeta esté conectada, puedo:

1. Releer este archivo para recuperar el contexto de lo ya encontrado.
2. Buscar el programa generador y el diccionario de campos.
3. Cruzar los hallazgos de la sección 6 y 8 contra la lógica real de cálculo de cantidades.

---
*Archivos originales analizados (adjuntos por el usuario en la conversación):*
- `REQUISICIONES_MASIVOS-10092026_ANTIGUO.txt` (16 líneas, 1 encabezado + 13 detalle)
- `REQUISICIONES_MASIVOS-10092026_NUEVO.txt` (21 líneas, 4 encabezados + 15 detalle)
