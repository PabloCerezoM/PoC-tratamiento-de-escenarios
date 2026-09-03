# PoC tratamiento de escenarios

## Qué es esto
Prueba de concepto: a partir de una carpeta de fotos de una localización (usada para
shootings de publicidad, principalmente moda), obtener automáticamente una foto
representativa por cada "grupo visual" (clustering) y una descripción textual de la
localización, usando Azure AI Foundry. La descripción está pensada como texto fuente
para generar un embedding por localización e indexarlo en Azure AI Search dentro de
una app — no es prosa de marketing, es texto denso en vocabulario descriptivo/sensorial
(estilo, luz, materiales, entorno, características espaciales) para maximizar similitud
semántica con búsquedas en lenguaje natural de un usuario.

## Estructura
- `Localizaciones/<nombre-localizacion>/*.jpg` — cada subcarpeta es una localización distinta
  (p. ej. `Casa Batllo`, `La Caleta`, `Solo House - Aragon`, `Teresitas Road`).
- `PoC_tratamiento-fotos-localizaciones.ipynb` — notebook principal de la PoC, en la
  **raíz** del proyecto (se movió fuera de `Localizaciones/`).
- `.venv/` — entorno virtual del proyecto (Python 3.13.15, ipykernel ya registrado).
- `requirements.txt` — dependencias pineadas del notebook.
- Repo con remoto en GitHub (`PabloCerezoM/PoC-tratamiento-de-escenarios`), **privado**.
  El usuario commitea/pushea también directamente desde el Source Control de VS Code,
  no solo a través de Claude Code — revisar `git log` antes de asumir el estado del repo.

## Entorno
- Kernel de Jupyter: seleccionar el intérprete de `.venv`. No usar el kernel global
  `.venv_upm`, no pertenece a este proyecto.
- Credenciales de Azure van en `.env` (ver `.env.example`), cargadas con `python-dotenv`.
  Nunca hardcodear endpoints/keys en el notebook ni en `.claude/CLAUDE.md`. Confirmado
  que `.env` nunca se ha commiteado en todo el historial.

## Pipeline implementado (ver el notebook, es la fuente de verdad para el detalle)
1. Ingesta local + normalización (Pillow, EXIF, resize a 768×768 máx.).
2. Embeddings de imagen vía Azure AI Vision — Multimodal Embeddings v4.0
   (`retrieval:vectorizeImage`, 1024 dims), con cache en disco (`.cache/embeddings/`,
   gitignored).
3. Dedup por similitud coseno, `DEDUP_THRESHOLD = 0.95`.
4. Clustering dinámico: `sklearn.cluster.HDBSCAN` (nativo desde sklearn 1.3+, no hace
   falta el paquete `hdbscan` aparte), `cluster_selection_method="leaf"` — el modo por
   defecto tiende a fusionar todo en 1-2 clusters gigantes con fotos visualmente
   homogéneas (mismo edificio/luz), "leaf" da grupos más finos.
5. Representante por cluster = medoid (embedding más cercano al centroide). Fotos sin
   grupo (ruido de HDBSCAN) se tratan como su propio cluster de una foto.
6. Descripción automática vía Azure OpenAI gpt-4o: patrón **map-reduce** porque Azure
   limita a 10 imágenes por llamada de chat y el número de representantes varía —
   lotes de ≤10 fotos independientes entre sí (sin contexto compartido, para evitar
   sesgo de anclaje) + una llamada final de síntesis solo texto. Prompts en celda
   dedicada y editable.
7. Seguimiento de coste/tokens instrumentado (`usage_log`, `usage_summary()`) — Vision
   se factura por transacción/imagen, OpenAI por token; precios son constantes a rellenar
   por el usuario, no hardcodear cifras de pricing sin confirmar.

Solo los pasos de embeddings y descripción dependen de Azure; el resto es Python local
(numpy, scikit-learn, Pillow, matplotlib).

## Convenciones
- No commitear `.venv/`, `.env`, `.cache/`, ni checkpoints de Jupyter (ver `.gitignore`).
- Las celdas de visualización (grids de fotos) incrustan las imágenes reales como base64
  en los outputs del `.ipynb` — sí acaban en git aunque los `.jpg` originales estén
  gitignored. Repo privado por ahora, pero tenerlo en cuenta si se comparte/hace público.
- Cambios de infraestructura Azure (crear recursos, desplegar modelos) se confirman
  con el usuario antes de ejecutarse — no son reversibles trivialmente.
