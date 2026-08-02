# AI Content Automation System — Power Platform Stack

Un workflow de n8n que genera, almacena y publica contenido para LinkedIn de forma automática usando IA, integrado con el ecosistema Microsoft (Power Automate, Dataverse y Teams/Yammer) en lugar de Notion.

Es la versión Microsoft-native del AI Content Automation System — misma lógica de generación de contenido y calendario editorial, pero con Dataverse como base de datos de contenido y Power Automate como puente hacia la publicación.

## Qué hace

1. **Configuración del contenido** — Define manualmente (o vía inputs) el tema, audiencia, plataforma, tono e idioma del post a generar.
2. **Generación con IA** — Un nodo de OpenAI (GPT-5-nano) redacta un post profesional para LinkedIn con hook inicial, cuerpo claro y CTA final.
3. **Hashtags con IA** — Un segundo nodo de OpenAI genera 10 hashtags relevantes a partir del contenido generado.
4. **Preparación de datos** — Se estructuran título, contenido, hashtags, plataforma, estado (`draft`) y fecha de creación; el contenido se recorta a 1950 caracteres para respetar límites de publicación.
5. **Guardar en Dataverse** — Se crea una fila en una tabla de Dataverse ("Content Calendar") vía un flujo de Power Automate, con fecha de publicación sugerida (mañana).
6. **Enrutamiento por publicación**:
   - **Publicación inmediata**: si `publish_now` es verdadero, publica directamente en el canal de Teams/Yammer configurado y, tras esperar 1 hora, actualiza el estado en Dataverse a `published`.
   - **Publicación programada**: si no, espera 1 día, consulta en Dataverse los borradores pendientes (`status = Draft`), los publica en Teams/Yammer y actualiza su estado a `published`.

## Arquitectura

```
Manual Trigger ─> Set Config ─> GPT-5-nano (Post) ─> GPT-5-nano (Hashtags) ─> Set Data ─> Recortar contenido
                                                                                                  │
                                                                                                  ▼
                                                                          Crear fila en Dataverse (Content Calendar)
                                                                                                  │
                                                                                                  ▼
                                                                                    ¿Publicar ahora?
                                                                     ┌────────────────────┴────────────────────┐
                                                                    SÍ                                        NO
                                                                     │                                          │
                                                    Publicar en Teams/Yammer (inmediato)              Wait 1 día
                                                                     │                                          │
                                                              Wait 1 hora                    Consultar borradores en Dataverse
                                                                     │                                          │
                                                Actualizar estado en Dataverse                     Publicar en Teams/Yammer
                                                                                                                 │
                                                                                            Actualizar estado en Dataverse
```

## Stack

- **n8n** — motor de orquestación
- **Power Automate** — puente entre n8n y el ecosistema Microsoft mediante flujos activados por HTTP (endpoints de Logic Apps)
- **Dataverse** — calendario de contenido y seguimiento de estado (draft / published)
- **Microsoft Teams / Yammer** — canal de publicación del contenido corporativo
- **OpenAI GPT-5-nano** — generación de copy y hashtags, vía `@n8n/n8n-nodes-langchain`

## Setup

1. **Importa el workflow** en tu instancia de n8n.
2. **Crea los flujos de Power Automate** que respaldan cada nodo HTTP Request, cada uno expuesto como endpoint de Logic App:
   - `POWER_AUTOMATE_DATAVERSE_CREATE_CONTENT` — crea una fila en la tabla de contenido de Dataverse
   - `POWER_AUTOMATE_DATAVERSE_QUERY_DRAFTS` — devuelve las filas con `status = Draft`
   - `POWER_AUTOMATE_TEAMS_PUBLISH_POST` — publica el contenido en el canal de Teams/Yammer designado
   - `POWER_AUTOMATE_DATAVERSE_UPDATE_STATUS` — actualiza el estado de la fila a `published`
3. **Sustituye las URLs de ejemplo** (`https://prod-xx.westeurope.logic.azure.com/...`) en cada nodo HTTP Request por las URLs reales de tus flujos.
4. **Configura la credencial de OpenAI** en n8n para los nodos `Message a model` y `Message a model1` (modelo: `gpt-5-nano`).
5. **Crea la tabla en Dataverse** con las columnas: `title`, `content`, `hashtags`, `platform`, `status`, `publish_date`.
6. **Ajusta el prompt** en `Message a model` si quieres cambiar el tono, formato o estructura del post.
7. **Define el mecanismo de publicación real en LinkedIn** dentro del flujo de Power Automate `POWER_AUTOMATE_TEAMS_PUBLISH_POST` — por ejemplo, encadenando ahí mismo un conector de LinkedIn o Buffer si necesitas publicar fuera de Microsoft 365, o publicando directamente en el canal de Teams si el contenido es de uso interno.
8. Activa el workflow.

## Notas

- Todas las acciones del lado Microsoft pasan por Power Automate en lugar de nodos nativos de n8n, útil si tu organización centraliza integraciones por Power Platform por motivos de gobernanza o seguridad.
- El campo `publish_now` sustituye al antiguo `property_publish_date` de la versión Notion como condición de la rama `If`.
- Ninguno de los nodos de OpenAI se modificó — la generación de contenido con IA es idéntica a la versión original; solo cambió el almacenamiento (Notion → Dataverse) y la publicación (LinkedIn → Teams/Yammer vía Power Automate).
- Si necesitas publicar directamente en LinkedIn en lugar de en un canal de Teams, el flujo de Power Automate detrás de `POWER_AUTOMATE_TEAMS_PUBLISH_POST` puede incluir el conector nativo de LinkedIn que ofrece Power Automate.
