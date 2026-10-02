# Requirements Document

## Introduction

HegiaStock es un asistente conversacional de consulta y verificación de inventario para Inversiones Hegiapesa S.A.C., empresa del sector retail ferretero, materiales de construcción y servicios afines. El sistema resuelve el descalce entre la información de inventario manejada en el mostrador y la existencia real de mercadería en almacén, eliminando las demoras en la atención al cliente y las ventas perdidas durante la verificación de producto. HegiaStock se integra con el servidor MCP de inventario existente (solo lectura), un motor RAG para consultas normativas y un módulo de registro de incidencias, todo ello sobre una pila Python 3.11+, FastAPI, SQLite y Gemini API, cumpliendo la Ley N.° 29733 de protección de datos personales.

## Glossary

- **Asistente**: El sistema conversacional HegiaStock en su totalidad.
- **Vendedor**: Usuario de mostrador que interactúa con el Asistente para atender consultas de clientes. Son 8 personas en turno de trabajo con acceso a un dispositivo conectado a la red local o a Internet.
- **Almacenero**: Usuario que gestiona físicamente el almacén y recibe solicitudes de verificación. Son 4 personas.
- **Administrador**: Usuario con rol de gerencia o administración que supervisa el sistema y carga archivos de datos.
- **SKU**: Código de referencia único (Stock Keeping Unit) que identifica un producto en el catálogo de inventario.
- **Nombre_Comercial**: Denominación textual de un producto tal como aparece en el catálogo (p. ej., «Cemento Sol 42.5 kg»).
- **Producto**: Artículo del catálogo identificado por un SKU y un Nombre_Comercial, con cantidad disponible y ubicación física registradas.
- **Categoría**: Agrupación lógica de Productos dentro del catálogo (p. ej., «Pinturas», «Clavos», «Conexiones PVC»).
- **Servidor_MCP**: Servidor de inventario propio (semana 4), operado en modo exclusivo de solo lectura, que expone datos de stock, ubicación y catálogo de los ~280 registros iniciales.
- **Motor_RAG**: Motor de recuperación aumentada por generación (Retrieval-Augmented Generation) que indexa los documentos normativos internos (guías de procedimientos, políticas comerciales, manuales) en formato PDF o Markdown.
- **Ticket_Verificacion**: Registro persistente en la base de datos SQLite que representa una solicitud formal de constatación física de un Producto con discrepancia de stock. Contiene identificador correlativo, SKU, estado y fecha de creación.
- **Estado_Pendiente**: Estado inicial asignado a todo Ticket_Verificacion recién creado, indicando que la constatación física aún no ha sido realizada.
- **Pasillo**: División física numerada del almacén donde se ubican los productos.
- **Estante**: Subdivisión dentro de un Pasillo identificada por un código alfanumérico (p. ej., «B-04»).
- **Discrepancia_Stock**: Situación en la que la cantidad disponible registrada en el Servidor_MCP no coincide con la presencia física real de un Producto en almacén.

---

## Requirements

### Requisito 1: Consulta de Stock Disponible

**Historia de usuario:** Como Vendedor, quiero consultar el stock disponible de un Producto por su SKU o Nombre_Comercial, para informar de inmediato al cliente si se cuenta con existencias y evitar ventas perdidas por falta de verificación.

#### Criterios de Aceptación

1. CUANDO el Vendedor envía una solicitud de consulta de stock indicando un SKU válido, EL Asistente SHALL responder mostrando el SKU, la descripción del Producto y la cantidad disponible en un tiempo igual o inferior a 3 segundos contados desde el envío de la solicitud.
2. CUANDO el Vendedor envía una solicitud de consulta de stock indicando un Nombre_Comercial válido, EL Asistente SHALL recuperar el Producto desde el Servidor_MCP por coincidencia de texto y mostrar el SKU, la descripción y la cantidad disponible en un tiempo igual o inferior a 3 segundos.
3. CUANDO el Nombre_Comercial ingresado coincide con más de un Producto en el Servidor_MCP, EL Asistente SHALL presentar al Vendedor la lista de coincidencias (SKU y Nombre_Comercial de cada una) para que seleccione el Producto deseado antes de mostrar el stock.
4. IF el SKU o Nombre_Comercial ingresado no existe en el catálogo del Servidor_MCP, THEN EL Asistente SHALL informar al Vendedor que el Producto no fue encontrado y sugerir revisar el código o la ortografía del nombre.
5. WHILE el Servidor_MCP no está disponible, EL Asistente SHALL notificar al Vendedor que el servicio de inventario está temporalmente fuera de línea e indicar que reintente en un máximo de 30 segundos.
6. THE Asistente SHALL invocar el Servidor_MCP exclusivamente en modo de solo lectura para toda consulta de stock, sin emitir instrucciones de modificación de datos.

