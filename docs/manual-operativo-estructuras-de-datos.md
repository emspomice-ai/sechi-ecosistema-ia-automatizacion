# Manual operativo de estructuras de datos

## 1. Base de Airtable — "Sechi — Redacción de Presupuestos"

### 1.1 Tabla central: `presupuestos`

Es el registro de cada caso, desde que entra por el formulario hasta que el presupuesto se envía por email. Campos confirmados (verificados contra el workflow real):

**Intake (cargados por el formulario):** `equipo`, `modelo`, `cantidad_bultos`, `cliente`, `direccion_origen`, `direccion_destino`, `fecha_propuesta`, `fotos_acceso`, `Tipo de trabajo`, `Variante` (Nuevo/Usado), `Nivel de Dificultad`, `Email_Cliente`.

**Cálculo (escritos por el Agente de cálculo):** `Lista de precios aplicada`, `Km calculados`, `Vehiculo asignado` (vínculo a `flota`), `Valor de Rigging` + `Moneda Rigging`, `Valor de Flete` + `Moneda flete`, `Valor Total`, `Necesita Terceros` + `Valor de Terceros` + `Moneda Terceros` + `Concepto de Terceros`, `Valor Dia de Trabajo`, `Valor Desconsolidado`, `Valor Montaje Extendido`, `Camion de apoyo aplicado` + `Valor Camion de apoyo`, `Necesita Visita`, `Nº de Presupuesto` (Autonumber).

**Handoff y redacción:** `Idea Semilla` (texto que resume el caso para el agente de redacción), `borrador generado`, `aprobado` (checkbox).

**Campo Lookup:** `Patente (from flota)` — trae la patente real del vehículo vinculado en `Vehiculo asignado` (un campo de vínculo de Airtable solo expone, vía API, el ID interno del registro vinculado — no el texto visible en la grilla; ver más abajo).

**`Status` (los valores reales, en el orden en que aparecen en el flujo):** `Recibido` → `Datos incompletos` | `Revisar direcciones` | `Pendiente cotizacion manual` | `Sin vehiculo disponible` | `Esperando terceros` | `Esperando visita` | `Revision de moneda` → `Calculo terminado` → `generando` → `en revision` → `aprobado` | `rechazado` → `publicado`. Más 6 valores de reintento: `Reintentar datos`, `Reintentar direcciones`, `Reintentar vehiculo`, `Reintentar moneda`, `Reintentar terceros`, `Reintentar cotizacion manual`.

### 1.2 Tablas de precio y flota

| Tabla | Vinculada a | Particularidad |
|---|---|---|
| `flota` | — | 9 registros (8 vehículos + 1 acoplado), `Patente` como campo primario |
| `precios_general` / `flete_y_varios_general` | `flota` (campo `vehiculo`) | `Equipo` se combina con `Modelo` solo para Resonador |
| `precios_tecnoimagen` / `flete_y_varios_tecnoimagen` | `flota` | `Equipo`+`Modelo` para Resonador y Mamógrafo; `flete_y_varios_tecnoimagen` tiene además la columna `equipos`, emparejada 1 a 1 con el vehículo |
| `precios_ge_dificultad` / `Flete_y_terceros_ge` | `flota` | El precio de rigging está partido por nivel de dificultad en 5 columnas (`Dificultad_0_USD` … `dificultad nivel 5`); `Equipo` nunca se combina con `Modelo` |
| `Niveles de dificultad - GE` | — | Tabla de referencia, 6 niveles (0 a 5) |
| `Empleados` | — | Datos personales de 21 personas reales — **excluida del enlace público de Airtable** |
| `Error Log` | `presupuestos` | `Presupuesto` (link), `Nodo`, `Mensaje de Error`, `Fecha y Hora` (automático), `Resuelto` (checkbox manual) |

Los nombres de los campos `Precio`/`Concepto` (precio/concepto de flete) y `Patente (from vehiculo)` (patentes candidatas en las tablas de precio) están unificados entre las 3 listas — originalmente tenían variantes de mayúsculas/minúsculas y singular/plural que se corrigieron durante la construcción.

## 2. Esquemas JSON de las integraciones

### 2.1 Entrada — Form Trigger (reemplaza al Webhook original)

