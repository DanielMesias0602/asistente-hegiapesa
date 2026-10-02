# Implementation Plan: HegiaStock

## Overview

Implementación del asistente conversacional HegiaStock para Inversiones Hegiapesa S.A.C.
El plan sigue el orden de dependencias entre componentes: infraestructura y modelos primero,
servicios de capa media después, y API + frontend al final. Las pruebas de propiedad con
Hypothesis acompañan a cada componente para validar corrección durante el desarrollo.

---

## Tasks

- [ ] 1. Configurar estructura del proyecto e infraestructura base
  - Crear la estructura de directorios según el diseño: `backend/`, `backend/api/`, `backend/core/`, `backend/services/`, `backend/models/`, `backend/tests/unit/`, `backend/tests/property/`, `backend/tests/integration/`, `frontend/`, `docs/`
  - Crear `requirements.txt` con dependencias fijadas: `fastapi==0.115.*`, `uvicorn[standard]==0.32.*`, `sqlalchemy==2.0.*`, `pydantic==2.10.*`, `google-generativeai==0.8.*`, `langchain==0.3.*`, `langchain-google-genai==2.*`, `langchain-chroma==0.1.*`, `chromadb==0.6.*`, `hypothesis==6.*`, `pytest==8.*`, `pytest-asyncio==0.24.*`, `pymupdf==1.25.*`, `python-multipart==0.0.*`, `python-dotenv==1.0.*`
  - Crear `.env.example` con todas las variables de entorno requeridas (`GOOGLE_API_KEY`, `MCP_SERVER_COMMAND`, `MCP_SERVER_ARGS`, `MCP_TRANSPORT`, `MCP_SERVER_URL`, `SQLITE_DB_PATH`, `CHROMA_DB_PATH`, `DOCS_PATH`, `TZ`) sin valores reales
  - Crear `backend/config.py` que cargue variables de entorno con `python-dotenv` y las exponga como constantes tipadas
  - Crear archivos `__init__.py` en todos los paquetes Python
  - _Requisitos: R1–R9 (infraestructura transversal)_

- [ ] 2. Implementar modelos de base de datos y schemas Pydantic
  - [ ] 2.1 Implementar configuración de SQLite con SQLAlchemy
    - Crear `backend/models/database.py` con el engine SQLite, la sesión `AsyncSession` y la función `init_db()`
    - Definir los modelos ORM para las tablas `tickets_verificacion` y `audit_log` conforme al DDL del diseño (sección 4.1): columnas, constraints `CHECK`, índices `idx_tickets_estado` e `idx_tickets_sku`
    - Incluir comentario explícito en el modelo: «Sin campos de datos personales (Ley 29733)»
    - _Requisitos: R5.1, R5.6, R7.1_

  - [ ] 2.2 Implementar schemas Pydantic
    - Crear `backend/models/schemas.py` con `ChatRequest` (`mensaje: str`, `min_length=1`, `max_length=2000`; `session_id`), `ChatResponse` (`respuesta`, `tipo_respuesta`, `ticket_id`, `fuentes`, `duracion_ms`), `TicketResponse` (`ticket_id` en formato VER-NNN, `sku`, `estado`, `fecha_creacion`) y los dataclasses de dominio MCP (`ProductStock`, `ProductLocation`, `ProductMatch`)
    - _Requisitos: R1, R2, R5.2_

  - [ ]* 2.3 Escribir pruebas unitarias para modelos y schemas
    - Verificar que `ChatRequest` rechaza mensajes vacíos y mensajes superiores a 2000 caracteres
    - Verificar que `TicketResponse` acepta el formato `VER-001` y rechaza formatos inválidos
    - _Requisitos: R5.1_

