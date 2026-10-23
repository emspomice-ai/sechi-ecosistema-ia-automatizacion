# Sechi Medical Rigging — Ecosistema de Automatización IA Autónomo

Automatización de punta a punta del proceso real de **Elaboración de Presupuestos** de rigging de equipos médicos de alta complejidad en Transporte Sechi, construida en n8n sobre una base de Airtable.

## Qué resuelve

Desde que un presupuesto entra por un formulario público hasta que el presupuesto final se envía al cliente por email y queda generado como documento, el ecosistema calcula automáticamente kilómetros de traslado, asigna vehículo y acoplado disponibles, resuelve el precio de rigging y flete según la lista de precios que corresponda, suma recargos y adicionales, redacta el cuerpo del email con un modelo de lenguaje (con aprobación humana antes de enviarlo), y genera el documento final del presupuesto sobre la plantilla real de la empresa. Los casos que dependen de criterio humano (necesidad de contratar terceros, visita de acceso, revisión de direcciones o de montos) se pausan explícitamente y retoman solos una vez que una persona confirma la decisión.

## Contenido de este repositorio

- **`workflow/`** — el archivo `.json` exportado del workflow real de n8n (100 nodos).
- **`docs/Documentacion-Tecnica-Ecosistema-Sechi.pdf`** (y `arquitectura.png` / `.dot` fuente) — documentación técnica consolidada del ecosistema: mapa de arquitectura (triggers, routers, llamadas a APIs externas, el único punto donde se usa un modelo de IA, los puntos de decisión humana (HITL), el mecanismo de reintento y las rutas de error técnico), manual operativo de estructuras de datos (esquema de Airtable y esquemas JSON de cada integración), estrategia de optimización de costos de IA y seguridad/privacidad/resiliencia del sistema.
- **`evidencia/`** — capturas reales del Test de Estrés (ver sección abajo).

## Decisión final (2026-10-23)

El documento final del presupuesto queda tal como está: **Google Slides**, sin exportar a PDF. Fase 4 cierra sin cambios adicionales.

## Base de datos (Airtable, modo lectura)

Vistas públicas de solo lectura, una por tabla — **excluye a propósito la tabla `Empleados`** (datos personales de 21 personas, ver `docs/Documentacion-Tecnica-Ecosistema-Sechi.pdf`, sección "Seguridad, Privacidad y Resiliencia"). Confirmado por el alumno que los 11 links cargan la tabla directo, sin pedir login.

- [`presupuestos`](https://airtable.com/app2fFp8wPC2M1WCG/shrDN6BgwdHnyNZhd) — tabla central de cada caso.
- [`Base de Conocimiento — Redacción`](https://airtable.com/app2fFp8wPC2M1WCG/shrQmArXWeDnkbwLz) — criterios de redacción recuperados por RAG.
- [`flota`](https://airtable.com/app2fFp8wPC2M1WCG/shralqzLtfE6kMkAT)
- [`precios_general`](https://airtable.com/app2fFp8wPC2M1WCG/shr1Yr5FbC9hTsMd6)
- [`flete_y_varios_general`](https://airtable.com/app2fFp8wPC2M1WCG/shrqMxObFMUsdwoPK)
- [`precios_tecnoimagen`](https://airtable.com/app2fFp8wPC2M1WCG/shrPS7erkbtmAtLyv)
- [`flete_y_varios_tecnoimagen`](https://airtable.com/app2fFp8wPC2M1WCG/shrsKi2d8RHTKa3W0)
- [`Niveles de dificultad - GE`](https://airtable.com/app2fFp8wPC2M1WCG/shrIf3zQmYwvyY5Il)
- [`precios_ge_dificultad`](https://airtable.com/app2fFp8wPC2M1WCG/shr6TYm9L0YH4iiNt)
- [`Flete_y_terceros_ge`](https://airtable.com/app2fFp8wPC2M1WCG/shrnSuVIHhB2e2Xcj)
- [`Error Log`](https://airtable.com/app2fFp8wPC2M1WCG/shrSM0MdirtWYEDZB) — alimenta el KPI de tasa de errores técnicos del Dashboard.

## Dashboard de Control Ejecutivo

[Vista pública "Dashboard de Control Ejecutivo"](https://airtable.com/app2fFp8wPC2M1WCG/shrGyAzMoLCzZzkfB) — vista única de `presupuestos`, agrupada por `Status` y luego por `Mes de creación`, confirmada por el alumno que carga directo sin login. En una sola vista cubre los 5 KPIs confirmados:

- **Volumen de presupuestos, total y por período** — conteo por grupo de `Mes de creación` dentro de cada `Status`.
- **Tasa de aprobación/rechazo** — se lee directo de los grupos de `Status` = `aprobado` / `rechazado` (quedan vacíos hasta que haya casos reales con decisión final).
- **Escalados a cotización manual** — grupo de `Status` = `Pendiente cotizacion manual` / `Reintentar cotizacion manual`.
- **Tiempo promedio de aprobación** — campo fórmula `Días hasta aprobación` (**redefinido 2026-10-23**: mide el ciclo completo del caso, desde `Presupuesto creado` hasta `fecha de aprobacion`, no solo el tramo de redacción — no existía un campo con la fecha exacta en que un caso entra a `Status = "generando"`, y agregarlo en n8n quedó descartado por tiempo), agregado con **Promedio** en la barra de resumen.
- **Tasa de errores técnicos** — campo `Errores técnicos` (Count de los registros vinculados en `Error Log`), agregado con **Suma** en la barra de resumen, para comparar contra el total de presupuestos.

## Test de Estrés (2026-10-23)

Se corrió el flujo real 6 veces contra el workflow productivo (no simulado), incluyendo al menos un "Camino Infeliz". En el camino se encontraron y corrigieron 3 bugs reales — ver el detalle completo de cada uno en `docs/Documentacion-Tecnica-Ecosistema-Sechi.pdf`, sección "Seguridad, Privacidad y Resiliencia":

1. **Camino Infeliz — dirección no geocodificable** (Trenque Lauquen): el sistema frenó correctamente en `Status = "Revisar direcciones"` y avisó por Slack. Evidencia en `evidencia/caso-direccion-invalida-trenque-lauquen/`.
2. **Necesita Terceros**: un caso real (Mamógrafo Giotto, cliente Tecnoimagen) se pausó en `Status = "Esperando terceros"` y disparó el aviso + formulario de revisión operativa, tal como está diseñado. Evidencia en `evidencia/caso-necesita-terceros/`.
3. **Caso feliz completo**: el mismo caso, una vez resuelta la revisión operativa, recorrió el resto de la cadena (recargo, redacción con IA, aprobación humana, envío real por Gmail) hasta `Status = "publicado"`, con el documento final generado correctamente en Google Slides con los montos reales (Rigging $1.041.954 + Flete $539.091 = Total $1.739.150). Evidencia en `evidencia/caso-feliz-completo-mamografo-giotto/`.
4. Dos intentos adicionales (cliente escrito en minúscula, y un envío sin el campo Modelo) revelaron y permitieron corregir, respectivamente: un error de comparación de mayúsculas/minúsculas en la fórmula de búsqueda de flete de Tecnoimagen, y confirmaron que el comportamiento ante un campo Modelo vacío es el esperado (no un bug).
