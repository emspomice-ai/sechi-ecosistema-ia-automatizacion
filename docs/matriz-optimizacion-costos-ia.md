# Matriz de optimización de costos y recursos de IA

## Principio de diseño: la IA se usa donde agrega valor, no por defecto

Del flujo completo de "Elaboración de Presupuestos" — intake, geocodificación, cálculo de Km, identificación de lista de precios, resolución de vehículo y acoplado, cálculo de rigging/flete/total, chequeo de necesidad de terceros, recargo por fin de semana/feriado, reservas de calendario, generación del documento final y redacción del email — **un solo punto del ecosistema usa un modelo de lenguaje**. Todo lo demás se resuelve con lógica determinística (nodos Code, fórmulas de Airtable, condicionales) o con intervención humana explícita (HITL) cuando el dato depende de criterio humano.

Esta tabla recorre cada tarea del pipeline y documenta qué la procesa y por qué.

| Tarea | Qué la procesa | Por qué no usa IA (o por qué sí) |
|---|---|---|
| Clasificar tipo de trabajo (movimiento interno / rigging estándar) | Campo declarado por el usuario en el formulario | El dato ya es un input directo de la conversación con el cliente; hacer que una IA lo infiera leyendo texto libre sería gastar tokens en algo que no reduce incertidumbre real (ver regla de diseño documentada desde 2026-10-02). |
| Identificar lista de precios aplicable (GE / Tecnoimagen / General) | Nodo Code, comparación exacta de 2 strings | Regla de negocio binaria y estable (solo 2 clientes tienen lista propia) — una IA no mejora la precisión de una comparación de igualdad. |
| Geocodificación y validación de direcciones | OpenRouteService (API determinística) + nodo Code de validación cruzada (altura/calle/localidad) | El riesgo no es de lenguaje natural sino de confiabilidad de datos geográficos — se resuelve con validación de reglas, no con razonamiento de un LLM (que además podría "alucinar" una coincidencia en vez of señalar la incertidumbre). |
| Cálculo de Km, resolución de vehículo/acoplado, precios de rigging/flete/total | Nodos Code + Airtable Search/Update | Aritmética y búsquedas estructuradas sobre datos ya tabulados — determinístico por naturaleza, cero ambigüedad que justifique un modelo probabilístico. |
| Necesidad de contratar terceros / categoría de riesgo del equipo | Gate que siempre pausa y pide confirmación humana (HITL) | Los 3 criterios de decisión (categoría del equipo, dificultad de acceso visible en fotos, información insuficiente → escalar) dependen de juicio humano sobre fotos y contexto operativo real — ni un LLM de visión ni uno de texto tiene el contexto operativo de Sechi para decidir esto con la confiabilidad que exige un presupuesto real. Automatizarlo con IA no sería más barato (igual requeriría revisión humana para confirmar) y sí sería menos confiable. |
| Recargo fin de semana/feriado | Nodo Code + lista de feriados oficiales (fuente: argentina.gob.ar) | Regla de calendario, verificable contra una fuente primaria — no hay nada "difuso" que requiera lenguaje natural. |
| Documento final del presupuesto (Google Slides) | Nodo Code que mapea campos + Google Slides API (Replace Text) | Es un llenado de plantilla campo por campo — una tarea de sustitución de texto exacto, no de generación. Usar un LLM para "redactar" un documento que ya tiene su formato y sus 37 campos fijos sería más caro y más lento que una sustitución directa, sin ganar nada. |
| **Redacción del cuerpo del email de presupuesto** | **Google Gemini (`gemini-flash-lite-latest`), con RAG sobre la base de conocimiento de criterios de redacción** | Esta es la única tarea genuinamente de lenguaje natural del ecosistema: combinar datos estructurados del caso con piezas de conocimiento de estilo/condiciones comerciales de la empresa, en un texto coherente y natural. Es la tarea correcta para reservar el único llamado a un modelo de lenguaje. |

## Por qué Gemini Flash-Lite y no un modelo más grande, para la tarea de redacción

La tarea de redacción es de **combinación de texto, no de razonamiento complejo**: el prompt ya le entrega al modelo (a) los datos del caso ya resueltos (`Idea Semilla`) y (b) los criterios de contenido ya recuperados por RAG desde Airtable, con reglas explícitas de qué incluir y qué no (ver el prompt real del nodo `Redactar borrador con IA (Gemini)`, que instruye textualmente "no inventes un valor" y "no agregues ningún dato... que no esté en los datos del caso o en los criterios"). No hace falta razonamiento multi-paso ni conocimiento de mundo — hace falta seguir instrucciones de forma literal y redactar con fluidez. Para ese perfil de tarea, un modelo "Flash-Lite" (el más liviano de la familia) rinde igual que uno más grande, a una fracción del costo.

