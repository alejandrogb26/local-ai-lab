# 02. Arquitectura general

## 1. Introducción

La plataforma desarrollada en este proyecto se plantea como un laboratorio de inteligencia artificial local integrado dentro de una infraestructura informática propia.

La arquitectura no está formada únicamente por un modelo de lenguaje. El funcionamiento completo depende de varias capas que deben trabajar conjuntamente:

```text
┌─────────────────────────────────────────────────────────────┐
│                         USUARIO                             │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                    CAPA DE APLICACIÓN                       │
│                                                             │
│   OpenWebUI · n8n · OpenCode · futuras aplicaciones         │
└────────────────────────────┬────────────────────────────────┘
                             │
                    HTTP / REST / API
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                 CAPA DE INFERENCIA                         │
│                                                             │
│                         Ollama                             │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                       MODELOS                               │
│                                                             │
│ Qwen · Gemma · modelos especializados · futuros modelos    │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                  RECURSOS DE COMPUTACIÓN                   │
│                                                             │
│              CPU · RAM · GPU · VRAM · almacenamiento        │
└─────────────────────────────────────────────────────────────┘
```

Esta separación es importante porque permite sustituir o ampliar una capa sin tener que rediseñar necesariamente las demás.

Por ejemplo, cambiar el modelo de lenguaje no implica necesariamente cambiar OpenWebUI. Del mismo modo, una aplicación externa puede utilizar directamente la API de Ollama sin necesidad de utilizar OpenWebUI.

La arquitectura se ha diseñado, por tanto, alrededor de un principio de desacoplamiento entre interfaz, inferencia y modelo.

---

## 2. Visión general de la infraestructura

El proyecto se ejecuta sobre infraestructura de virtualización propia basada en Proxmox.

Dentro de esta infraestructura se dispone de un nodo dedicado al laboratorio de IA, identificado durante las pruebas como `ai01`.

Sobre este entorno se ejecutan los servicios necesarios mediante Docker y Docker Compose.

La arquitectura actual puede resumirse como:

```text
                    INFRAESTRUCTURA PROXMOX
                             │
                             ▼
                     ┌───────────────┐
                     │     ai01      │
                     │  Nodo IA      │
                     └───────┬───────┘
                             │
                        Docker Engine
                             │
                    Docker Compose
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
   ┌─────────┐          ┌───────────┐       ┌─────────┐
   │ Ollama  │          │ OpenWebUI │       │ SearXNG │
   └────┬────┘          └───────────┘       └─────────┘
        │
        ▼
     Modelos
```

La infraestructura actual se ha construido de forma progresiva. Inicialmente el objetivo era disponer de un sistema capaz de ejecutar modelos de lenguaje localmente. Posteriormente se añadieron la interfaz web, las pruebas mediante API y diferentes modelos.

El resultado es una arquitectura que puede ampliarse con nuevos servicios sin modificar necesariamente el núcleo de inferencia.

---

## 3. Capas de la arquitectura

Para documentar correctamente el sistema resulta útil dividirlo en diferentes capas lógicas.

### 3.1. Capa física

Es la capa formada por los recursos hardware sobre los que se ejecuta la plataforma.

Incluye:

- procesador;
- memoria RAM;
- GPU;
- memoria VRAM;
- almacenamiento;
- interfaces de red.

En el laboratorio existe una restricción especialmente importante: actualmente se dispone de una única GPU con **8 GB de VRAM**.

Esta limitación condiciona de forma directa qué modelos pueden ejecutarse completamente en GPU y cuáles deben utilizar memoria de CPU.

Por tanto, la GPU no debe considerarse simplemente como un componente de aceleración. En este proyecto constituye uno de los principales factores que determinan la arquitectura y la selección de modelos.

Los detalles completos del hardware y de la infraestructura de virtualización se documentarán en `03-infraestructura-proxmox.md`. En este documento se destaca únicamente aquello que afecta directamente al diseño de la plataforma de IA.

---

## 4. La limitación de los 8 GB de VRAM

La capacidad disponible de GPU es una de las restricciones fundamentales del laboratorio.