---

### Requisito 2: Consulta de Ubicación Física de Producto

**Historia de usuario:** Como Vendedor, quiero consultar la ubicación física exacta de un Producto dentro del almacén, para orientar al Almacenero o localizar el artículo sin realizar desplazamientos infructuosos por los pasillos.

#### Criterios de Aceptación

1. CUANDO el Vendedor solicita la ubicación de un Producto mediante SKU o Nombre_Comercial, EL Asistente SHALL devolver el número de Pasillo y el código de Estante registrados en el Servidor_MCP en un tiempo igual o inferior a 3 segundos contados desde el envío de la solicitud.
2. CUANDO el Nombre_Comercial ingresado coincide con más de un Producto en el Servidor_MCP, EL Asistente SHALL presentar la lista de coincidencias (SKU y Nombre_Comercial) para que el Vendedor seleccione el Producto antes de devolver la ubicación.
3. IF el Producto no tiene ubicación física asignada en el Servidor_MCP, THEN EL Asistente SHALL informar al Vendedor que la ubicación no está registrada para ese Producto.
4. IF el SKU o Nombre_Comercial ingresado no existe en el catálogo, THEN EL Asistente SHALL informar al Vendedor que el Producto no fue encontrado en el catálogo de inventario.
5. WHILE el Servidor_MCP no está disponible, EL Asistente SHALL notificar al Vendedor que el servicio de inventario está temporalmente fuera de línea e indicar que reintente más tarde.
6. THE Asistente SHALL presentar la ubicación con el formato exacto «Pasillo {N}, Estante {código}» donde {N} es el número entero del pasillo y {código} es el identificador alfanumérico del estante registrado en el Servidor_MCP.

---

### Requisito 3: Filtro de Productos por Categoría

**Historia de usuario:** Como Vendedor, quiero filtrar y listar los Productos pertenecientes a una Categoría específica con sus existencias, para ofrecer alternativas al cliente cuando no recuerde el nombre exacto del artículo.

#### Criterios de Aceptación

1. CUANDO el Vendedor solicita el listado de Productos de una Categoría válida, EL Asistente SHALL recuperar del Servidor_MCP hasta 10 Productos de esa Categoría que tengan cantidad disponible mayor a cero y mostrar, por cada uno, el SKU, el Nombre_Comercial y la cantidad disponible, ordenados de mayor a menor cantidad disponible, en un tiempo igual o inferior a 4 segundos.
2. CUANDO la Categoría contiene más de 10 Productos con stock disponible, EL Asistente SHALL mostrar los 10 Productos con mayor cantidad disponible e informar al Vendedor que existen más productos en esa categoría.
3. IF la Categoría solicitada no existe en el catálogo del Servidor_MCP, THEN EL Asistente SHALL informar al Vendedor que la Categoría no fue encontrada y sugerir revisar el nombre de la categoría.
4. IF la Categoría existe pero ninguno de sus Productos tiene cantidad disponible mayor a cero, THEN EL Asistente SHALL informar al Vendedor que no hay existencias disponibles en esa Categoría en este momento.
5. WHILE el Servidor_MCP no está disponible, EL Asistente SHALL notificar al Vendedor que el servicio de inventario está temporalmente fuera de línea e indicar que reintente más tarde.
6. THE Asistente SHALL invocar el Servidor_MCP exclusivamente en modo de solo lectura para toda consulta de catálogo por Categoría.

---

### Requisito 4: Consulta de Procedimiento ante Discrepancias de Stock

**Historia de usuario:** Como Vendedor, quiero consultar el procedimiento operativo interno cuando un Producto no coincide entre el registro del sistema y la presencia en pasillo, para aplicar la regla institucional correcta sin interrumpir a gerencia.

#### Criterios de Aceptación

