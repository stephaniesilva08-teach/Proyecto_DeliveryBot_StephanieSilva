DeliveryBot – Gestión de Pedidos Internos de cafeteria
# 🛵 DeliveryBot - Sistema Automático de Pedidos y Gestión de Delivery

**DeliveryBot** es un sistema integral de automatización basado en **n8n** y **Telegram** para gestionar el ciclo completo de pedidos en un establecimiento gastronómico o tienda en línea. 

El sistema administra la interacción con el cliente (menú interactivo, carrito de compras, solicitud de dirección de envío y consulta de historial) y la comunicación con el panel/bot de cocina (notificación de pedidos entrantes y actualización de estados en tiempo real). Toda la información se persiste dinámicamente en **Google Sheets**.

---

## 📊 Base de Datos (Google Sheets)

La base de datos del sistema está centralizada en la siguiente hoja de cálculo de Google Sheets:

🔗 **[Acceder a la Base de Datos en Google Sheets](https://docs.google.com/spreadsheets/d/1YOy5qKHM1GZ30ZU68ZUrjLR5dOuIyy5ZtVUbM7b4U5I/edit)** *(Reemplaza este enlace si usas otra URL)*

### 📑 Estructura de Pestañas (Hojas):
1. **`MENU`**:
   * `id_producto`: Identificador único del producto (ej. `P001`).
   * `nombre`: Nombre del producto.
   * `descripcion`: Detalles del ingrediente/producto.
   * `precio`: Precio unitario.
   * `categoria`: Categoría del producto (ej. `bebidas`, `comidas`, `snacks`).
   * `stock`: Cantidad disponible (se descuenta automáticamente con cada pedido).
   * `disponible`: Estado de disponibilidad.
2. **`CARRITO`**:
   * `chat_id`: ID del chat del usuario en Telegram.
   * `id_pedido`: Código único generado para la orden (ej. `PED-12345`).
   * `detalle`: Resumen de items agregados.
   * `total`: Valor total a pagar.
   * `estado`: Estado actual del pedido (`borrador`, `En Preparación`, `En Camino`, `Entregado`).
   * `estado_conversacion`: Control del flujo de conversación (ej. `ESPERANDO_DATOS_ENTREGA`, `DATOS_RECIBIDOS`).
   * `datos_entrega`: Dirección y teléfono proporcionados por el cliente.
3. **`PEDIDOS`**:
   * Registro histórico de transacciones detalladas con fecha, productos y montos acumulados.

---

## 🏗️ Arquitectura de Flujos n8n

El proyecto se divide en 5 sub-flujos (workflows) interconectados:

### 1. `DeliveryBot - Principal-Corregido` (Enrutador de Entrada)
* **Función**: Punto de entrada para el bot del cliente.
* **Nodes clave**:
  * `Telegram Trigger`: Recibe los mensajes e interacciones (*callback query*) del cliente.
  * `¿Es Clic en Botón?` & `Router por Intención`: Determina si el usuario envió un comando de texto (`/start`, `hola`) o interactuó con un botón en línea.
  * `Code in JavaScript`: Evalúa el estado de conversación actual.
  * `Enviar Bienvenida y Menú`: Muestra las categorías principales (`🥤 Bebidas`, `🥪 Comidas`, `🍿 Snacks`, `🛒 Ver Carrito`, `📝 Historial / Estado`).
  * `Call Sub-workflow`: Llama al sistema completo de procesamiento.

### 2. `Sistema de Pedidos - Completo` (Lógica de Carrito y Stock)
* **Función**: Procesa la selección de productos, controla el inventario y envía comandas a la cocina.
* **Nodes clave**:
  * `Switch Accion`: Clasifica las acciones del cliente (`Producto Seleccionado`, `Confirmar final`, `botones`).
  * `Buscar Productos MENU` & `Filtrar y Armar Botones`: Obtiene de Sheets los productos de la categoría seleccionada y los presenta en formato de botones Inline.
  * `Procesar Pedido e ID` & `Calcular Nuevo Stock`: Genera un ID único para la orden y descuenta las unidades del inventario.
  * `Actualizar Stock`: Modifica el stock disponible en la hoja `MENU`.
  * `Armar Resumen Carrito` & `Append or update row in sheet`: Persiste la orden en la hoja `CARRITO`.
  * `HTTP Request1` (Comanda Cocina): Notifica inmediatamente al bot/chat de cocina con los detalles del nuevo pedido.

### 3. `DeliveryBot - Cocina` (Panel de Estado)
* **Función**: Permite al personal de cocina actualizar el estado del pedido mediante botones interactivos.
* **Nodes clave**:
  * `Telegram Trigger (Cocina)`: Captura los clics de la cocina (`estado_preparacion`, `estado_encamino`, `estado_entregado`).
  * `Parsear Callback Cocina` & `Actualizar Estado en Sheets`: Procesa la acción y actualiza la columna `estado` en la base de datos.
  * `Switch (Tipo de Estado)`: 
    * **En Preparación**: Notifica al cliente y le solicita la dirección de envío (`Solicitar Dirección`).
    * **En Camino**: Notifica al cliente que el repartidor va hacia la dirección registrada.
    * **Entregado**: Marca el pedido como completado.

### 4. `DeliveryBot - Direcciones` (Captura de Datos de Envío)
* **Función**: Recibe la respuesta del cliente cuando proporciona su dirección o teléfono.
* **Nodes clave**:
  * `Telegram Trigger (Cliente)`: Captura mensajes de texto.
  * `Buscar Cliente en Sheets` & `If (Esperando Datos?)`: Verifica si el cliente está en el estado `ESPERANDO_DATOS_ENTREGA`.
  * `Guardar Datos Entrega`: Registra la dirección en Google Sheets.
  * `Confirmar al Cliente`: Envia confirmación de recepción de dirección.

### 5. `DeliveryBot - Control Historial` (Consulta de Pedidos)
* **Función**: Permite a los usuarios consultar sus compras anteriores.
* **Nodes clave**:
  * `Es Comando /historial?`: Detecta cuando el cliente escribe `/historial`.
  * `Buscar Pedidos en HISTORIAL`: Filtra los registros en Google Sheets coincidentes con el `chat_id` del usuario.
  * `Formatear Mensaje Historial`: Renderiza los últimos 5 pedidos realizados con su formato, fecha, monto y resumen.

---

## 🚀 Requisitos e Instalación

### Requisitos Previos:
1. Instancia activa de **n8n**.
2. Dos bots de Telegram creados vía **@BotFather**:
   * Bot Cliente (para la atención a usuarios).
   * Bot Cocina / Admin (para recepción de comandas y actualización de estados).
3. Credenciales de **Google OAuth2** configuradas en n8n para lectura y escritura en Google Sheets.

### Pasos de Configuración:
1. **Importar los workflows**: Importa los 5 archivos JSON provistos en este repositorio dentro de tu instancia de n8n.
2. **Configurar Credenciales**:
   * Asigna las credenciales de Telegram API en los nodos `Telegram Trigger` y `Telegram`.
   * Asigna las credenciales de Google Sheets en todos los nodos del tipo `Google Sheets`.
3. **Vincular Hoja de Cálculo**:
   * Asegúrate de actualizar el `documentId` de Google Sheets en los nodos correspondientes con el ID de tu hoja de trabajo.
4. **Activar los Triggers**:
   * Cambia el estado de los workflows a `Active` para registrar los Webhooks de Telegram automáticamente.

---

## 🛠️ Tecnologías Utilizadas

* **n8n**: Orquestación y automatización de procesos.
* **Telegram Bot API**: Interfaz de comunicación interactiva (Inline Keyboards, Callbacks, Markdown).
* **Google Sheets API**: Persistencia de datos (Base de datos Serverless).
* **JavaScript (Node.js)**: Lógica interna de parseo, generación de IDs y formateo en nodos Code.