Una GPU con 8 GB de VRAM puede ejecutar modelos relativamente pequeños con una utilización elevada de la GPU, pero no permite asumir que cualquier modelo pueda cargarse completamente en memoria gráfica.

Durante las pruebas se observó claramente este comportamiento.

Por ejemplo, `qwen2.5-coder:7b` llegó a ejecutarse con:

```text
100% GPU
```

mientras que `qwen3:8b` presentó:

```text
12% CPU / 88% GPU
```

Otros modelos de mayor tamaño necesitaron repartir el trabajo entre CPU y GPU. En las pruebas realizadas:

```text
qwen2.5-coder:14b
≈ 36% CPU / 64% GPU

qwen3:14b
≈ 45% CPU / 55% GPU

qwen3:30b
≈ 66% CPU / 34% GPU
```

Esto tiene una consecuencia práctica muy importante:

> El tamaño del modelo no solo afecta a la memoria necesaria para almacenarlo; también puede determinar dónde se ejecutan sus operaciones durante la inferencia.

Cuando un modelo no cabe completamente en VRAM, parte de sus recursos puede gestionarse mediante CPU y memoria RAM. Esto permite ejecutar modelos que superarían la capacidad de la GPU, pero normalmente introduce una penalización importante en el rendimiento.

La situación puede representarse de forma simplificada:

```text
                    MODELO PEQUEÑO
                         │
                         ▼
                  ┌─────────────┐
                  │     GPU     │
                  │   8 GB VRAM │
                  └─────────────┘
                         │
                    alta velocidad


                    MODELO GRANDE
                         │
                         ▼
             ┌──────────────────────┐
             │      CPU + RAM       │
             └──────────┬───────────┘
                        │
                        │ intercambio / offload
                        ▼
             ┌──────────────────────┐
             │        GPU           │
             │      8 GB VRAM       │
             └──────────────────────┘
                         │
                    menor velocidad
```

Este comportamiento explica parte de las diferencias observadas durante las pruebas de rendimiento.

Por ejemplo, el modelo `qwen2.5-coder:7b` produjo alrededor de 53 tokens/s en la prueba de programación, mientras que `qwen2.5-coder:14b` se situó alrededor de 7,6 tokens/s bajo la configuración utilizada.

El salto no se debe únicamente al número de parámetros. También interviene la distribución del modelo entre CPU y GPU.

---

## 5. Tamaño del modelo y recursos disponibles

Una de las conclusiones arquitectónicas obtenidas durante las pruebas es que no existe una relación sencilla entre "modelo más grande" y "modelo mejor para el laboratorio".

El tamaño del modelo tiene que analizarse conjuntamente con:

- capacidad de VRAM;
- memoria RAM;
- cuantización;
- tamaño del contexto;
- utilización de CPU;
- utilización de GPU;
- velocidad de generación;
- calidad de la respuesta;
- tipo de tarea.

Durante las pruebas se manejaron modelos con tamaños de almacenamiento muy diferentes.

Como referencia, se observaron aproximadamente los siguientes tamaños en Ollama:

```text
qwen2.5-coder:7b   → ~5,1–5,6 GB
qwen3.5:9b         → ~5,5 GB
qwen3:8b           → ~7,8 GB
qwen2.5-coder:14b  → ~10–12 GB
qwen3:14b          → ~12 GB
qwen3:30b          → ~20 GB
```

Estos valores no deben interpretarse como equivalentes exactos a la VRAM necesaria durante la ejecución.

El modelo necesita memoria adicional para estructuras de ejecución, contexto, buffers y otros componentes del proceso de inferencia.

Por tanto:

```text
Tamaño del archivo del modelo
          ≠
Memoria total necesaria para inferencia
```

Esta distinción resulta especialmente importante en una GPU limitada a 8 GB.

---

## 6. Capa de virtualización

Proxmox constituye la capa de infraestructura sobre la que se ejecuta el laboratorio.

Su función dentro del proyecto es proporcionar:

- aislamiento;
- gestión de recursos;
- administración del sistema;
- almacenamiento;
- red;
- capacidad de crear y gestionar máquinas virtuales o contenedores;
- posibilidad de ampliar posteriormente la infraestructura.