1. CUANDO el Vendedor consulta sobre el procedimiento a seguir ante una Discrepancia_Stock, EL Asistente SHALL recuperar la respuesta del Motor_RAG y presentar al menos una guía numerada de pasos institucionales en un tiempo igual o inferior a 2 segundos contados desde el envío de la consulta.
2. WHEN el Administrador carga o actualiza documentos normativos en el sistema, THE Motor_RAG SHALL indexar los nuevos documentos en formato PDF o Markdown para que el Asistente pueda recuperar procedimientos sobre Discrepancias_Stock.
3. WHEN la respuesta sobre el procedimiento ante Discrepancia_Stock es generada, THE Asistente SHALL obtenerla exclusivamente del Motor_RAG sin realizar consultas al Servidor_MCP.
4. IF el Motor_RAG no retorna resultados relacionados con el procedimiento consultado, THEN EL Asistente SHALL informar al Vendedor que no se encontró información normativa sobre esa consulta y recomendar consultar directamente con el Administrador.
5. WHILE el Motor_RAG no está disponible, EL Asistente SHALL notificar al Vendedor que el servicio de consulta normativa está temporalmente fuera de línea e indicar que reintente más tarde.

---

### Requisito 5: Registro de Solicitud de Verificación Física

**Historia de usuario:** Como Vendedor, quiero registrar una solicitud formal de verificación física sobre un Producto con Discrepancia_Stock, para que el Almacenero realice la constatación directa en el pasillo correspondiente.

#### Criterios de Aceptación

1. CUANDO el Vendedor solicita registrar una verificación física indicando un SKU válido, EL Asistente SHALL crear un Ticket_Verificacion en la base de datos SQLite con un identificador correlativo único en formato «VER-NNN» (donde NNN es un número entero de tres o más dígitos que se incrementa de forma continua sin reiniciarse por día), el SKU referenciado, el estado «Pendiente» y la fecha y hora de creación en hora local (UTC-5 Lima).
2. CUANDO el Ticket_Verificacion es creado exitosamente, EL Asistente SHALL confirmar al Vendedor la creación mostrando el identificador del ticket en formato «VER-NNN» y el estado «Pendiente» en tiempo igual o inferior a 3 segundos.
3. IF el SKU ingresado para el registro de verificación no existe en el catálogo del Servidor_MCP, THEN EL Asistente SHALL rechazar la creación del Ticket_Verificacion e informar al Vendedor que el SKU no es válido.
4. IF la escritura del Ticket_Verificacion en la base de datos SQLite falla, THEN EL Asistente SHALL informar al Vendedor que el ticket no pudo ser registrado e indicar que reintente o contacte al Administrador.
5. WHILE el Servidor_MCP no está disponible al momento de validar el SKU, EL Asistente SHALL notificar al Vendedor que no es posible validar el SKU en este momento e indicar que reintente más tarde.
6. THE Asistente SHALL garantizar que el identificador correlativo «VER-NNN» sea único dentro de toda la base de datos de tickets, sin duplicados.
7. THE Asistente SHALL registrar en el Ticket_Verificacion exclusivamente el SKU del Producto y los metadatos del ticket (identificador, estado, fecha y hora), sin almacenar datos personales de clientes ni del Vendedor.

---

### Requisito 6: Consulta de Políticas Comerciales, de Devolución y Garantía

**Historia de usuario:** Como Vendedor, quiero consultar las políticas comerciales de devolución y garantía de mercadería, para responder con precisión las consultas de los clientes durante la venta sin interrumpir la atención.

#### Criterios de Aceptación

1. CUANDO el Vendedor consulta sobre políticas de devolución o garantía de un Producto o Categoría de mercadería, EL Asistente SHALL recuperar del Motor_RAG las cláusulas institucionales cuyo texto contenga términos relacionados con la consulta y presentar hasta 3 cláusulas relevantes en un tiempo igual o inferior a 2 segundos contados desde el envío de la consulta.
2. WHEN el Administrador carga o actualiza el manual de políticas comerciales, THE Motor_RAG SHALL indexar el documento en formato PDF o Markdown para que el Asistente pueda recuperar cláusulas de devolución y garantía.
3. WHEN la respuesta sobre políticas comerciales es generada, THE Asistente SHALL presentar al Vendedor el texto de las cláusulas recuperadas del Motor_RAG sin modificar su contenido, identificando el documento fuente.
4. IF el Motor_RAG no retorna cláusulas relacionadas con la política consultada, THEN EL Asistente SHALL informar al Vendedor que no se encontró información sobre esa política y recomendar consultar directamente con el Administrador.
5. WHILE el Motor_RAG no está disponible, EL Asistente SHALL notificar al Vendedor que el servicio de consulta de políticas está temporalmente fuera de línea e indicar que reintente más tarde.