12 campos reales, confirmados contra el nodo `Solicitud de presupuesto de traslado — Transporte Sechi`:

```
Tipo de trabajo        (dropdown, obligatorio: Movimiento interno / Rigging Estandar)
Equipo                 (dropdown, obligatorio: 24 opciones reales — equipos y combos equipo+modelo)
Modelo                 (texto libre, opcional)
Cantidad de bultos      (número, opcional)
Cliente                (texto, obligatorio)
Dirección de origen     (texto, obligatorio)
Dirección de destino    (texto, obligatorio)
Fecha propuesta de traslado (fecha, obligatorio)
Fotos/video de acceso (link) (texto, opcional)
Estado del equipo      (dropdown, opcional: Nuevo / Usado)
Nivel de dificultad     (texto, opcional — a propósito no es tipo Number, ver nota técnica abajo)
Email del cliente       (email, obligatorio)
```

Un nodo adaptador (`adaptar datos del formulario`) traduce estas 12 claves por label a la forma `{ body: {...} }` que consume el resto de la cadena, sin que el resto del workflow dependa de si el origen es un Form Trigger o un Webhook crudo.

**Nota técnica real:** el campo "Nivel de dificultad" se armó deliberadamente como Text Input, no como Number. Con tipo Number, un campo vacío en un Form Trigger de n8n se castea con `Number('')`, que da `0` — un valor numéricamente válido como nivel de dificultad ("planta baja sin obstáculos"), así que no se puede distinguir "vino vacío" de "cargaron 0" si el tipo es Number. Con Text Input, un campo vacío llega como string vacío real, y el nodo `ordenador de datos` puede aplicarle su default (nivel 1) de forma inequívoca.

### 2.2 Geocodificación — OpenRouteService

**Pedido** (`GET /geocode/search`, uno por cada dirección):
```
?api_key=<key>&text=<dirección limpia>&size=1&boundary.country=AR
```

**Respuesta relevante** (formato GeoJSON — el orden de coordenadas es `[longitud, latitud]`, al revés del orden habitual):
```json
{
  "features": [{
    "properties": { "housenumber": "...", "street": "...", "locality": "...", "region": "..." },
    "geometry": { "coordinates": [lon, lat] }
  }]
}
```

### 2.3 Cálculo de ruta — OpenRouteService Directions

**Pedido** (`POST /v2/directions/driving-hgv`):
```json
{
  "coordinates": [[origen_lon, origen_lat], [destino_lon, destino_lat]],
  "units": "km"
}
```

**Respuesta relevante:** `routes[0].summary.distance` (ya en km, por el parámetro `units`).

### 2.4 Calendario de flota — Google Calendar

Reservas y chequeos de disponibilidad se identifican por **substring de patente en el título del evento** (`summary`), no por un campo estructurado — ej. `[MFK 183] - Cliente General - Resonador`. Los eventos de ocupación son de tipo "evento de todo el día" (`{"date": "YYYY-MM-DD"}`, con `end` exclusivo = un día después del último día real cubierto).

### 2.5 Documento final — Google Slides `Replace Text`

El workflow llama a la operación nativa de n8n para Google Slides (no HTTP Request + `batchUpdate` crudo): recibe `presentationId` + una lista de 36 pares `{ text: "{{PLACEHOLDER}}", replaceText: "<valor real>" }`, uno por cada campo de la plantilla real (`Hoja Presupuesto Nueva`, 37 cuadros editables — 36 automáticos + 1 manual).

## 3. Diagrama de relaciones (texto)

```
flota ◄───────┬──────────── precios_general.vehiculo
              ├──────────── precios_tecnoimagen.vehiculo
              ├──────────── precios_ge_dificultad.vehiculo
              ├──────────── flete_y_varios_general.vehiculo
              ├──────────── flete_y_varios_tecnoimagen.vehiculo
              ├──────────── Flete_y_terceros_ge.vehiculo
              └──────────── presupuestos.Vehiculo asignado  (+ Lookup "Patente (from flota)")

presupuestos ◄──────────── Error Log.Presupuesto

Empleados  (tabla aislada, sin vínculo a flota ni a presupuestos — asignación de cuadrilla manual)
```