- [ ] 3. Implementar Filtro PDP (Ley N.° 29733)
  - [ ] 3.1 Implementar `PDPFilter`
    - Crear `backend/core/pdp_filter.py` con los patrones regex para DNI (8 dígitos), RUC, teléfono, email y tarjeta definidos en el diseño (sección 3.8)
    - Implementar `sanitizar_para_log(texto: str) -> str` que reemplaza coincidencias con `[DATO_PERSONAL]`
    - Implementar `validar_ticket_sin_pdp(ticket: dict) -> bool` que verifica que los campos sean exactamente `{id, numero_correlativo, ticket_id, sku, estado, fecha_creacion}`
    - _Requisitos: R7.1, R7.2, R7.3, R7.4_

  - [ ]* 3.2 Escribir prueba de propiedad P11 — El pipeline no persiste datos personales embebidos
    - **Propiedad 11: El pipeline no persiste datos personales embebidos en consultas**
    - **Valida: Requisitos 7.1, 7.3, 7.4**
    - Para todo mensaje que contenga un dato personal generado (DNI ficticio, teléfono, email), `sanitizar_para_log()` no debe retener el dato original en la cadena resultante
    - Usar `hypothesis` con estrategias que inyecten los patrones PDP en strings arbitrarios
    - Configurar `@settings(max_examples=200)`; tag: `# Feature: hegia-stock, Property 11: El pipeline no persiste datos personales`
    - _Requisitos: R7.1, R7.3, R7.4_

  - [ ]* 3.3 Escribir prueba de propiedad P8 — Los tickets no contienen campos de datos personales
    - **Propiedad 8: Los tickets no contienen campos de datos personales**
    - **Valida: Requisitos 5.7, 7.1, 7.2**
    - Para todo dict de ticket, `validar_ticket_sin_pdp()` debe retornar `True` si y solo si el conjunto de claves es exactamente el permitido y ningún valor coincide con los patrones PDP
    - Tag: `# Feature: hegia-stock, Property 8: Los tickets no contienen campos de datos personales`
    - _Requisitos: R5.7, R7.1, R7.2_

- [ ] 4. Implementar Cliente MCP con Read-Only Guard
  - [ ] 4.1 Implementar `MCPClient`
    - Crear `backend/services/mcp_client.py` con la `frozenset` `READ_ONLY_TOOLS` que incluye exactamente las herramientas definidas en el diseño (sección 3.4): `get_product_by_sku`, `search_products_by_name`, `get_product_location`, `list_products_by_category`, `get_categories`
    - Implementar `call_tool(tool_name, arguments)` que lanza `ReadOnlyViolationError` si el nombre no está en la lista blanca, antes de cualquier llamada de red
    - Implementar métodos de alto nivel: `get_stock_by_sku(sku)`, `search_by_name(nombre)`, `get_location(sku)`, `filter_by_category(categoria, limit=10)`, cada uno con `asyncio.wait_for` con timeout de 5 segundos
    - Implementar la conexión al Servidor_MCP via `stdio` o `sse` según `MCP_TRANSPORT` leído desde `config.py`
    - _Requisitos: R1.1, R1.2, R1.5, R2.1, R3.1, R8.1, R8.2, R8.4, R9.3_

  - [ ]* 4.2 Escribir prueba de propiedad P3 — Invocaciones MCP son siempre de solo lectura
    - **Propiedad 3: Todas las invocaciones al Servidor_MCP son de solo lectura**
    - **Valida: Requisitos 1.6, 8.2, 8.4**
    - Para todo nombre de herramienta que no esté en `READ_ONLY_TOOLS`, `call_tool()` debe lanzar `ReadOnlyViolationError` sin realizar ninguna comunicación de red
    - Usar `st.text()` y `st.sampled_from(lista_no_permitida)` para generar nombres de herramientas inválidas
    - Tag: `# Feature: hegia-stock, Property 3: Invocaciones MCP son siempre de solo lectura`
    - _Requisitos: R1.6, R8.2, R8.4_

  - [ ]* 4.3 Escribir prueba de propiedad P9 — Validador MCP bloquea todas las operaciones de escritura
    - **Propiedad 9: El validador MCP bloquea todas las operaciones de escritura**
    - **Valida: Requisitos 8.1, 8.4**
    - Para todo nombre de herramienta que no esté en la lista blanca, `call_tool()` lanza `ReadOnlyViolationError` y el mock de red no registra ninguna llamada
    - Tag: `# Feature: hegia-stock, Property 9: El validador MCP bloquea todas las operaciones de escritura`
    - _Requisitos: R8.1, R8.4_

  - [ ]* 4.4 Escribir pruebas unitarias para MCPClient
    - Caso: Servidor MCP caído → mock lanza `ConnectionError` → método retorna `None` o propaga excepción controlada
    - Caso: Timeout MCP → mock duerme 6 segundos → `asyncio.wait_for` cancela a los 5s
    - Caso: Búsqueda con múltiples coincidencias → mock retorna lista de 3 productos
    - _Requisitos: R1.5, R9.3_

