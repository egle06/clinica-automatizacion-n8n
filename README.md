# Clínica: Orquestación Multicanal con n8n

Workflow de automatización en **n8n** para una clínica que tiene problemas de comunicación: pacientes que no leen los correos, médicos que no se enteran a tiempo de los cambios y una recepción saturada.

El flujo toma los correos entrantes de **Gmail**, usa **Google Gemini** para clasificar su prioridad y distribuye la respuesta entre **Slack** (equipo interno) y **WhatsApp vía Twilio** (paciente).

## Contenido del repositorio

| Archivo | Descripción |
|---|---|
| `workflow_clinica_multicanal.json` | Blueprint del workflow, listo para importar en n8n |
| `README.md` | Este documento |

## Cómo funciona

```
Gmail Trigger
     |
     v
Basic LLM Chain (Google Gemini)  -->  clasifica prioridad y devuelve JSON
     |
     v
Parsear JSON de Gemini (Code)
     |
     v
If prioridad máxima
     |-- true  --> Slack - Alerta crítica --> WhatsApp (Twilio) al paciente
     |-- false --> Slack - Resumen al equipo
```

| Nodo | Función |
|---|---|
| **Gmail Trigger** | Detecta correos nuevos sin leer (OAuth2) |
| **Basic LLM Chain + Gemini** | Analiza el texto y devuelve `prioridad`, `resumen`, `accion_requerida` y `telefono_e164` |
| **Parsear JSON de Gemini** | Convierte la respuesta de la IA en campos utilizables |
| **If prioridad máxima** | Router: desvía el flujo según la prioridad asignada |
| **Slack - Alerta crítica** | Alerta con `@here` al canal `#urgencias-clinica` |
| **WhatsApp (Twilio)** | Mensaje dinámico al paciente, con su número en formato internacional (E.164) |
| **Slack - Resumen al equipo** | Resumen automático al canal `#resumen-bandeja` |

## Cómo usarlo

1. En n8n: **Workflows → ⋯ → Import from file** y seleccionar `workflow_clinica_multicanal.json`.
2. Conectar las credenciales propias en cada nodo:
   - **Gmail:** OAuth2 (cliente creado en Google Cloud con la Gmail API habilitada).
   - **Google Gemini:** API key de Google AI Studio.
   - **Slack:** Bot Token con los permisos `chat:write`, `chat:write.public` y `channels:read`.
   - **Twilio:** Account SID y Auth Token.
3. Crear en Slack los canales `urgencias-clinica` y `resumen-bandeja`, e invitar al bot.
4. En Twilio, unir el celular de prueba al Sandbox de WhatsApp (`join ...`).
5. Activar el workflow o ejecutarlo con **Execute workflow**.

## Notas

- **Seguridad:** el archivo no contiene tokens, API keys ni contraseñas. Las credenciales se configuran en cada instalación de n8n.
- **Ventana de 24 horas de WhatsApp:** con una cuenta de prueba de Twilio, el envío de texto libre puede ser rechazado con el error `21654 (ContentSid Required)`. Esto responde a la política de Meta, que exige una **plantilla pre-aprobada** para mensajes fuera de la ventana de 24 horas. En producción se resuelve usando un Content Template aprobado.
- **Formato internacional:** el teléfono del paciente se extrae del correo y se normaliza al formato E.164 (por ejemplo, `+549...`).
