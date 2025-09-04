# Propel AI Responses

Servicio **FastAPI + OpenAI** que genera respuestas breves y accionables en función de **temas preconfigurados** (*topics* configurado por defecto a temas de financiamiento y/o apoyo a proyectos digitales para ONGs) y un flujo de automatización con **n8n**.

## ¿Qué hace?
- Expone un endpoint `POST /answer` (FastAPI) que:
  - Matchea la pregunta contra **topics** en `topics.json` (usando RapidFuzz).
  - Construye un prompt y llama a **OpenAI** para generar la respuesta.
  - Devuelve JSON con la respuesta y el topic detectado.
- **n8n** recibe preguntas vía **webhook** y:
  - Reenvía la pregunta al API (`/answer`).
  - Devuelve el resultado al cliente.
  - **Guarda la respuesta** como archivo `.json` en `automation/out/` (demostración de automatización).

## Arquitectura (resumen)
```
Cliente (curl / formulario / bot)
        │
        ▼
  n8n Webhook  ──► HTTP Request → http://api:8000/answer (FastAPI)
        │                                 │
        │                                 └─ OpenAI (chat.completions)
        ├─► Respond to Webhook (respuesta inmediata)
        └─► Convert to File → Write File (automation/out/response-*.json)
```

---

## Requisitos
- **OpenAI API Key**
- **Docker** y **Docker Compose**

---

## Start · Ejecutar con Docker Compose
1. Clonar y configurar variables:
   ```bash
   cp .env.example .env
   # Edita .env y pon tu API key
   # OPENAI_API_KEY=tu_api_key_de_openai
   ```

2. Levantar servicios:
   ```bash
   docker compose up --build
   ```
   - **API**: `http://localhost:8000`
   - **n8n**: `http://localhost:5678`

3. Importar y activar el workflow en **n8n**:
   - Abrir `http://localhost:5678` (crea cuenta demo si es la primera vez).
   - Menú **Workflows → Import from file** → selecciona `automation/n8n-workflow.json`.
   - Abrir el workflow importado y **verificar** el nodo **HTTP Request (FastAPI /answer)**:
     - **Method**: `POST`
     - **URL**: `http://api:8000/answer`.
     - **Body (JSON)**:
       ```json
       { "question": "={{$json.question || $json.body?.question}}" }
       ```
   - **Activar** el workflow.

4. Probar con **webhook de n8n**:
   ```bash
   curl -X POST "http://localhost:5678/webhook/propel-qa" \
     -H "Content-Type: application/json" \
     -d '{"question":"¿Dónde busco grants para una ONG de educación en Colombia?"}'
   ```
   - Respuesta: JSON con `body.answer`, `body.topic`, `statusCode=200`.
   - Se genera un archivo `.json` en `automation/out/`.

5. Probar **directo al API** (opcional):
   ```bash
   curl -X POST "http://localhost:8000/answer" \
     -H "Content-Type: application/json" \
     -d '{"question":"¿Cómo organizar mi recolección de datos?"}'
   ```
   Salud de la API:
   ```bash
   curl http://localhost:8000/health
   ```

---

## Configuración de Topics (`topics.json`)

Este archivo contiene los **temas/plantillas** que representan el “conjunto de preguntas definidas previamente” del enunciado.

Estructura de un topic:
```json
{
  "id": "financiacion_inicio",
  "titulo": "¿Cómo aplicar a fondos internacionales?",
  "descripcion": "Mapeo de donantes, encaje temático, dossier y calendario.",
  "ejemplos_usuario": [
    "¿Cómo aplicar a fondos internacionales?",
    "¿Dónde busco grants?",
    "Necesito financiamiento para mi ONG",
    "Fondos internacionales para proyectos sociales",
    "Cómo aplicar a convocatorias de cooperación"
  ],
  "plantilla_respuesta": [
    "Lista donantes alineados a tu tema/país.",
    "Prepara dossier (ToC, KPIs, presupuesto).",
    "Asegura compliance básico.",
    "Siguiente paso: redacta un pitch de 3 líneas (problema, solución, impacto)."
  ]
}
```

### ¿Cómo agregar un topic nuevo?
1. Edita `topics.json` y agrega un objeto con:
   - `id` único (snake-case recomendado).
   - `titulo` y `descripcion` claras.
   - `ejemplos_usuario` realistas para ayudar al match.
   - `plantilla_respuesta` con **3–6 bullets** y **terminar con “Siguiente paso: …”**.
2. Prueba con una pregunta que encaje con la `descripcion` o `ejemplos_usuario`.

---

## Variables de entorno

Archivo `.env`:
```env
PORT=8000
OPENAI_API_KEY=tu_api_key_de_openai
OPENAI_MODEL=gpt-4o-mini
TOPIC_THRESHOLD=45   # umbral de similitud (0-100)
TOPIC_TOPK=3        # cuántas alternativas evaluar
```

- `TOPIC_THRESHOLD` controla cuán “estricto” es el match (RapidFuzz). Por debajo, el servicio toma un fallback razonable.

---

## Estructura del repo (resumen)
```
.
├─ automation/
│  ├─ n8n-workflow.json      # Workflow para importar en n8n
│  └─ out/                   # (generado) archivos JSON de respuestas
├─ n8n_data/                 # (runtime n8n) NO se comitea
├─ src/
│  ├─ ai_client.py           # Cliente OpenAI
│  ├─ main.py                # FastAPI app
│  ├─ prompt.py              # System prompt + builder del user message
│  └─ topics.py              # Carga + matching de topics (RapidFuzz)
├─ topics.json               # Topics preconfigurados
├─ requirements.txt
├─ Dockerfile
├─ docker-compose.yml
├─ .env.example
└─ README.md
```

---

## n8n: exportar workflow actualizado (opcional)
Si ajustas el workflow en la UI y quieres versionarlo:
1. Menú **Workflows → Export** → **Download to file**.
2. Reemplaza `automation/n8n-workflow.json` en el repo con el archivo exportado.

---

## Troubleshooting
- **Webhook 404** en n8n: asegúrate de que el workflow esté **Active**
- **No se generan archivos** en `automation/out/`:
  - El workflow debe estar **Active**.
  - El volumen `./automation/out:/data/out` debe existir (Docker Compose).
  - El nodo “Read/Write Files from Disk” debe apuntar a `=/data/out/response-{{$now}}.json`.
- **Errores con OpenAI**: revisa `OPENAI_API_KEY` en `.env` y conectividad saliente desde el contenedor.

---

## 📜 Licencia

Propósito de prueba técnica.