- [ ] 5. Checkpoint — Verificar infraestructura base
  - Asegurarse de que todas las pruebas de los módulos 2, 3 y 4 pasan sin errores. Consultar al usuario si hay decisiones pendientes sobre la conexión al Servidor_MCP existente.

- [ ] 6. Implementar Gemini Client
  - [ ] 6.1 Implementar `GeminiClient`
    - Crear `backend/services/gemini_client.py` que lea `GOOGLE_API_KEY` desde variable de entorno (nunca la loguea ni imprime)
    - Implementar `generate(prompt, json_mode=False, timeout_s=10.0)` con `response_mime_type="application/json"` cuando `json_mode=True`
    - Implementar reintentos con backoff exponencial: 2 reintentos, esperas 1s y 2s, solo para errores 5xx y timeout; sin reintentos en 4xx
    - Implementar circuit breaker de sesión: tras 3 fallos consecutivos, las llamadas de síntesis retornan los datos crudos formateados sin LLM
    - _Requisitos: R1.1, R1.2, R4.1, R6.1_

  - [ ]* 6.2 Escribir pruebas unitarias para GeminiClient
    - Caso: API key ausente → `KeyError` al instanciar
    - Caso: Error 5xx → reintenta 2 veces con backoff y luego lanza excepción
    - Caso: Error 4xx → no reintenta, lanza excepción de inmediato
    - Caso: Circuit breaker → tras 3 fallos retorna datos crudos sin llamar a la API
    - _Requisitos: R4.1, R6.1_

- [ ] 7. Implementar Motor RAG
  - [ ] 7.1 Implementar indexación de documentos
    - Crear `backend/services/rag_engine.py` con el flujo de indexación: `PyMuPDFLoader` para PDF, `UnstructuredMarkdownLoader` para Markdown, `RecursiveCharacterTextSplitter(chunk_size=800, overlap=100)`, embeddings con `GoogleGenerativeAIEmbeddings(model="text-embedding-004")` y persistencia en ChromaDB colección `"hegia_normas"`
    - Implementar `index_document(file_path, doc_name) -> IndexResult` con metadatos: `fuente`, `tipo`, `pagina`, `fecha_indexacion`
    - _Requisitos: R4.2, R6.2_

  - [ ] 7.2 Implementar consulta RAG
    - Implementar `retrieve(query, top_k=3) -> RAGResult` con `asyncio.wait_for` timeout de 4 segundos
    - El resultado debe incluir `respuesta` (sintetizada por Gemini), `chunks` con texto y metadato de fuente, y `fuentes` (lista de nombres de documentos únicos)
    - Si ChromaDB retorna lista vacía, retornar `RAGResult` con `respuesta=None` y `chunks=[]`
    - _Requisitos: R4.1, R4.4, R6.1, R6.3, R6.4, R9.2, R9.4_

  - [ ]* 7.3 Escribir prueba de propiedad P10 — RAG retorna como máximo 3 cláusulas
    - **Propiedad 10: El número de cláusulas RAG retornadas nunca supera 3**
    - **Valida: Requisitos 6.1, 6.3**
    - Para toda consulta de texto arbitrario con ChromaDB mockeado que retorne entre 0 y N chunks, `retrieve()` debe retornar a lo sumo 3 fragmentos y cada uno debe incluir el campo `fuente`
    - Usar `st.text()` para queries y `st.integers(min_value=0, max_value=20)` para el número de chunks del mock
    - Tag: `# Feature: hegia-stock, Property 10: RAG retorna como máximo 3 cláusulas`
    - _Requisitos: R6.1, R6.3_

  - [ ]* 7.4 Escribir pruebas unitarias para RAGEngine
    - Caso: ChromaDB retorna lista vacía → `retrieve()` retorna `RAGResult` con chunks vacíos
    - Caso: Timeout RAG (mock duerme 5s) → cancelado a los 4s, excepción controlada
    - Caso: Carga de documento PDF → indexación genera al menos un chunk con metadato `fuente`
    - _Requisitos: R4.4, R6.4, R9.4_

