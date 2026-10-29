# embedded-llm-sandbox
ollama llm run on docker in RAG infrastructure

# 🏛️ Arquitectura de la Aplicación de IA (RAG) Local

Este documento detalla la arquitectura técnica, el flujo de datos y la configuración de contenedores para la aplicación de Inteligencia Artificial con arquitectura **RAG (Retrieval-Augmented Generation)** utilizando procesamiento local.

---

## 📦 1. Diagrama de Contenedores (Docker Compose)

La infraestructura está completamente dockerizada y desacoplada en **4 contenedores independientes** conectados mediante una red interna de Docker (`rag_network`).

```text
+------------------------------------------------------------------------------------+
|                                 RAG_NETWORK (Docker)                              |
|                                                                                   |
|  +--------------------+      +--------------------+      +---------------------+  |
|  | 1. ollama_service  | <--- |   2. fastapi_api   | <--- | 3. streamlit_client |  |
|  | (Inferencia LLM)   |      | (Backend + MiniLM) |      | (Interfaz de Chat)  |  |
|  +--------------------+      +--------------------+      +---------------------+  |
|                                        |                                          |
|                        ┌───────────────┴───────────────┐                          |
|                        ▼                               ▼                          |
|              +--------------------+          +--------------------+               |
|              | 4. vector_database |          | 5. postgres_sql    |               |
|              | (ChromaDB)         |          | (Historial/Métricas|               |
|              +--------------------+          +--------------------+               |
+------------------------------------------------------------------------------------+
```

---

## 🔀 2. Diagrama de Flujo de Comunicación (Tiempo Real)

El siguiente flujo describe cómo interactúan los componentes desde que el usuario envía una pregunta hasta que recibe la respuesta enriquecida:

```text
[Usuario] 
   │
   ▼ (1. Escribe pregunta)
[3. streamlit_client]
   │
   ▼ (2. Petición POST HTTP /api/v1/chat)
[2. fastapi_api] 
   │
   ├─► (3. Procesa texto localmente con MiniLM y genera el Vector de la pregunta)
   │
   ├─► [4. vector_database (ChromaDB)]
   │      │ 
   │      └─► (4. Busca y devuelve los 3 fragmentos de Kaggle más parecidos numéricamente)
   │
   ├─► [1. ollama_service (Llama 3.2:1b)]
   │      │
   │      └─► (5. Envía Pregunta + Contexto. El LLM redacta la respuesta final)
   │
   ├─► [5. postgres_sql]
   │      │
   │      └─► (6. Guarda de forma asíncrona la pregunta, respuesta y latencia para analítica)
   │
   ▼ (7. Devuelve la respuesta final redactada)
[3. streamlit_client] ──► [Usuario ve la respuesta en pantalla]
```

---

## 🛠️ 3. Librerías Recomendadas por Contenedor

Para levantar y construir el código interno de cada contenedor, se deben utilizar las siguientes dependencias de Python y del sistema:

### Contenedor 1: `ollama_service` (Inferencia LLM)
*   **Base:** Imagen oficial de Docker `ollama/ollama:latest`.
*   **Modelo cargado:** ´´Por confirmar´.

### Contenedor 2: `fastapi_api` (Backend + MiniLM)
*   **Base:** Imagen `python:3.10-slim`.
*   **Librerías Python (`requirements.txt`):**
    *   `fastapi` y `uvicorn` (Para levantar el servidor web y los endpoints).
    *   `sentence-transformers` (Para cargar y ejecutar el modelo **MiniLM** de forma local y generar los embeddings).
    *   `chromadb` (Cliente oficial para conectarse a la base vectorial).
    *   `psycopg2-binary` o `asyncpg` (Para conectarse y guardar el historial en PostgreSQL).
    *   `pydantic` (Para la validación estricta de los esquemas de datos de entrada y salida).

### Contenedor 3: `streamlit_client` (Frontend)
*   **Base:** Imagen `python:3.10-slim`.
*   **Librerías Python:**
    *   `streamlit` (Para construir la interfaz gráfica de chat).
    *   `requests` (Para consumir la API de FastAPI mediante peticiones HTTP).

### Contenedor 4: `vector_database` (ChromaDB)
*   **Base:** Imagen oficial `chromadb/chroma:latest`.

### Contenedor 5: `postgres_sql` (Auditoría y Analítica)
*   **Base:** Imagen oficial `postgres:15-alpine` (Ligera y optimizada para Linux).

---

## 🚀 4. Diagrama de Despliegue Local (Hardware)

Este diagrama muestra cómo se distribuyen los componentes dentro de los componentes físicos de tu computadora de desarrollo (**Acer Aspire / Fedora 44**):

```text
+-------------------------------------------------------------------------------------+
| 💻 HARDWARE FÍSICO (Laptop Acer Aspire - CPU Ryzen)                                 |
|                                                                                     |
|  +-------------------------------------------------------------------------------+  |
|  | 🐧 SISTEMA OPERATIVO (Fedora 44 con Kernel Linux Nativo)                      |  |
|  |                                                                               |  |
|  |  +-------------------------------------------------------------------------+  |  |
|  |  | 🧠 MEMORIA RAM (Asignación estimada de recursos en ejecución)           |  |  |
|  |  |                                                                         |  |  |
|  |  | [Ollama (LLM por confirmar)]  ~1.2 GB RAM (Uso intensivo de hilos)      |  |  |
|  |  | [FastAPI + MiniLM]    ~800 MB RAM (Carga del modelo en caché)           |  |  |
|  |  | [ChromaDB + Postgres] ~500 MB RAM                                       |  |  |
|  |  | [Streamlit UI]        ~200 MB RAM                                       |  |  |
|  |  | ----------------------------------------------------------------------- |  |  |
|  |  | Total Estimado:       ~2.7 GB - 3.2 GB RAM                              |  |  |
|  |  +-------------------------------------------------------------------------+  |  |
|  +-------------------------------------------------------------------------------+  |
+-------------------------------------------------------------------------------------+
```
