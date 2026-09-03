# PoC tratamiento de escenarios

## Qué es esto
Prueba de concepto: a partir de una carpeta de fotos de una localización (usada para
shootings de publicidad, principalmente moda), obtener automáticamente una foto
representativa por cada "grupo visual" (clustering) y una descripción textual de la
localización, usando Azure AI Foundry.

## Estructura
- `Localizaciones/<nombre-localizacion>/*.jpg` — cada subcarpeta es una localización distinta
  (p. ej. `Casa Batllo`, `La Caleta`, `Solo House - Aragon`, `Teresitas Road`).
- `Localizaciones/PoC_tratamiento-fotos-localizaciones.ipynb` — notebook principal de la PoC.
- `.venv/` — entorno virtual del proyecto (Python 3.13.15, ipykernel ya registrado).

## Entorno
- Kernel de Jupyter: seleccionar el intérprete de `.venv` (aparece como recomendado al
  abrir el notebook desde la raíz del workspace). No usar el kernel global `.venv_upm`,
  no pertenece a este proyecto.
- Credenciales de Azure van en `.env` (ver `.env.example`), cargadas con `python-dotenv`.
  Nunca hardcodear endpoints/keys en el notebook ni en `.claude/CLAUDE.md`.

## Pipeline acordado
1. Ingesta local de imágenes (Pillow, normalizar EXIF/orientación).
2. Embeddings de imagen vía Azure AI Vision — Multimodal Embeddings v4.0
   (`retrieval:vectorizeImage`, 1024 dims). Recurso Foundry, disponible solo en ciertas
   regiones — comprobar al desplegar.
3. Dedup / near-duplicates: similitud coseno entre embeddings, umbral ~0.97–0.98.
4. Clustering dinámico (número de grupos desconocido a priori): HDBSCAN o
   Agglomerative con `distance_threshold`, en local (scikit-learn / hdbscan).
5. Selección de representante por cluster: medoid (embedding más cercano al centroide).
6. Descripción automática: enviar la(s) foto(s) representativa(s) a un modelo de chat
   con visión desplegado en Azure OpenAI dentro de AI Foundry (gpt-4o / gpt-4.1),
   vía Chat Completions, con prompt orientado a shootings de moda/publicidad.

Solo los pasos 2 y 6 dependen de recursos de AI Foundry; el resto es Python local.

## Convenciones
- No commitear `.venv/`, `.env`, ni checkpoints de Jupyter (ver `.gitignore`).
- Cambios de infraestructura Azure (crear recursos, desplegar modelos) se confirman
  con el usuario antes de ejecutarse — no son reversibles trivialmente.