- [ ] 8. Implementar Gestor de Tickets
  - [ ] 8.1 Implementar `TicketService`
    - Crear `backend/services/ticket_service.py` con `crear_ticket(sku) -> Ticket` que usa transacción `BEGIN IMMEDIATE` para obtener `MAX(numero_correlativo) + 1` y garantizar unicidad del VER-NNN sin race conditions
    - El `ticket_id` se genera como `f"VER-{numero_correlativo:03d}"` (mínimo 3 dígitos, sin reinicio por día)
    - La `fecha_creacion` se genera en `America/Lima` (UTC-5) en formato ISO 8601
    - Implementar `obtener_ticket(ticket_id)` y `listar_tickets(estado=None, limit=50)`
    - _Requisitos: R5.1, R5.2, R5.4, R5.6, R5.7_

  - [ ]* 8.2 Escribir prueba de propiedad P6 — Tickets cumplen invariantes de formato y estado
    - **Propiedad 6: Los tickets creados cumplen invariantes de formato y estado**
    - **Valida: Requisitos 5.1, 5.2**
    - Para todo SKU válido arbitrario, el ticket creado debe tener `ticket_id` que coincida con `^VER-\d{3,}$`, `estado == "Pendiente"` y `fecha_creacion` parseable como datetime en zona `America/Lima`
    - Usar `st.text(min_size=1, max_size=50, alphabet=st.characters(whitelist_categories=("Lu","Ll","Nd")))` para SKUs
    - Tag: `# Feature: hegia-stock, Property 6: Tickets cumplen invariantes de formato y estado`
    - _Requisitos: R5.1, R5.2_

  - [ ]* 8.3 Escribir prueba de propiedad P7 — IDs de tickets son únicos y monótonamente crecientes
    - **Propiedad 7: Los identificadores de tickets son únicos y monótonamente crecientes**
    - **Valida: Requisitos 5.6**
    - Para todo N ≥ 2 tickets creados en secuencia, los valores numéricos en sus `ticket_id` deben ser todos distintos y estrictamente crecientes
    - Usar `st.integers(min_value=2, max_value=50)` para N; crear cada ticket con un SKU ficticio válido
    - Tag: `# Feature: hegia-stock, Property 7: IDs de tickets son únicos y monótonamente crecientes`
    - _Requisitos: R5.6_

  - [ ]* 8.4 Escribir pruebas unitarias para TicketService
    - Caso: Creación exitosa → retorna ticket con formato VER-001 y estado Pendiente
    - Caso: Fallo SQLite (mock lanza excepción) → excepción propagada correctamente
    - Caso: Creación concurrente de N tickets → sin `ticket_id` duplicados (usando `asyncio.gather`)
    - _Requisitos: R5.2, R5.4, R5.6_

- [ ] 9. Checkpoint — Verificar servicios de capa media
  - Asegurarse de que todas las pruebas de los módulos 6, 7 y 8 pasan sin errores. Consultar al usuario si hay ajustes en la lógica de tickets o RAG.

