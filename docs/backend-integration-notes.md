# Notas de arquitectura: llevar este pipeline a producción

Contexto para el futuro proyecto de backend que integrará esta lógica con Power Platform
(Power Apps + Power Automate). Son conclusiones de una conversación de asesoramiento,
todavía no implementadas ni validadas en código — punto de partida, no diseño cerrado.

## Por qué no un Azure Function "a pelo" en plan Consumption

El pipeline depende de numpy, scikit-learn, Pillow — dependencias nativas pesadas. En un
plan Consumption de Azure Functions esto se nota en tamaño de paquete y cold start.
Alternativas:
- Functions en plan **Premium** (instancias precalentadas) o desplegado como **contenedor
  Docker** (evita los límites de zip-deploy).
- **Azure Container Apps**: empaquetar el pipeline como una API pequeña (p. ej. FastAPI) en
  un contenedor, con scale-to-zero. Menos fricción que Functions para stacks científicos
  pesados, coste en reposo prácticamente nulo. Se llama desde Power Automate con una acción
  HTTP genérica (no hay conector dedicado, pero tampoco hace falta).

**Recomendación de partida**: Container Apps, salvo que surja una razón concreta para
preferir el conector nativo de Functions en Power Automate.

## El problema real: síncrono vs. asíncrono

Procesar una localización (varias llamadas a Vision + varias a gpt-4o encadenadas, patrón
map-reduce) tarda más de lo que aguanta cómodamente una llamada HTTP síncrona de un flujo
de Power Automate.

Patrón propuesto:
1. Power Automate deja un mensaje en una **Storage Queue** (`{"localizacion": "...",
   "blobUrl": "..."}`) y sigue — operación instantánea, no espera respuesta.
2. El contenedor hace polling de la cola, procesa en background, y al terminar escribe el
   resultado (fotos representativas + descripción) en Dataverse.
3. El flujo (u otro flujo) reacciona al terminar vía el trigger "cuando se crea/modifica una
   fila" en Dataverse, en vez de esperar la respuesta HTTP.

Storage Queue (no Service Bus) es suficiente para este caso: un mensaje simple por
localización a procesar, sin necesidad de colas con prioridad, sesiones o dead-lettering
avanzado.

## Almacenamiento de fotos

El notebook actual lee de disco local. En producción, el contenedor no tiene acceso al
filesystem del usuario, así que las fotos deben subirse a **Blob Storage** — desde la Power
App directamente, o vía SharePoint + Power Automate copiándolas a Blob.

## Escribir en Dataverse desde el contenedor

Dataverse expone una Web API REST (OData v4) que cualquier cliente autenticado puede llamar.
Pasos necesarios:

1. **App Registration en Microsoft Entra ID** — Client ID + Client Secret (o certificado),
   flujo **client credentials** (machine-to-machine, sin usuario humano).
2. **Application User en Dataverse** — dentro del Power Platform admin center, en el
   environment correspondiente (Settings → Users → Application users), registrar esa
   aplicación de Entra ID.
3. **Security Role** — asignar al Application User un rol con permisos mínimos (create/read/
   write) solo sobre la tabla de localizaciones. Nunca System Administrator.
4. **Llamada desde Python**: pedir token OAuth2 a
   `https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token` (scope del entorno
   Dataverse, `https://<org>.crm4.dynamics.com/.default` o el dominio que corresponda), y
   usarlo como Bearer en `POST https://<org>.crm.dynamics.com/api/data/v9.2/<tabla>`.
   Librerías: `msal` (adquisición de token) + `requests` (llamadas REST) — mismo estilo que
   ya usa el notebook para Azure AI Vision y Azure OpenAI, sin SDK adicional.

Pendiente de definir: el esquema de la tabla de Dataverse (columnas: nombre de
localización, descripción, URLs de fotos representativas, etc.) — es un paso de modelado
de datos previo a la integración.

## Resumen de piezas nuevas necesarias

| Pieza | Para qué |
|---|---|
| Azure Container Apps | Ejecutar el pipeline (embeddings, dedup, clustering, descripción) |
| Storage Queue | Desacoplar el disparo desde Power Automate del procesamiento pesado |
| Blob Storage | Almacenar las fotos de las localizaciones (sustituye al disco local) |
| App Registration + Application User + Security Role | Autenticar el contenedor contra Dataverse |
| Tabla Dataverse (a modelar) | Almacenar resultados; fuente de datos nativa de la Power App |
| Azure AI Search (ya previsto en el objetivo original) | Indexar las descripciones para búsqueda semántica en la app |
