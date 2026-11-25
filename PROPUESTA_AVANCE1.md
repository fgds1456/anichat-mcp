# Proyecto Integrador — Unidad 4: AniChat
**Asignatura:** Programación Lógica y Funcional  
**Instituto Tecnológico de Nuevo León**  
**Integrante:** Diana Gaytan  
**Stack Tecnológico:** Node.js, Express, MCP SDK (`@modelcontextprotocol/sdk`), Gemini API, Stripe / Mercado Pago (Modo Prueba)

## 1. Idea, Problema, Usuarios y Objetivo

### Idea
**AniChat** es una plataforma web de streaming y catálogo de anime tipo Crunchyroll 
donde la interfaz principal es un **asistente conversacional con IA**. 
A través del chat, el usuario autenticado consulta títulos disponibles, 
gestiona su lista de seguimiento y progreso de episodios, recibe recomendaciones personalizadas y administra su suscripción entre plan gratuito y plan premium.

### Problema
Los asistentes conversacionales tradicionales no tienen visibilidad ni control sobre la sesión activa del usuario, 
su tipo de suscripción ni su historial real de visualización, lo que limita sus respuestas a información genérica 
y obliga al usuario a navegar manualmente por múltiples pantallas para gestionar su cuenta.

### Usuarios y Roles
* **Usuario Estándar (`USER` — Plan `FREE`):** Puede iniciar sesión, consultar el catálogo general, administrar su lista personal de seguimiento
  y acceder únicamente a episodios marcados como gratuitos.
* **Usuario Suscriptor (`USER` — Plan `PREMIUM`):** Conserva todas las funciones anteriores y desbloquea acceso completo al catálogo exclusivo,
  recomendaciones avanzadas y gestión prioritaria de listas mediante suscripción activa.
* **Administrador (`ADMIN`):** Puede consultar métricas globales de usuarios, suscripciones activas y catálogo,
  además de administrar los títulos disponibles en la plataforma.

### Objetivo
Desarrollar y desplegar una aplicación web funcional cuyo núcleo de interacción sea un chatbot conectado a un
**servidor MCP (Model Context Protocol)**, capaz de descubrir e invocar herramientas autorizadas que crucen la identidad del usuario 
autenticado con al menos dos bases de datos y un flujo real de suscripción en modo de prueba.

## 2. Arquitectura Preliminar y Diagrama

La aplicación separa la interfaz de usuario, la lógica de autenticación y permisos, el servidor MCP y las fuentes de datos para garantizar que el modelo de IA 
nunca acceda directamente a las bases de datos ni confíe en identificadores enviados sin validación del lado del servidor.

```text
+-------------------------------------------------------------------+
|                      FRONTEND WEB (HTML / JS / CSS)               |
|         Login / Registro   |   Chat Principal IA   |   Planes     |
+---------------------------------+---------------------------------+
                                  | HTTPS (JWT / Sesión)
                                  v
+-------------------------------------------------------------------+
|                   BACKEND & AGENTE IA (Node.js + Express)         |
|  - Validación de sesión y roles (USER / ADMIN, FREE / PREMIUM)    |
|  - Orquestador LLM (Gemini API) + Cliente MCP                     |
|  - Webhooks y endpoints de pago (Stripe / Mercado Pago Sandbox)   |
+----------------+---------------------------------+----------------+
                 | Protocolo MCP                   | API Pagos
                 v                                 v
+-----------------------------------+   +---------------------------+
|         SERVIDOR MCP              |   |  PASARELA DE PAGOS        |
|  Expone Tools con permisos:       |   |  (Modo Prueba / Sandbox)  |
|  1. obtener_perfil_y_plan         |   +---------------------------+
|  2. buscar_animes                 |
|  3. consultar_mi_lista            |
|  4. actualizar_progreso_episodio  |
|  5. recomendar_animes             |
|  6. estadisticas_admin_plataforma |
+--------+-----------------+--------+
         |                 |
         v                 v
+-----------------+ +---------------------+
|   BASE DE DATOS | |   BASE DE DATOS 2   |
|  1: Cuentas y   | |  Catálogo y         |
|  Suscripciones  | |  Actividad Usuario  |
+-----------------+ +---------------------+
```
Despliegue Previsto
Frontend: Vercel.

Backend + Servidor MCP: Render o Railway.

Bases de Datos: PostgreSQL / MySQL alojadas en Supabase o Railway (dos bases independientes).
Base de Datos 1: anichat_cuentas (Usuarios, Roles y Suscripciones)
Gestiona la identidad, autenticación, autorización por rol/plan y el estado de facturación.

Tabla usuarios

id (PK, UUID / INT)

nombre (VARCHAR)

email (VARCHAR, UNIQUE)

password_hash (VARCHAR)

rol (ENUM: 'USER', 'ADMIN')

plan (ENUM: 'FREE', 'PREMIUM')

creado_en (TIMESTAMP)

Tabla suscripciones

id (PK, UUID / INT)

usuario_id (FK -> usuarios.id)

proveedor (VARCHAR: 'stripe' / 'mercadopago')

referencia_externa (VARCHAR — ID de sesión o suscripción en sandbox)

estado (ENUM: 'inactiva', 'activa', 'cancelada')

fecha_inicio (TIMESTAMP)

fecha_fin (TIMESTAMP)
Base de Datos 2: anichat_catalogo (Contenido y Seguimiento)
Gestiona el catálogo de animes y el progreso personal de cada usuario.

Tabla animes

id (PK, INT)

titulo (VARCHAR)

genero (VARCHAR)

episodios_totales (INT)

solo_premium (BOOLEAN)

sinopsis (TEXT)

Tabla listas_usuario

id (PK, INT)

usuario_id (INT — referencia lógica al usuario autenticado en BD 1)

anime_id (FK -> animes.id)

estado (ENUM: 'pendiente', 'viendo', 'completado')

episodio_actual (INT)

actualizado_en (TIMESTAMP)

4. Flujo de Pago / Suscripción (Modo Prueba)
El usuario autenticado con plan FREE solicita actualizar a PREMIUM desde la interfaz o mediante el chatbot.

El backend en Node.js/Express crea una sesión de pago en modo de prueba (Stripe Checkout Test Mode o Mercado Pago Sandbox) asociada al usuario_id autenticado.

El usuario completa el pago utilizando credenciales/tarjetas de prueba del proveedor (sin datos reales).

El proveedor notifica el resultado mediante webhook (o confirmación verificada en servidor) y el backend actualiza:

Registro en anichat_cuentas.suscripciones con estado = 'activa'.

Campo plan = 'PREMIUM' en anichat_cuentas.usuarios.

A partir de ese momento, las herramientas MCP reflejan automáticamente el nuevo plan del usuario y desbloquean el catálogo exclusivo.
















|   BASE DE DATOS | |   BASE DE DATOS 2   |
|  1: Cuentas y   | |  Catálogo y         |
|  Suscripciones  | |  Actividad Usuario  |
+-----------------+ +---------------------+