- [ ] 10. Implementar Clasificador de Intención
  - [ ] 10.1 Implementar `IntentClassifier`
    - Crear `backend/core/intent_classifier.py` con el prompt estructurado para `gemini-2.0-flash` que retorna JSON con `tipo` (uno de: `STOCK_SKU`, `STOCK_NOMBRE`, `UBICACION`, `CATEGORIA`, `PROCEDIMIENTO`, `POLITICA`, `CREAR_TICKET`, `AMBIGUO`) y `entidades` (`sku`, `nombre`, `categoria`) y `confianza`
    - Usar `response_mime_type="application/json"` para salida determinista
    - El método `clasificar(mensaje, historial)` debe pasar el historial de la sesión como contexto al prompt
    - _Requisitos: R1.1, R1.2, R2.1, R3.1, R4.1, R5.1, R6.1_

  - [ ]* 10.2 Escribir prueba de propiedad P5 — Consultas normativas se enrutan al RAG y nunca al MCP
    - **Propiedad 5: Las consultas normativas se enrutan al RAG y nunca al MCP**
    - **Valida: Requisitos 4.3, 6.1**
    - Para toda consulta cuyo texto exprese procedimiento ante discrepancia o política comercial (generadas con `st.sampled_from` de plantillas variadas), el tipo retornado debe ser `PROCEDIMIENTO` o `POLITICA`, nunca un tipo que enrute al MCP
    - Mockear `GeminiClient.generate` para retornar tipos controlados según el contenido del prompt
    - Tag: `# Feature: hegia-stock, Property 5: Consultas normativas se enrutan al RAG y nunca al MCP`
    - _Requisitos: R4.3, R6.1_

  - [ ]* 10.3 Escribir pruebas unitarias para IntentClassifier
    - Caso: Consulta «¿Cuánto cemento Sol hay?» → tipo `STOCK_NOMBRE`, entidad nombre=«cemento Sol»
    - Caso: Consulta «¿Dónde está el SKU CEM-001?» → tipo `UBICACION`, entidad sku=«CEM-001»
    - Caso: Consulta sobre devolución → tipo `POLITICA`
    - Caso: Respuesta Gemini malformada (no JSON) → manejo de excepción, tipo `AMBIGUO`
    - _Requisitos: R1.1, R4.1, R6.1_

- [ ] 11. Implementar Orquestador de Conversación
  - [ ] 11.1 Implementar `ConversationOrchestrator`
    - Crear `backend/core/conversation_orchestrator.py` con `procesar(mensaje, session_id) -> ChatResponse`
    - Mantener contexto de sesión en memoria (dict `session_id → historial`) para pasar historial al clasificador
    - Implementar la tabla de enrutamiento completa del diseño (sección 3.2): cada tipo de intención → método correcto del servicio correspondiente
    - Para intenciones MCP, llamar a `MCPClient`; para normativas, llamar a `RAGEngine`; para tickets, validar SKU con MCP antes de crear ticket
    - Medir `duracion_ms` para cada respuesta e incluirlo en `ChatResponse`
    - _Requisitos: R1–R6, R9.1, R9.2_

  - [ ] 11.2 Implementar `ResponseFormatter`
    - Crear `backend/core/response_formatter.py` con `format_category_results(productos)` que filtra productos con `cantidad_disponible > 0`, toma los 10 de mayor cantidad y ordena de mayor a menor
    - Implementar `format_location(location)` que genera la cadena «Pasillo {N}, Estante {código}»
    - Implementar `format_multiple_matches(matches)` para presentar lista de coincidencias cuando hay más de un producto
    - _Requisitos: R2.6, R3.1, R3.2_

  - [ ]* 11.3 Escribir prueba de propiedad P1 — Búsqueda por nombre retorna todas las coincidencias exactas
    - **Propiedad 1: Búsqueda por nombre retorna todas las coincidencias exactas**
    - **Valida: Requisitos 1.2, 1.3, 2.2**
    - Para todo catálogo de productos y toda cadena de búsqueda que sea subconjunto léxico de uno o más `nombre_comercial`, `search_by_name` debe retornar exactamente los productos cuyo nombre contiene la cadena buscada (sin omisiones ni adiciones)
    - Usar `st.lists(st.text(min_size=1))` para catálogo y `st.text(min_size=1)` para query; la lógica de búsqueda se prueba sobre el formateador con datos mockeados
    - Tag: `# Feature: hegia-stock, Property 1: Búsqueda por nombre retorna todas las coincidencias exactas`
    - _Requisitos: R1.2, R1.3, R2.2_

  - [ ]* 11.4 Escribir prueba de propiedad P2 — Búsqueda de producto inexistente retorna «no encontrado»
    - **Propiedad 2: Búsqueda de producto inexistente retorna no encontrado**
    - **Valida: Requisitos 1.4, 2.4**
    - Para todo SKU o nombre que no esté en el catálogo mockeado, la respuesta del orquestador debe indicar «no encontrado» sin retornar ningún dato de inventario
    - Tag: `# Feature: hegia-stock, Property 2: Búsqueda de producto inexistente retorna no encontrado`
    - _Requisitos: R1.4, R2.4_

  - [ ]* 11.5 Escribir prueba de propiedad P4 — Filtro de categoría aplica correctamente stock, límite y orden
    - **Propiedad 4: Filtro de categoría aplica correctamente stock, límite y orden**
    - **Valida: Requisitos 3.1, 3.2**
    - Para toda lista de `ProductStock` de longitud arbitraria (con cantidades entre 0 y N), `format_category_results()` debe retornar una lista con solo productos de `cantidad_disponible > 0`, máximo 10 elementos, ordenados de mayor a menor `cantidad_disponible`
    - Usar `st.lists(st.builds(ProductStock, ...))` con cantidades generadas por `st.integers(min_value=0, max_value=1000)`
    - Tag: `# Feature: hegia-stock, Property 4: Filtro de categoría aplica correctamente stock, límite y orden`
    - _Requisitos: R3.1, R3.2_

  - [ ]* 11.6 Escribir pruebas unitarias para el orquestador
    - Caso: Intención `CREAR_TICKET`, SKU inválido → ticket rechazado, mensaje de error al usuario
    - Caso: MCP no disponible → respuesta «fuera de línea» en ≤ timeout configurado
    - Caso: RAG sin resultados → respuesta «no se encontró información normativa»
    - Caso: Formato ubicación → «Pasillo 3, Estante B-04» para pasillo=3, estante=«B-04»
    - _Requisitos: R2.6, R4.4, R5.3_