---

### Requisito 7: Protección de Datos Personales

**Historia de usuario:** Como Administrador, quiero que el sistema no registre ni almacene datos personales de clientes en ningún componente, para cumplir con la Ley N.° 29733 de Protección de Datos Personales del Perú.

#### Criterios de Aceptación

1. THE Asistente SHALL registrar cero (0) campos de datos personales de clientes — incluyendo DNI, nombres completos, teléfonos, direcciones o números de tarjeta de pago — en logs de aplicación, base de datos SQLite y cualquier otro componente del sistema.
2. THE Asistente SHALL procesar consultas utilizando únicamente datos de inventario (SKUs, cantidades, ubicaciones, categorías) y metadatos de tickets de verificación (identificador, SKU, estado, fecha y hora).
3. IF el Vendedor incluye datos personales identificables de clientes en el texto de una consulta conversacional, THEN EL Asistente SHALL responder a la intención de la consulta sin persistir ni registrar esos datos personales en ningún componente del sistema.
4. WHEN el sistema genera un log de evento o auditoría, THE Asistente SHALL omitir de dicho log cualquier campo que corresponda a datos personales de clientes conforme a la Ley N.° 29733.

---

### Requisito 8: Operación en Modo Solo Lectura del Servidor MCP

**Historia de usuario:** Como Administrador, quiero que toda comunicación con el Servidor_MCP sea estrictamente de solo lectura, para preservar la integridad del inventario y evitar modificaciones accidentales o no autorizadas.

#### Criterios de Aceptación

1. THE Servidor_MCP SHALL rechazar con un código de error cualquier instrucción de modificación de datos (equivalentes a INSERT, UPDATE o DELETE) proveniente del Asistente, sin ejecutar ningún cambio.
2. THE Asistente SHALL invocar el Servidor_MCP únicamente mediante operaciones de consulta de solo lectura (equivalentes a SELECT) para todas las funcionalidades dentro del alcance declarado.
3. IF el Asistente recibe una respuesta de error del Servidor_MCP indicando intento de escritura rechazado, THEN EL Asistente SHALL registrar el evento en el log del sistema incluyendo la operación intentada, la marca de tiempo y el código de error recibido, y notificar al Administrador.
4. WHEN el Asistente construye una invocación al Servidor_MCP, THE Asistente SHALL validar que la operación es de solo lectura antes de enviarla, descartando cualquier invocación que contenga instrucciones de escritura.

---

### Requisito 9: Rendimiento de Consultas

**Historia de usuario:** Como Vendedor, quiero que las consultas al sistema respondan en tiempos breves y predecibles, para no generar esperas al cliente durante la atención en el mostrador.

#### Criterios de Aceptación

1. THE Asistente SHALL responder el 95% de las consultas de stock e inventario al Servidor_MCP en un tiempo igual o inferior a 3 segundos medido desde el envío de la solicitud hasta la presentación del resultado, en una ventana de medición de 100 transacciones consecutivas en condiciones normales de operación.
2. THE Asistente SHALL responder el 95% de las consultas al Motor_RAG (procedimientos y políticas) en un tiempo igual o inferior a 2 segundos medido desde el envío de la solicitud hasta la presentación del resultado, en una ventana de medición de 100 transacciones consecutivas en condiciones normales de operación.
3. IF una consulta al Servidor_MCP alcanza los 5 segundos sin respuesta, THEN EL Asistente SHALL cancelar la solicitud, notificar al Vendedor que la consulta excedió el tiempo máximo de espera e indicar que reintente.
4. IF una consulta al Motor_RAG alcanza los 4 segundos sin respuesta, THEN EL Asistente SHALL cancelar la solicitud, notificar al Vendedor que la consulta normativa excedió el tiempo máximo de espera e indicar que reintente.
