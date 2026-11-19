# Assignment 2 – Modelo C4: DailyPulse, control de dailies en MS Teams

**Repositorio:** `s2026q4b-c4model-JulianRuiz`
**Autor:** Julián Ruiz
**Diagramas:** `c4model-dailypulse.drawio` (un archivo con 3 páginas: *C1 - Contexto*, *C2 - Contenedores*, *C3 - Componentes (Bot App)*)

---

## 1. Lo que entendí del modelo C4 (primera vez usándolo)

El modelo C4 describe un sistema como un mapa con zoom: cada nivel acerca la vista al anterior.

- **C1 – Contexto:** el sistema aparece como una sola caja. Se ve **quién lo usa** y **con qué sistemas externos habla**, sin tecnologías. Responde *“¿qué es y en qué entorno vive?”* y lo entiende cualquier persona del negocio.
- **C2 – Contenedores:** se abre la caja y aparecen las **piezas que se ejecutan o despliegan por separado** (una aplicación, una base de datos…), con su tecnología y la forma en que se comunican. Responde *“¿de qué piezas está hecho?”*.
- **C3 – Componentes:** se abre **un contenedor** y se ven sus **módulos internos**, cada uno con una responsabilidad, y cómo colaboran. Responde *“¿cómo se organiza el código?”* y sirve de guía para implementar.

## 2. Lógica de la estructura

El criterio principal fue que el diseño sea **lo más simple posible de implementar** en las próximas semanas. Por eso DailyPulse es **una sola aplicación Python** (el *Bot App*) más **un archivo SQLite**. La aplicación hace dos cosas. Escucha en `/api/messages` los mensajes que llegan de Teams y, con un bucle de `asyncio` (incluido en Python), revisa cada minuto si ya es la hora del recordatorio o del cierre. Así no hacen falta colas, funciones serverless ni librerías de programación de tareas. Teams no se conecta directamente con la app: los mensajes pasan por el **Bot Framework Service** de Microsoft. El bot se registra gratis en el **Teams Developer Portal**, sin suscripción de Azure. Toda la configuración (horas, canal y credenciales) va en un solo archivo `.env`. Los miembros se registran solos cuando instalan el bot, así que no hay que mantener una lista ni construir pantallas de configuración.

Dentro de la app hay **6 componentes**, uno por responsabilidad. Hay dos puntos de entrada: *Bot Handler* para lo que llega de Teams y *Daily Scheduler* para lo que pasa a una hora fija. *Report Service* maneja el reporte y *Rules* contiene las dos reglas del negocio (faltantes y tema repetido). Dos módulos hablan con el exterior: *Teams Sender* hacia Teams y *Database* hacia SQLite. Cada componente es un archivo `.py`, de modo que el diagrama C3 es prácticamente la estructura de carpetas del proyecto.

## 3. Cómo se cumple cada requisito

| # | Requisito | Solución | Componentes |
|---|-----------|----------|-------------|
| 1 | Notificación diaria a una hora definida | Cada minuto el bucle compara la hora actual (lunes a viernes, hora de Bogotá) con `HORA_DAILY` del `.env`. Cuando coinciden, envía la tarjeta a cada miembro por chat 1:1. | Daily Scheduler → Teams Sender |
| 2 | Reporte en casillas predefinidas | Adaptive Card con 4 casillas: **Yesterday**, **Today**, **Blockers** y **Tema / ticket**. Todas son obligatorias; Report Service lo vuelve a validar. | Bot Handler → Report Service |
| 3 | Validar quién no reporta | A la `HORA_CIERRE` se buscan los miembros sin reporte del día, se les envía un recordatorio y se publica en el canal la lista de quienes faltan. | Daily Scheduler → Rules → Teams Sender |
| 4 | Misma temática 3 veces seguidas | Se compara la casilla **Tema / ticket** de los últimos 3 reportes, en minúsculas y sin espacios. Si las 3 son iguales, se publica una alerta de posible bloqueo. | Report Service → Rules |
| 5 | Publicar en un channel | Cada reporte válido se publica de inmediato como tarjeta en el canal del equipo. | Report Service → Teams Sender |

## 4. Herramientas seleccionadas y por qué

Usé solo **5 herramientas**, más una de apoyo para desarrollo. Todo lo demás (servidor web, validación de tokens, tarjetas, programación de tareas, base de datos) ya viene incluido en alguna de ellas o en Python.

| # | Herramienta | Para qué se usa | Por qué se eligió |
|---|-------------|-----------------|-------------------|
| 1 | **Microsoft Teams** (con Adaptive Cards) | Interfaz de los usuarios: el formulario del daily con casillas, el canal y las alertas. | Es requisito del escenario y Acme ya trabaja ahí. Las Adaptive Cards son nativas de Teams, así que no hay que construir una app web ni un frontend. |
| 2 | **Teams Developer Portal** | Registrar el bot (single-tenant) y obtener sus credenciales. El portal lo conecta al **Bot Framework Service**, el servicio de Microsoft que entrega los mensajes entre Teams y la app. | Es gratis, no pide suscripción de Azure y se hace en minutos. Para un bot que solo vive en Teams dentro de una organización, hace lo mismo que el recurso Azure Bot. |
| 3 | **Python 3.12 + Microsoft 365 Agents SDK** | Toda la aplicación: recibir y validar los mensajes, enviar mensajes proactivos, publicar en el canal y programar las tareas diarias con `asyncio`. | Python es el lenguaje que más manejo. El Agents SDK es el SDK oficial y vigente de Microsoft para bots de Teams (reemplaza a `botbuilder`). Ya trae el servidor web (aiohttp) y la autenticación, y `asyncio` viene con Python, así que no hace falta una librería de scheduler. |
| 4 | **SQLite** | Guardar miembros y reportes. | Viene incluido en Python (`sqlite3`): no se instala nada y no hay servidor que administrar. El volumen es pequeño (un reporte por persona al día). Si se necesita escalar, se cambia a PostgreSQL modificando solo `db.py`. |
| 5 | **Archivo `.env`** | Credenciales del bot, horas del daily y del cierre, y canal donde publicar. | Un solo archivo de configuración y sin pantallas. Sigue Twelve-Factor: los secretos van fuera del código y `.env` está en `.gitignore`. |
| — | *Dev tunnels o ngrok* (solo desarrollo) | Darle a la app local una URL HTTPS pública. | El Bot Framework Service necesita una URL pública para entregar los mensajes; así se prueba sin desplegar nada. |
