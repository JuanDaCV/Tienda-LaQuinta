# Tienda LaQuinta

Sistema de control de despachos de bodega para **Distribuidora La Quinta S.A.S.** (Cali). Cada pedido se valida contra el inventario antes de producir efectos: el sistema consulta las existencias, calcula el saldo que quedaría y decide si el despacho sale, sale con alerta a compras o se rechaza.

Parcial 1 · Seminario de Actualización · Ingeniería de Sistemas · UNIAJC · 2026-2

**Sistema funcionando:** https://juandacv.github.io/Tienda-LaQuinta/

## Qué hace

- **Registrar despacho.** Solicitante, destino, producto, cantidad y motivo se eligen de listas; el producto se lee en vivo del inventario. Antes de enviar, una regla muestra cuánto quedaría y en qué zona cae el saldo. Al registrar, el pedido viaja al webhook de Make y la pantalla muestra la ruta que tomó y lo que hizo el escenario.
- **Inventario.** Existencias de cada producto frente a su mínimo, con el estado (disponible, bajo el mínimo o agotado) y el valor en bodega.
- **Historial de movimientos.** Despachos registrados con filtros, detalle por fila y descarga en CSV. Calcula los dos indicadores del análisis y muestra los incidentes de la ruta de respaldo.

## Cómo decide el escenario

El saldo se calcula dentro de Make: `saldo = existencias − cantidad`. El router enruta por ese valor.

| Condición | Ruta | Qué hace |
|---|---|---|
| El código no existe | Respaldo | Registra el incidente, responde `NO_VALIDO` y avisa a compras por correo |
| `saldo < 0` | Rechazado | Solo responde `RECHAZADO`. No descuenta, no registra, no envía confirmación |
| `0 ≤ saldo < mínimo` | Aprobado con alerta | Actualiza la misma fila del inventario, registra el movimiento, responde y avisa a compras |
| `saldo ≥ mínimo` | Aprobado | Actualiza la misma fila del inventario, registra el movimiento, responde y confirma al solicitante |

El caso 5 (saldo 0 con mínimo 20) cae en **aprobado con alerta**: las existencias cubren el pedido, y rechazarlo dejaría al cliente sin producto teniéndolo en bodega.

## Flujo

```
Interfaz web ──POST──▶ Webhook ─▶ Buscar producto en inventario ─▶ Agrupar resultado ─▶ Calcular saldo ─▶ Router
                                                                                                          ├─ Respaldo
                                                                                                          ├─ Rechazado
                                                                                                          ├─ Aprobado con alerta
                                                                                                          └─ Aprobado
```

El agregador existe por el caso 4: si el código no existe, la búsqueda no devuelve filas y el escenario se detendría en silencio. Con el agregador siempre llega un resultado al router, y el código inexistente se va a la ruta de respaldo.

## Base de datos (Google Sheets: BD_LaQuinta)

| Pestaña | Columnas |
|---|---|
| `inventario` | codigo, producto, unidad, existencias, minimo, costo_unitario |
| `movimientos` | id_movimiento, fecha_hora_solicitud, fecha_hora_respuesta, codigo_producto, producto, unidad, cantidad, saldo_anterior, saldo_resultante, minimo, destino, motivo, solicitante, correo, resultado, costo_unitario, valor_total |
| `incidentes` | fecha_hora, codigo_producto, cantidad, solicitante, correo, destino, detalle |

La tabla de movimientos responde preguntas como: qué productos bajan más seguido del mínimo, quién pide más, hacia qué destinos sale más mercancía, cuánto dinero salió por avería y cuánto tarda el sistema en responder. Los rechazos no se guardan en la hoja porque el caso 3 exige que no dejen rastro; quedan en el historial de ejecuciones de Make.

## Indicadores

| Indicador | Fórmula | Campos |
|---|---|---|
| I1 · Tiempo de respuesta | Promedio de (fecha_hora_respuesta − fecha_hora_solicitud) | `movimientos.fecha_hora_solicitud` (la pone la interfaz) y `movimientos.fecha_hora_respuesta` (la escribe Make) |
| I2 · Despachos bajo el mínimo | Filas con resultado = APROBADO_ALERTA ÷ filas aprobadas × 100 | `movimientos.resultado` (lo asigna la ruta del router) |

## Contrato con Make

La interfaz envía un formulario codificado (sin consulta previa de CORS) con: `solicitante`, `correo`, `codigo_producto`, `cantidad`, `destino`, `motivo`, `fecha_hora_solicitud`.

Cada ruta responde un JSON con el encabezado `Access-Control-Allow-Origin: *`:

```json
{
  "resultado": "APROBADO | APROBADO_ALERTA | RECHAZADO | NO_VALIDO",
  "mensaje": "texto para mostrar",
  "producto": "Harina de trigo x 25 kg",
  "unidad": "bulto",
  "cantidad": 10,
  "saldo_anterior": 48,
  "saldo_resultante": 38,
  "minimo": 15,
  "id_movimiento": "MOV-20261010-190001",
  "valor_total": 920000
}
```

## Casos de prueba

| Caso | Datos | Saldo | Resultado esperado |
|---|---|---|---|
| C1 | INS-0101 × 10 | 48 − 10 = 38 (mínimo 15) | Aprobado |
| C2 | INS-0102 × 5 | 12 − 5 = 7 (mínimo 10) | Aprobado con alerta |
| C3 | INS-0407 × 5 | 3 − 5 = −2 (mínimo 8) | Rechazado, sin escribir nada |
| C4 | INS-9999 × 3 | código inexistente | Respaldo e incidente |
| C5 | INS-0204 × 7 | 7 − 7 = 0 (mínimo 20) | Aprobado con alerta |

Los cinco se ejecutan desde la interfaz, con los botones de casos de prueba.

## Configuración

Al comienzo del script de `index.html`, en `CONFIG`:

- `WEBHOOK_URL`: dirección del webhook de Make.
- `SHEET_ID`: ID de la hoja BD_LaQuinta (compartida como «Cualquier persona con el enlace · Lector»).
- `SOLICITANTES` y `DESTINOS`: las opciones de las listas.

## Archivos

| Archivo | Qué es |
|---|---|
| `index.html` | La interfaz completa (HTML, CSS y JavaScript en un solo archivo) |
| `La_Quinta_v2_blueprint.json` | El escenario de Make para importar |
| `La_Quinta_datos.xlsx` | Estructura y datos iniciales de la hoja |

## Publicación

GitHub Pages desde la rama `main`, carpeta raíz.

## Autores

[Nombres de la pareja]