La utilización de Proxmox permite separar conceptualmente la infraestructura de IA del resto de servicios.

La arquitectura lógica es:

```text
┌──────────────────────────────────────────────┐
│                  Proxmox                     │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │              ai01                      │  │ 
│  │                                        │  │
│  │  Linux                                 │  │
│  │    └── Docker                          │  │
│  │         ├── Ollama                     │  │
│  │         ├── OpenWebUI                  │  │
│  │         └── SearXNG                    │  │
│  └────────────────────────────────────────┘  │
│                                              │
└──────────────────────────────────────────────┘
```

La configuración concreta de Proxmox, los recursos asignados y la forma de proporcionar acceso a la GPU se documentarán en el capítulo específico de infraestructura.

---

## 7. Capa de sistema operativo y contenedores

Sobre el entorno de ejecución se dispone de un sistema Linux que proporciona la base para los servicios de IA.

Docker se utiliza como tecnología de contenerización.

La decisión de utilizar Docker responde principalmente a la necesidad de aislar servicios y simplificar su despliegue.

En lugar de instalar directamente todos los componentes sobre el sistema operativo, se utilizan contenedores independientes:

```text
Sistema Linux
     │
     ▼
 Docker
     │
     ├───────────────┐
     │               │
     ▼               ▼
  Ollama         OpenWebUI
     │
     ▼
  Modelos
```

Esta organización facilita:

- actualización de servicios;
- reinicio independiente;
- gestión de dependencias;
- persistencia mediante volúmenes;
- configuración reproducible;
- ampliación posterior.

La configuración se gestiona mediante Docker Compose.

---

## 8. Capa de inferencia: Ollama

Ollama constituye el núcleo de la plataforma de IA.

Su función principal es proporcionar un entorno para descargar, almacenar y ejecutar modelos de lenguaje.

Además, expone una API HTTP que permite a otras aplicaciones realizar inferencias.

Conceptualmente:

```text
Aplicación
    │
    │ HTTP
    ▼
Ollama API
    │
    ▼
Modelo seleccionado
    │
    ▼
Inferencia
    │
    ▼
Respuesta JSON
```

Esta API es uno de los elementos más importantes de la arquitectura porque permite que Ollama sea utilizado independientemente de OpenWebUI.

Durante las pruebas se utilizó directamente el endpoint de generación para ejecutar modelos y obtener métricas como:

- número de tokens generados;
- duración de la generación;
- tokens por segundo;
- número de tokens del prompt;
- duración total.

Esta posibilidad resulta especialmente útil para realizar pruebas reproducibles y automatizadas.

---

## 9. Capa de modelos

Ollama actúa como gestor de los diferentes modelos disponibles.

Durante la fase de evaluación se instalaron, entre otros:

```text
qwen2.5-coder:7b
qwen2.5-coder:14b
qwen3:8b
qwen3:14b
qwen3:30b
qwen3.5:9b
gemma3:12b
```

La existencia de varios modelos permite adaptar la plataforma a diferentes tareas.

No todos los modelos deben utilizarse para el mismo propósito.

Por ejemplo:

```text
Tarea general
      │
      ├── modelo generalista
      │
      ├── modelo orientado a razonamiento
      │
      └── modelo ligero

Tarea de programación
      │
      └── modelo especializado en código
```

En este proyecto se comprobó además que disponer de un modelo especializado en programación no implica automáticamente que sea adecuado para trabajar como agente autónomo dentro de un entorno real.

Esta cuestión se desarrolla con más detalle en los documentos de pruebas e integración.

---

## 10. Capa de presentación: OpenWebUI

OpenWebUI proporciona inicialmente la interfaz gráfica de usuario.

Su función es ofrecer una forma sencilla de interactuar con los modelos sin tener que utilizar directamente la API de Ollama.

El flujo básico es:

```text
Usuario
   │
   ▼
OpenWebUI
   │
   ▼
Ollama
   │
   ▼
Modelo
   │
   ▼
Respuesta
   │
   ▼
OpenWebUI
   │
   ▼
Usuario
```

