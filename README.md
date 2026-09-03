# PoC tratamiento de escenarios

Prueba de concepto: a partir de las fotos de una localización (shootings de moda/publicidad),
selecciona automáticamente una foto representativa por cada ambiente visualmente distinto y
genera una descripción de texto pensada para indexar la localización en Azure AI Search
(búsqueda semántica).

## Estructura

- `PoC_tratamiento-fotos-localizaciones.ipynb` — notebook con todo el pipeline, explicado paso a paso.
- `Localizaciones/<nombre>/*.jpg` — fotos de cada localización (no versionadas en git).
- `requirements.txt` — dependencias del proyecto.
- `.env.example` — plantilla de credenciales de Azure.

## Puesta en marcha

```
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
```

Copia `.env.example` como `.env` y rellena `AZURE_AI_KEY`, `AZURE_OPENAI_ENDPOINT` y
`AZURE_VISION_ENDPOINT` con los valores de tu recurso de Azure AI Foundry
(*Keys and Endpoint* del recurso, no del proyecto).

Abre el notebook en VS Code, selecciona el intérprete de `.venv` como kernel, y ejecuta las
celdas en orden.

## Pipeline

1. Ingesta y normalización de fotos (Pillow).
2. Embeddings de imagen — Azure AI Vision (*multimodal embeddings v4.0*).
3. Deduplicación por similitud coseno.
4. Clustering dinámico — HDBSCAN (scikit-learn).
5. Selección del representante de cada grupo (medoid).
6. Descripción automática — Azure OpenAI gpt-4o (patrón map-reduce).
7. Seguimiento de tokens/coste de cada llamada a Azure.

Más detalle de cada paso, en el propio notebook y en `.claude/CLAUDE.md`.