**Precios verificados (Google AI, tier estándar de pago, USD por 1.000.000 de tokens, vigentes a octubre de 2026 — fuente: [ai.google.dev/pricing](https://ai.google.dev/pricing)):**

| Modelo | Input / 1M tokens | Output / 1M tokens |
|---|---|---|
| Gemini 3.5 Flash-Lite (familia del modelo usado) | USD 0,30 | USD 2,50 |
| Gemini 3.8 Flash (modelo estándar, no-Lite, de la misma generación) | USD 0,75 | USD 3,75 |

**Estimación de ahorro por presupuesto redactado** (a partir de un prompt real de ~1.500 tokens de entrada — instrucciones + criterios de RAG + Idea Semilla — y ~500 tokens de salida, que es el orden de magnitud real de este prompt; es una estimación de tamaño de prompt, no un dato de negocio del alumno):

- Con Flash-Lite: (1.500 × 0,30 + 500 × 2,50) / 1.000.000 ≈ **USD 0,0017 por presupuesto**.
- Con Flash estándar (misma familia, un escalón más caro): (1.500 × 0,75 + 500 × 3,75) / 1.000.000 ≈ **USD 0,003 por presupuesto**.
- **Ahorro: ~43% por cada presupuesto redactado**, usando el modelo correcto para la complejidad real de la tarea, sin resignar calidad (el prompt ya hace todo el trabajo de razonamiento vía reglas explícitas y RAG, no vía el tamaño del modelo).

**Volumen real de Sechi (ya calculado en la Pre-entrega 4 / `Celula-Hibrida-Proyecto-Real-Sechi-Medical-Rigging.docx`, sección de ROI estimado):** 9,4 solicitudes completas (las que llegan a presupuesto enviado — y por lo tanto pasan por la redacción con IA) por día × 5 días/semana ≈ 47 presupuestos/semana. Extendiendo esa misma cifra a mes y año (≈4,33 semanas/mes, 52 semanas/año — conversión de calendario estándar, no un dato nuevo del negocio):

- **≈ 204 presupuestos/mes**, **≈ 2.444 presupuestos/año**.

**Ahorro mensual/anual de IA, con ese volumen:**

| | Flash-Lite (elegido) | Flash estándar | Ahorro |
|---|---|---|---|
| Por mes (≈204 presupuestos) | ≈ USD 0,35 | ≈ USD 0,61 | ≈ USD 0,26 (~43%) |
| Por año (≈2.444 presupuestos) | ≈ USD 4,16 | ≈ USD 7,34 | ≈ USD 3,18 (~43%) |

El monto absoluto es chico — tiene sentido, porque la redacción de un email es una tarea liviana y el volumen de Sechi es bajo (10-15 solicitudes/día). El punto no es cuánta plata se ahorra eligiendo Flash-Lite en vez de Flash estándar (es marginal), sino que, como se explica abajo, la decisión de diseño que realmente importa para el costo es que 14 de las ~15 tareas del ecosistema no llaman a ningún modelo de IA.

**Aclaración sobre la base del cálculo:** el tamaño del prompt (~1.500 tokens de entrada, ~500 de salida) sigue siendo una estimación de orden de magnitud, no medida contra una ejecución real con conteo de tokens — no cambia la conclusión (la proporción de ahorro es la misma cualquiera sea el tamaño exacto del prompt), pero si se quiere un número exacto en el futuro, se puede leer el conteo de tokens real desde la respuesta de la API de Gemini en una ejecución real.

## El ahorro real y más importante: evitar llamadas a IA, no solo abaratarlas

El punto central de esta estrategia no es "qué modelo de IA es más barato" sino que **de las ~15 tareas de decisión/cálculo del ecosistema, 14 no usan IA en absoluto**. Si el diseño hubiera usado un LLM para cada paso (clasificar tipo de trabajo, decidir lista de precios, validar direcciones, decidir necesidad de terceros, etc. — un patrón común en automatizaciones armadas apurado), el costo por presupuesto sería un múltiplo de lo que es hoy, y además cada paso determinístico ganaría una fuente de error no determinístico (alucinación) donde hoy no existe. Reservar la IA para la única tarea que realmente la necesita es, en sí mismo, la optimización de costos más grande del ecosistema — mayor que la diferencia entre dos modelos de la misma familia.