OpenWebUI es, por tanto, una capa de aplicación y no el motor que ejecuta los modelos.

Esta separación es importante porque permite mantener la plataforma funcional aunque OpenWebUI sea sustituido en el futuro por otra aplicación.

---

## 11. Capa de búsqueda y servicios auxiliares

La infraestructura incluye también SearXNG como servicio auxiliar para búsquedas.

La presencia de servicios auxiliares permite construir posteriormente flujos en los que la IA pueda combinar información procedente de distintas fuentes.

La arquitectura conceptual podría evolucionar hacia:

```text
                    ┌─────────────┐
                    │    Usuario  │
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │  OpenWebUI  │
                    └──────┬──────┘
                           │
             ┌─────────────┴─────────────┐
             │                           │
             ▼                           ▼
        ┌─────────┐                 ┌─────────┐
        │ Ollama  │                 │ SearXNG │
        └────┬────┘                 └─────────┘
             │
             ▼
          Modelos
```

No obstante, cada integración se incorporará de forma progresiva y se documentará por separado.

---

## 12. API como punto de integración

Uno de los principios fundamentales de la arquitectura es que Ollama no debe quedar limitado a OpenWebUI.

La API permite que otras aplicaciones se conecten directamente al servidor de inferencia.

Esto permite imaginar una arquitectura futura como:

```text
                         ┌─────────────┐
                         │  OpenWebUI  │
                         └──────┬──────┘
                                │
                         ┌──────▼──────┐
                         │             │
                         │   Ollama    │
                         │             │
                         └──────┬──────┘
                                │
             ┌──────────────────┼──────────────────┐
             │                  │                  │
             ▼                  ▼                  ▼
          n8n              Aplicación          Scripts
             │
             ▼
       Automatizaciones
```

Esta característica será especialmente importante cuando se incorpore n8n.

El objetivo será que un flujo automatizado pueda enviar información al modelo, recibir una respuesta y continuar el proceso.

---

## 13. RAG y persistencia

La arquitectura también contempla el uso de RAG.

En este caso aparece una capa adicional encargada de almacenar información que el modelo podrá utilizar durante las consultas.

El esquema general es:

```text
             DOCUMENTOS
                  │
                  ▼
          Procesamiento
                  │
                  ▼
              Chunking
                  │
                  ▼
             Embeddings
                  │
                  ▼
          Base/vector store
                  │
                  │
                  ▼
Consulta ───► Recuperación
                  │
                  ▼
              Contexto
                  │
                  ▼
                Ollama
                  │
                  ▼
               Modelo
                  │
                  ▼
              Respuesta
```

La experimentación con RAG y persistencia ya se realizó anteriormente y forma parte de la experiencia acumulada del laboratorio.

El detalle de estas pruebas se recogerá en `10-rag-y-persistencia.md`.

---

## 14. Persistencia de Ollama

Los modelos no se almacenan únicamente dentro de la capa efímera del contenedor.

La instalación utiliza un volumen Docker dedicado para `/root/.ollama`.

Conceptualmente:

```text
Docker container
┌──────────────────────────┐
│         Ollama           │
│                          │
│      /root/.ollama       │
└────────────┬─────────────┘
             │
             │ volumen
             ▼
┌──────────────────────────┐
│   ollama-data            │
│                          │
│   modelos y datos        │
└──────────────────────────┘
```

Esta separación permite reconstruir o reiniciar el contenedor sin tener que descargar necesariamente todos los modelos de nuevo.

La persistencia constituye, por tanto, una parte importante de la arquitectura y no un detalle secundario de la instalación.

---

## 15. Red y exposición de servicios

La arquitectura utiliza una separación entre los servicios internos de Docker y los servicios que deben ser accesibles desde otros equipos.

Dentro de Docker, los servicios pueden comunicarse utilizando la red interna de Compose.

Por ejemplo:

```text
OpenWebUI ───────► ollama:11434
       │
       └─────────► searxng
```

En cambio, cuando un servicio necesita ser consumido desde el exterior del entorno Docker, se utiliza un puerto publicado.