- [ ] 12. Implementar API Gateway FastAPI
  - [ ] 12.1 Implementar punto de entrada y middleware
    - Crear `backend/main.py` con la instancia `FastAPI`, registro de routers, middleware de sanitización PDP (que llama a `PDPFilter.sanitizar_para_log` antes de pasar el mensaje al orquestador) y `lifespan` para inicializar la base de datos con `init_db()`
    - Crear `backend/api/chat.py` con `POST /api/chat`: valida `ChatRequest`, aplica filtro PDP al mensaje, delega a `ConversationOrchestrator.procesar()`, retorna `ChatResponse`
    - Implementar `GET /api/health` que verifica conectividad MCP, disponibilidad ChromaDB y estado SQLite; retorna JSON con estado de cada componente
    - _Requisitos: R1–R6, R7.1, R9.1, R9.2_

  - [ ] 12.2 Implementar endpoints de tickets y documentos
    - Crear `backend/api/tickets.py` con `GET /api/tickets` (lista tickets con filtro opcional de estado) y `GET /api/tickets/{ticket_id}` (detalle de un ticket)
    - Crear `backend/api/documents.py` con `POST /api/documents` que recibe un archivo (PDF o Markdown), lo guarda en `DOCS_PATH` y llama a `RAGEngine.index_document()`; protegido con verificación de tipo de archivo
    - _Requisitos: R4.2, R5.2, R6.2_

  - [ ]* 12.3 Escribir pruebas unitarias para los endpoints
    - Usar `httpx.AsyncClient` con la app FastAPI en modo test
    - Caso: `POST /api/chat` con mensaje válido → status 200 y `ChatResponse` bien formada
    - Caso: `POST /api/chat` con mensaje vacío → status 422 (validación Pydantic)
    - Caso: `GET /api/tickets/{ticket_id}` con ID inexistente → status 404
    - Caso: `POST /api/documents` con archivo no PDF/MD → status 400
    - _Requisitos: R1.1, R5.2_

- [ ] 13. Implementar Frontend HTML + JavaScript
  - [ ] 13.1 Implementar interfaz de chat
    - Crear `frontend/index.html` con estructura semántica: área de historial de mensajes, campo de texto para la consulta, botón de envío y sección de estado del sistema
    - Crear `frontend/chat.js` con la lógica de envío (`fetch` a `POST /api/chat`), renderizado de burbujas de mensajes (usuario y asistente), indicador de carga durante la espera y manejo de errores de red
    - Mostrar el `ticket_id` (VER-NNN) en la burbuja cuando la respuesta incluya un ticket creado
    - Mostrar `fuentes` de documentos RAG al pie de las respuestas normativas
    - _Requisitos: R1–R6_

  - [ ] 13.2 Implementar estilos y accesibilidad
    - Crear `frontend/styles.css` con diseño responsivo y legible para uso en mostrador (tamaño de fuente mínimo 16px, contraste WCAG AA)
    - Atributos ARIA en el chat (`role="log"`, `aria-live="polite"` en el área de mensajes) y etiqueta `<label>` asociada al campo de entrada
    - _Requisitos: R1–R6_

