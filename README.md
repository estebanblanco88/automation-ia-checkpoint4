# AutoFlow IA · Pre-Entrega 4 — Sincronización del Cerebro Agéntico con Ecosistemas de Negocio

**Curso:** Automation IA (CoderHouse) · **Alumno:** Esteban Blanco
**Archivo:** `checkpoint4_esteban_blanco.json` (n8n → Workflow → Import from File)

## Contexto

AutoFlow IA es una consultora B2B de automatización con IA. Este checkpoint parte del workflow del Módulo 3 (Manager Orquestador de Leads con memoria híbrida) y lo conecta con herramientas reales del negocio por OAuth2. El caso del vivo (Shopify → ERP → Slack → Klaviyo) se aplica con herramientas equivalentes que cumplen el mismo rol:

| Rol en el caso del vivo | Herramienta | Credencial |
|---|---|---|
| Casilla de soporte del cliente | Gmail (Trigger + Create Draft) | Gmail OAuth2 |
| CRM / fuente de datos del cliente | HubSpot (CRM gratuito) | HubSpot OAuth2 |
| Canal del equipo de operaciones | Slack (`#autoflow-ia-soporte-ops`) | Slack OAuth2 |
| Base operativa y memoria de largo plazo | Airtable (desde el M1/M3) | Airtable OAuth2 |

## Flujo

```
Gmail Trigger (casilla de soporte)
  └─ ① IF anti auto-reply ── Sí → STOP (bucle neutralizado)
        └─ No → Set · Limpiar payload → IF · ¿Payload válido? ── No → Descartar (evita 400)
              └─ Sí → Memoria híbrida M3 → Agente de Calificación → Orquestación W1/W2 → Memoria
                    └─ ② HubSpot Look up por email ── existe → Update / no existe → Create (evita 409)
                          └─ ③ Gmail Create Draft (respuesta de la IA, pendiente de aprobación humana)
                                └─ ④ Set payload mínimo → Slack · Aviso a Operaciones
                                      └─ Persistencia en Airtable + Log de observabilidad
```

## Criterios de la rúbrica

### 1. Conectividad segura y mínimo privilegio (40 %)
- Cuatro integraciones externas reales, todas por **OAuth2** con el semáforo en verde: Gmail, HubSpot, Slack y Airtable.
- *Allowed HTTP Request Domains = None* en todas las credenciales: solo pueden usarse desde sus nodos nativos, no desde un HTTP Request genérico.
- Operaciones acotadas a la función: HubSpot solo lee y escribe **contactos**; Gmail lee la casilla de soporte y crea **borradores** (no envía al cliente); Slack publica en **un único canal**.
- El Gmail Trigger filtra con `to:<casilla de soporte> -from:me`, de modo que solo procesa correos de soporte.
- **Limitación conocida:** en n8n Cloud las credenciales usan *Managed OAuth2*, cuyos scopes los define la app de n8n. En una instancia self-hosted se crearía una app propia limitada a `crm.objects.contacts.read`, `crm.objects.contacts.write` (HubSpot) y `chat:write` (Slack). Las cuentas de HubSpot y Slack son dedicadas al proyecto y no contienen datos reales de clientes.

### 2. Contención de bucles y mitigación de errores (35 %)
- **IF anti auto-reply** inmediatamente después del trigger. Descarta asuntos con *Auto-reply, Automatic reply, Respuesta automática, Out of office, Fuera de la oficina, Undeliverable, Delivery Status Notification*; remitentes *no-reply, noreply, do-not-reply, mailer-daemon, postmaster, bounce*; correos de la propia casilla; y encabezados RFC 3834 (`Auto-Submitted`, `Precedence: bulk/junk/list/auto_reply`, `X-Autoreply`).
- **Error 400:** el Set de limpieza deja solo email, nombre, asunto, cuerpo (sin citas, firmas, imágenes incrustadas, avisos legales ni HTML; máximo 3.000 caracteres) e IDs del hilo; un IF valida el formato del email y que el contenido no esté vacío antes de llegar a la IA o al CRM.
- **Error 409:** búsqueda por email en HubSpot antes de crear. Si el contacto existe se actualiza; si no, se crea. Evita el conflicto por email duplicado y los contactos repetidos.
- Reintentos x2 y *continue on error* en cada integración: un fallo externo no corta la persistencia ni el log.