Durante la configuración de Ollama se realizó inicialmente una publicación restringida a la interfaz local:

```text
127.0.0.1:11434 → 11434/tcp
```

Esta configuración permitió posteriormente acceder al servicio desde el entorno donde se estaba ejecutando el cliente de pruebas.

La exposición de una API de inferencia debe tratarse como una cuestión de seguridad y no simplemente como un problema de conectividad.

No es recomendable exponer directamente el servicio de inferencia a Internet sin controles adicionales.

---

## 16. Arquitectura lógica completa

Uniendo las diferentes capas, la arquitectura actual puede representarse de la siguiente manera:

```text
┌──────────────────────────────────────────────────────────────┐
│                         USUARIOS                             │
└──────────────────────────────┬───────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────┐
│                     APLICACIONES                             │
│                                                              │
│     OpenWebUI        Scripts        OpenCode       n8n*       │
└───────────────┬───────────────┬───────────────┬──────────────┘
                │               │               │
                └───────────────┼───────────────┘
                                │
                           HTTP / API
                                │
                                ▼
┌──────────────────────────────────────────────────────────────┐
│                         OLLAMA                               │
│                                                              │
│  API REST · Gestión de modelos · Inferencia · Contexto       │
└──────────────────────────────┬───────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────┐
│                         MODELOS                              │
│                                                              │
│ Qwen · Gemma · otros modelos disponibles                     │
└──────────────────────────────┬───────────────────────────────┘
                               │
                    CPU / RAM / GPU / VRAM
                               │
                               ▼
┌──────────────────────────────────────────────────────────────┐
│                       AI01 / PROXMOX                         │
│                                                              │
│ Docker · almacenamiento · red · recursos de computación     │
└──────────────────────────────────────────────────────────────┘

* n8n corresponde a una ampliación prevista del proyecto.
```

---

## 17. Arquitectura física frente a arquitectura lógica

Es importante distinguir dos conceptos.

La arquitectura física describe dónde se ejecutan realmente los componentes:

```text
Hardware
   ↓
Proxmox
   ↓
ai01
   ↓
Docker
   ↓
Contenedores
```

La arquitectura lógica describe cómo se relacionan los servicios:

```text
Usuario
   ↓
OpenWebUI
   ↓
Ollama API
   ↓
Modelo
```

Ambas perspectivas son necesarias.

La arquitectura física permite comprender las restricciones de recursos y disponibilidad.

La arquitectura lógica permite comprender los flujos de información y las dependencias entre servicios.

---

## 18. La GPU como recurso compartido

La presencia de una única GPU de 8 GB de VRAM introduce además una restricción arquitectónica adicional: la GPU es un recurso limitado que debe compartirse entre los diferentes modelos.

No resulta práctico asumir que todos los modelos estarán cargados simultáneamente.

Por ello, el comportamiento normal del laboratorio consiste en cargar el modelo necesario para la prueba y detenerlo o sustituirlo posteriormente cuando se necesita utilizar otro modelo.

El propio estado de Ollama permite observar qué modelo está cargado y cómo se distribuye el procesamiento.

Por ejemplo:

```text
NAME              SIZE      PROCESSOR
qwen2.5-coder:7b  5.6 GB    100% GPU

qwen2.5-coder:14b 10 GB     36%/64% CPU/GPU

qwen3:30b         20 GB     66%/34% CPU/GPU
```

Estos datos son especialmente útiles porque permiten relacionar directamente las características del modelo con el comportamiento de la infraestructura.

---

## 19. Contexto como recurso

La VRAM no es la única limitación.

El tamaño del contexto también tiene impacto sobre los recursos necesarios.

Durante las pruebas se utilizaron configuraciones como:

```text
num_ctx = 8192
num_ctx = 16384
```

Aumentar el contexto permite proporcionar más información al modelo, pero también puede incrementar el consumo de memoria.

Por tanto:

```text
Mayor contexto
      ↓
Más información disponible
      ↓
Mayor demanda de recursos
      ↓
Posible aumento del tiempo de procesamiento
```

