# Guion y puntos clave — Video Demo (3 minutos)

Armado para grabar la pantalla con el workflow real corriendo en n8n ("Sechi Medical Rigging - automatizacion de presupuestos"). No reemplaza la grabación — es la guía de qué mostrar y qué decir en cada tramo, para no perder tiempo improvisando frente a la pantalla. Usa únicamente datos y nodos ya documentados y probados — nada se inventa para el video.

**Caso sugerido para la demo:** el caso Tecnoimagen / Mamografo-Giotto ya usado en las pruebas end-to-end documentadas (Email_Cliente `contacto@tecnoimagen.com.ar`, vehículos candidatos `AH 011 QM` / `AF 018 IR`, sin acoplado) — es un caso real ya probado, así que no hay sorpresas en pantalla. Si preferís otro caso real para la demo, armamos el guion con ese en vez de este.

## Minuto 0:00–0:20 — Qué problema resuelve (cámara en el formulario público)

Decir algo como: "Este es el ecosistema de automatización que armé para el proceso real de Elaboración de Presupuestos de Transporte Sechi — rigging de equipos médicos de alta complejidad. Todo el proceso, desde que entra el pedido hasta que el cliente recibe el presupuesto, corre solo, con intervención humana solo en los puntos que realmente la necesitan."

Mostrar: el Form Trigger público abierto en el navegador (con la estética de marca).

## Minuto 0:20–0:50 — Intake y disparo del cálculo

Cargar el caso de prueba en el formulario y enviarlo. Mientras carga, explicar en voz: "Apenas se manda el formulario, se crea automáticamente el registro en Airtable con todos los datos del pedido, y arranca la cadena de cálculo: identificación de la lista de precios del cliente, geocodificación y validación de las direcciones, cálculo de kilómetros con la API de rutas, resolución del precio de rigging según el equipo y el nivel de dificultad, chequeo de disponibilidad de vehículo en el calendario de flota, cálculo de flete, y el total final con los adicionales que correspondan."

Mostrar: el canvas de n8n corriendo esa ejecución en vivo (o una ejecución ya guardada si corrés el caso antes de grabar), siguiendo el camino verde de nodos.

## Minuto 0:50–1:30 — Punto de IA (único en todo el ecosistema) + por qué ahí y no en otro lado

Decir algo como: "De las cerca de 15 tareas de decisión que tiene este ecosistema, una sola usa inteligencia artificial: la redacción del cuerpo del email del presupuesto. El resto — cálculo de kilómetros, asignación de vehículo, precios, moneda — es lógica determinística, porque no hay ninguna ambigüedad de lenguaje natural que resolver ahí. Donde sí hay texto en lenguaje natural para combinar con los datos del caso, uso un modelo Gemini Flash-Lite con RAG: recupera los criterios de redacción reales de la base de conocimiento y los combina con los datos ya calculados del presupuesto, con instrucciones explícitas de no inventar ningún valor que no esté en esos datos."

Mostrar: el nodo `Redactar borrador con IA (Gemini)` abierto, con el prompt visible (sin mostrar API keys).

## Minuto 1:30–2:15 — Los puntos de control humano (HITL)

Decir algo como: "El sistema nunca decide solo las cosas que dependen de criterio humano. Por ejemplo, si un presupuesto necesita contratar terceros o una visita de acceso, el flujo se pausa, avisa por Slack, y recién sigue cuando una persona carga la decisión. Y antes de que cualquier presupuesto salga por email al cliente, un humano tiene que aprobar el borrador que redactó la IA — nunca se manda nada sin esa aprobación."

Mostrar: el aviso real de Slack de una de las excepciones (terceros o visita), y la pantalla de aprobación del borrador (Airtable, campo `aprobado`).

## Minuto 2:15–2:45 — Resiliencia: qué pasa si algo falla

Decir algo como: "El ecosistema también maneja los errores técnicos explícitamente: si falla una llamada a una API externa — por ejemplo el cálculo de ruta o la reserva en el calendario — el error se registra en una tabla de log con el nodo que falló y el mensaje real, y se avisa por un canal de Slack separado para errores técnicos. A propósito, el sistema no reintenta solo un error técnico: lo revisa una persona, porque un reintento automático sobre un error que no es pasajero — como una credencial vencida — solo generaría llamados repetidos inútiles."

Mostrar: la tabla `Error Log` en Airtable (con algún caso de prueba ya cargado), y el aviso de Slack correspondiente.

## Minuto 2:45–3:00 — Cierre

Decir algo como: "Todo esto corre sobre n8n y Airtable, con Gmail, Slack, Google Calendar y Google Drive/Slides como integraciones reales — automatizando de punta a punta el proceso real de presupuestos de Sechi, con los puntos de control humano justo donde el negocio los necesita."

Mostrar: el canvas completo del workflow, alejado (zoom out) para que se vea la escala completa.

## Notas para grabar

- Grabar en una ejecución real (no simulada) para que el "Check de seguridad" del video sea consistente con lo que pide la consigna.
- Si el nodo `¿Presupuesto aprobado?` todavía no tiene aplicado el fix de tipo de dato documentado en `seguridad-y-resiliencia.md`, no hace falta esperar a corregirlo para grabar — funciona igual, el fix es una mejora de robustez, no algo que cambie el comportamiento visible en la demo.
- Si para el momento de grabar ya está resuelto el export a PDF del documento final (ver `plan-entrega-final.md`), vale la pena agregar 10-15 segundos mostrando el PDF generado — mueve el minutaje de arriba, pero es información real y suma a la demo.