- [ ] 14. Implementar pruebas de integración
  - [ ]* 14.1 Prueba de integración — Rendimiento MCP (p95 ≤ 3s)
    - Implementar `backend/tests/integration/test_performance_mcp.py` que ejecuta 100 consultas de stock consecutivas contra el Servidor_MCP real (marcadas con `pytest.mark.integration`) y calcula el percentil 95 de `duracion_ms`; el test falla si p95 > 3000 ms
    - _Requisitos: R9.1_

  - [ ]* 14.2 Prueba de integración — Rendimiento RAG (p95 ≤ 2s)
    - Implementar `backend/tests/integration/test_performance_rag.py` que ejecuta 100 consultas RAG consecutivas con documento de prueba indexado y calcula el percentil 95; el test falla si p95 > 2000 ms
    - _Requisitos: R9.2_

  - [ ]* 14.3 Prueba de integración — Flujo completo de consulta de stock
    - Enviar consulta «¿Cuántas unidades de cemento Sol hay?» al endpoint `/api/chat` real; verificar que la respuesta contiene SKU, descripción y cantidad en `duracion_ms` ≤ 3000
    - _Requisitos: R1.1, R1.2_

  - [ ]* 14.4 Prueba de integración — Flujo completo de creación de ticket
    - Enviar solicitud de verificación física para un SKU de prueba existente; verificar que se retorna `ticket_id` en formato VER-NNN con estado Pendiente
    - _Requisitos: R5.1, R5.2_

- [ ] 15. Checkpoint final — Verificar la implementación completa
  - Ejecutar `pytest backend/tests/ -v --tb=short` (sin flag `--integration`) y confirmar que todos los tests pasan
  - Verificar que `.env` no está versionado (`.gitignore`) y que `.env.example` tiene todos los campos requeridos
  - Consultar al usuario si hay ajustes antes de dar el trabajo por finalizado.

---

## Notes

- Las sub-tareas marcadas con `*` son opcionales y pueden omitirse para obtener un MVP más rápido; sin embargo, se recomienda ejecutarlas para validar las 11 propiedades de corrección del diseño.
- Las pruebas de integración (tarea 14) requieren que el Servidor_MCP esté configurado y que `GOOGLE_API_KEY` sea válida; se ejecutan con `pytest backend/tests/ --integration -v`.
- Cada prueba de propiedad Hypothesis usa `@settings(max_examples=200)` configurado en `conftest.py` con el perfil `"ci"`.
- El correlativo VER-NNN **nunca** se reinicia por día; el número sigue creciendo en toda la vida de la base de datos.
- Los únicos datos que pueden aparecer en `audit_log` y `tickets_verificacion` son los especificados en el diseño; cualquier campo adicional violaría la Ley N.° 29733.
- La migración de SQLite a PostgreSQL en el futuro solo requiere cambiar la cadena de conexión en `database.py`.

---

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["2.1", "2.2"] },
    { "id": 1, "tasks": ["2.3", "3.1"] },
    { "id": 2, "tasks": ["3.2", "3.3", "4.1"] },
    { "id": 3, "tasks": ["4.2", "4.3", "4.4", "6.1"] },
    { "id": 4, "tasks": ["6.2", "7.1", "8.1"] },
    { "id": 5, "tasks": ["7.2", "7.3", "7.4", "8.2", "8.3", "8.4"] },
    { "id": 6, "tasks": ["10.1"] },
    { "id": 7, "tasks": ["10.2", "10.3", "11.1"] },
    { "id": 8, "tasks": ["11.2"] },
    { "id": 9, "tasks": ["11.3", "11.4", "11.5", "11.6", "12.1"] },
    { "id": 10, "tasks": ["12.2"] },
    { "id": 11, "tasks": ["12.3", "13.1"] },
    { "id": 12, "tasks": ["13.2"] },
    { "id": 13, "tasks": ["14.1", "14.2", "14.3", "14.4"] }
  ]
}
```