La configuración del contexto debe adaptarse a la tarea.

Para una consulta sencilla puede no ser necesario utilizar un contexto grande. Para analizar documentos extensos o código puede resultar necesario.

---

## 20. Arquitectura orientada a experimentación

La arquitectura se ha diseñado deliberadamente para facilitar la experimentación.

Los principales factores que favorecen este enfoque son:

- modelos intercambiables;
- configuración centralizada;
- despliegue mediante Docker Compose;
- API REST;
- persistencia;
- acceso desde diferentes clientes;
- posibilidad de monitorizar recursos;
- separación entre servicios.

El ciclo experimental puede representarse como:

```text
              ┌─────────────────┐
              │ Seleccionar      │
              │ modelo           │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │ Configurar      │
              │ parámetros      │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │ Ejecutar prueba │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │ Medir recursos  │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │ Analizar        │
              │ resultados      │
              └────────┬────────┘
                       │
                       └──────► siguiente modelo
```

Esta capacidad de repetir rápidamente las pruebas es uno de los principales objetivos de la infraestructura.

---

## 21. Escalabilidad futura

Aunque el laboratorio actual está condicionado por una única GPU de 8 GB, la arquitectura no se ha diseñado exclusivamente para este hardware.

La separación entre aplicación, inferencia y modelos permite ampliar posteriormente la infraestructura.

Por ejemplo, una futura ampliación podría consistir en incorporar un nodo con una GPU de mayor capacidad:

```text
             Actual
               │
               ▼
       ┌──────────────┐
       │     ai01     │
       │   GPU 8 GB   │
       └──────────────┘


             Futuro
               │
               ▼
       ┌─────────────────────┐
       │ Cluster / servidores│
       ├─────────────────────┤
       │ Nodo IA 1           │
       │ Nodo IA 2           │
       │ GPU de mayor VRAM   │
       └─────────────────────┘
```

No obstante, la ampliación de hardware no constituye actualmente un requisito del proyecto.

Una parte importante del laboratorio consiste precisamente en estudiar hasta dónde puede llegarse con recursos limitados.

---

## 22. Decisiones arquitectónicas principales

Las decisiones más relevantes adoptadas hasta este momento pueden resumirse en los siguientes puntos.

### Ejecución local

Los modelos se ejecutan dentro de la infraestructura propia.

### Ollama como servidor de inferencia

Ollama proporciona una interfaz común para ejecutar los diferentes modelos.

### OpenWebUI como interfaz

OpenWebUI proporciona una capa gráfica independiente del motor de inferencia.

### Docker

Los servicios se despliegan de forma aislada y reproducible.

### Docker Compose

La configuración de los servicios se mantiene centralizada.

### API REST

Las aplicaciones externas pueden comunicarse directamente con Ollama.

### Persistencia

Los datos de Ollama se almacenan mediante un volumen Docker.

### Hardware limitado

La arquitectura se ha diseñado teniendo en cuenta que únicamente existe una GPU con 8 GB de VRAM.

### Evaluación experimental

La selección de modelos se realiza mediante pruebas y no únicamente mediante sus especificaciones teóricas.

---

## 23. Limitaciones arquitectónicas actuales

La plataforma presenta varias limitaciones que deben quedar documentadas.

### 23.1. Capacidad limitada de VRAM

La principal limitación es la GPU de 8 GB.

Esto restringe la ejecución completamente acelerada de modelos grandes.

### 23.2. Penalización por CPU offload

Los modelos que no caben completamente en VRAM utilizan recursos de CPU y RAM, lo que puede reducir significativamente la velocidad de generación.

### 23.3. Una única GPU

La disponibilidad de una sola GPU implica que los modelos compiten por el mismo recurso.

### 23.4. Recursos limitados

El laboratorio está diseñado para experimentación y no para cargas de producción intensivas.

### 23.5. Dependencia de la configuración del modelo

El comportamiento puede variar considerablemente según:

- cuantización;
- contexto;
- temperatura;
- prompt;
- número de tokens generados;
- distribución CPU/GPU.

### 23.6. Seguridad de las integraciones