### 3. Gobernanza operativa e interfaz HITL (25 %)
- El conector de correo usa exclusivamente la operación **Create Draft**: la respuesta redactada por la IA (Worker W2) queda como borrador **en el mismo hilo** del cliente.
- Ningún nodo envía correos al cliente. Un operador humano revisa, corrige y envía manualmente desde Gmail.
- Slack avisa al equipo con un payload mínimo (remitente, asunto, clasificación, ruta, estado de HubSpot y del borrador), sin cuerpo completo ni adjuntos.

## Reparto de roles: Airtable y HubSpot

Ambas herramientas guardan nombre, email y empresa, pero cada dato tiene un único dueño:

| | Airtable | HubSpot |
|---|---|---|
| Rol | Base operativa del sistema de IA | CRM del equipo comercial |
| Fuente de verdad de | El análisis de la IA: clasificación, prioridad, presupuesto, servicios recomendados (W1), estado del caso y memoria de sesiones | La identidad del contacto: quién es y cómo contactarlo |
| Qué registra | Solo leads calificados por el agente | Todo remitente válido que escribe a soporte |
| Quién lo usa | El workflow (lectura y escritura de memoria y resultados) | Las personas del equipo comercial |

Mejora prevista: guardar el ID de contacto de HubSpot en el registro de Airtable para vincular ambos sistemas.

## Test de regresión manual (paso 9)

Cada prueba se ejecutó con *Fetch Test Event* + *Execute Workflow*, enviando un correo real a la casilla de soporte.

| # | Escenario | Correo enviado | Resultado verificado |
|---|---|---|---|
| 1 | Lead completo | Nombre, empresa, problema y presupuesto USD 4.000 | Clasificación ALTO · ruta PROPUESTA_COMERCIAL · HubSpot **Create** · borrador en el hilo · aviso en Slack · lead en Airtable |
| 2 | Anti-409 | Correo nuevo del mismo remitente | Look up encuentra el contacto · IF por *true* · HubSpot **Update** con `isNew: false` · *Create* sin ejecutar · un solo contacto en HubSpot · Slack: "contacto existente actualizado (sin duplicar)" |
| 3 | Anti-bucle | Asunto "Respuesta automática: Consulta" | Se detiene en **STOP · Bucle neutralizado** · ni IA, ni CRM, ni borrador, ni Slack |
| 4 | Anti-400 | Sin asunto ni texto, solo firma | La limpieza elimina la firma (marcas HTML de Outlook/Gmail, nombre del remitente, separadores y avisos legales) · termina en **Descartar · Payload inválido** |
| 5 | HITL · datos incompletos | "Hola, quisiera saber qué servicios ofrecen." | El agente no registra · ruta **REVISION_HUMANA** · HubSpot Update · **sin borrador** · Slack: "No generado · requiere respuesta manual del equipo" · alerta al supervisor por correo |

Las capturas de cada ejecución están en la carpeta `evidencias/`.

## Configuración al importar

El JSON del repositorio no contiene claves, tokens ni IDs de credenciales: n8n exporta solo el tipo y el nombre de cada credencial, y los datos de la instancia original fueron reemplazados por marcadores. Para ejecutarlo en otra instancia:

| Marcador | Dónde aparece | Reemplazar por |
|---|---|---|
| `casilla-soporte+autoflow@tu-dominio.com` | Filtro de búsqueda del Gmail Trigger | La dirección de soporte (alias "+autoflow" de la casilla conectada) |
| `casilla-soporte@tu-dominio.com` | IF anti auto-reply (remitente = propia casilla), Log de Observabilidad y Alerta a supervisor | La casilla conectada en la credencial de Gmail |
| `appXXXXXXXXXXXXXX`, `tblLEADSXXXXXXXXX`, `tblMEMORIAXXXXXXX` | Nodos de Airtable | La base y las tablas *Leads* y *Memoria Sesiones* |
| `ID_WORKFLOW_W1`, `ID_WORKFLOW_W2` | Nodos *Delegar a W1 / W2* | Los sub-workflows *W1 Analista de Servicios* y *W2 Redactor de Correos* |
| Credenciales | Todos los nodos de integración | Credenciales OAuth2 propias de Gmail, HubSpot, Slack y Airtable, y la API key de Anthropic |