Cada nueva aplicación que pueda comunicarse con Ollama introduce una nueva superficie de integración que debe controlarse.

---

## 24. Principio de evolución incremental

La arquitectura seguirá un principio de evolución incremental.

No se añadirán numerosos servicios simultáneamente.

La incorporación de cada nuevo componente deberá seguir aproximadamente este proceso:

```text
Nuevo componente
       ↓
Despliegue aislado
       ↓
Prueba funcional
       ↓
Integración
       ↓
Prueba de seguridad
       ↓
Medición
       ↓
Documentación
```

Este enfoque evita que una modificación de la infraestructura produzca demasiadas variables simultáneas y dificulte determinar el origen de un problema.

---

## 25. Arquitectura objetivo

La arquitectura futura pretende evolucionar desde una plataforma centrada en el chatbot hacia una plataforma de servicios de IA.

La evolución prevista es:

```text
                        ┌──────────────┐
                        │   Usuarios   │
                        └───────┬──────┘
                                │
                ┌───────────────┼────────────────┐
                │               │                │
                ▼               ▼                ▼
           OpenWebUI           n8n          Herramientas
                │               │                │
                └───────────────┼────────────────┘
                                │
                                ▼
                         ┌─────────────┐
                         │   Ollama    │
                         └──────┬──────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ▼                 ▼                 ▼
           Modelo A          Modelo B          Modelo C
              │                 │                 │
              └─────────────────┼─────────────────┘
                                │
                                ▼
                       Recursos hardware
```

Esta arquitectura permite que la IA pase de ser una aplicación aislada a convertirse en un servicio reutilizable por diferentes herramientas.

---

## 26. Relación con el resto de la documentación

Este documento proporciona una visión general de la arquitectura.

Los detalles se distribuyen posteriormente de la siguiente forma:

```text
02 - Arquitectura general
          │
          ├── 03 - Infraestructura Proxmox
          │
          ├── 04 - Despliegue de la plataforma IA
          │
          ├── 05 - Ollama
          │
          ├── 06 - OpenWebUI
          │
          ├── 07 - Modelos y recursos
          │
          ├── 08 - Metodología de pruebas
          │
          ├── 09 - Rendimiento y calidad
          │
          ├── 10 - RAG y persistencia
          │
          ├── 11 - Integración con OpenCode
          │
          ├── 12 - Acceso remoto y red
          │
          └── 13/14 - Resultados y evolución
```

De esta forma se evita repetir información y se mantiene una separación clara entre arquitectura, implementación, pruebas y conclusiones.

---

## 27. Conclusión

La arquitectura del laboratorio se basa en una separación clara entre infraestructura, contenedores, servidor de inferencia, modelos y aplicaciones.

El elemento central es Ollama, que proporciona una API común para ejecutar diferentes modelos. OpenWebUI actúa como interfaz de usuario, mientras que Docker y Proxmox proporcionan las capas de infraestructura necesarias para ejecutar y gestionar los servicios.

Una característica especialmente importante de esta arquitectura es que se ha diseñado alrededor de una restricción real: la disponibilidad de una única GPU con 8 GB de VRAM.

Esta limitación condiciona directamente la selección de modelos y explica por qué algunos modelos pueden ejecutarse prácticamente de forma completa en GPU mientras que otros requieren una combinación de CPU y GPU.

La arquitectura, por tanto, no debe entenderse únicamente como un conjunto de software instalado. El hardware disponible forma parte de las decisiones arquitectónicas.

La combinación:

```text
Proxmox
   ↓
Sistema Linux
   ↓
Docker / Compose
   ↓
Ollama
   ↓
Modelos
   ↓
OpenWebUI / APIs / automatizaciones
```

constituye la base sobre la que se desarrollarán las siguientes fases del proyecto.

A partir de esta arquitectura será posible incorporar nuevas aplicaciones, realizar pruebas comparativas y estudiar diferentes usos de la inteligencia artificial sin modificar necesariamente el núcleo de la plataforma.

La siguiente fase de la documentación profundizará en la infraestructura Proxmox y en la configuración concreta del entorno hardware y virtualizado.
