# Local AI Lab

Laboratorio de inteligencia artificial local orientado a la
experimentación, evaluación e integración de modelos de lenguaje en una
infraestructura informática propia.

El objetivo de este proyecto es estudiar, mediante pruebas reales, qué
aplicaciones de la inteligencia artificial pueden aportar valor en un
entorno de informática administrado localmente, evitando depender de
servicios externos cuando no sea necesario.

La plataforma se ejecuta sobre infraestructura propia y utiliza
contenedores Docker para desplegar los principales servicios. Su diseño
es modular para poder incorporar progresivamente nuevos modelos,
aplicaciones, fuentes de información y automatizaciones.

## 1. Objetivos

Los objetivos principales son:

-   Ejecutar modelos de lenguaje localmente mediante Ollama.
-   Proporcionar una interfaz web mediante OpenWebUI.
-   Evaluar distintos modelos atendiendo a calidad, velocidad y consumo
    de recursos.
-   Experimentar con diferentes tamaños de contexto y parámetros de
    inferencia.
-   Estudiar sistemas RAG (Retrieval-Augmented Generation).
-   Comprobar la persistencia de modelos, datos y configuraciones.
-   Exponer la inferencia mediante una API utilizable por otras
    aplicaciones.
-   Evaluar el uso de modelos locales en herramientas de desarrollo y
    agentes, especialmente OpenCode.
-   Determinar qué tareas informáticas pueden automatizarse de forma
    fiable.
-   Investigar nuevas aplicaciones de IA en administración de sistemas,
    automatización y ciberseguridad.
-   Mantener una infraestructura reproducible y una documentación
    técnica completa.

El proyecto no pretende determinar qué modelo es universalmente mejor.
Las conclusiones se corresponden con las pruebas realizadas en este
laboratorio y con los recursos de hardware disponibles.

## 2. Filosofía del laboratorio

El proyecto se plantea como un laboratorio técnico y no únicamente como
una instalación de un chatbot.

Cada aplicación se aborda mediante un ciclo experimental:

1.  Planteamiento del caso de uso.
2.  Instalación y configuración de la infraestructura.
3.  Ejecución de una prueba controlada.
4.  Registro de parámetros.
5.  Observación de resultados.
6.  Análisis de problemas y limitaciones.
7.  Decisión sobre la utilidad del sistema para ese caso.
8.  Documentación.

Los resultados negativos también son parte del proyecto. Que un modelo
genere una respuesta técnicamente correcta no significa que pueda
utilizarse como agente autónomo dentro de un flujo de trabajo real.

## 3. Arquitectura general

La arquitectura inicial está basada en un servidor de IA que ejecuta
varios servicios Docker.

``` text
                         RED LOCAL
                             │
                       ┌─────┴─────┐
                       │  Proxmox  │
                       └─────┬─────┘
                             │
                       ┌─────┴─────┐
                       │   ai01    │
                       │ Servidor  │
                       │    IA     │
                       └─────┬─────┘
                             │
                    ┌────────┴────────┐
                    │ Docker Compose  │
                    │                 │
              ┌─────┴─────┐     ┌────┴─────┐
              │  Ollama   │     │ OpenWebUI│
              │           │     │          │
              │   LLMs    │     │   Web UI │
              └─────┬─────┘     └──────────┘
                    │
                    │ API REST
                    │
              ┌─────┴─────┐
              │  Modelos  │
              │ Qwen etc. │
              └───────────┘

              ┌───────────┐
              │  SearXNG  │
              └───────────┘
```

Ollama constituye la capa de inferencia. Los modelos se almacenan y
ejecutan en el servidor, mientras que OpenWebUI proporciona una interfaz
de interacción.

La separación entre ambas capas es deliberada. Ollama expone una API que
permite que otros sistemas consuman directamente los modelos, sin
depender de OpenWebUI como intermediario.

Esta característica será especialmente importante para futuras
integraciones, como n8n.

## 4. Componentes actuales

### Proxmox

Proxmox proporciona la infraestructura de virtualización sobre la que se
ejecuta el entorno.

El servidor de IA utilizado durante las pruebas se identifica como
`ai01`.

### Docker

Los servicios se ejecutan mediante contenedores Docker y Docker Compose.

El proyecto de infraestructura se encuentra en:

``` text
/opt/local-ai
```

La configuración principal utiliza:

``` text
/opt/local-ai/compose.yaml
```

### Ollama

Ollama es el motor utilizado para ejecutar los modelos de lenguaje
localmente.

El servicio utiliza el puerto `11434` y proporciona una API REST para
consultar modelos, ejecutar inferencias y configurar parámetros como el
contexto.

### OpenWebUI

OpenWebUI proporciona la interfaz web para interactuar con los modelos y
trabajar con funcionalidades relacionadas con RAG y documentos.

### SearXNG

SearXNG forma parte del despliegue actual como componente de búsqueda
local y queda disponible para futuras integraciones y experimentos.

## 5. Modelos evaluados

Durante las distintas fases se han instalado y/o evaluado varios
modelos:

| Modelo | Uso principal |
|---|---|
| `qwen3.5:9b` | Generación general y evaluación de calidad |
| `qwen3:30b` | Evaluación de calidad y comportamiento con mayor tamaño |
| `qwen3:14b` | Análisis técnico y depuración |
| `qwen3:8b` | Análisis técnico y depuración |
| `qwen2.5-coder:14b` | Evaluación de programación |
| `qwen2.5-coder:7b` | Evaluación de programación |
| `gemma3:12b` | Modelo disponible para futuras pruebas |

La disponibilidad de un modelo en el servidor no implica que haya sido
evaluado exhaustivamente.

## 6. Evaluación de modelos

Uno de los objetivos principales ha sido comparar modelos mediante
pruebas reproducibles.

Las pruebas han registrado métricas proporcionadas por la API de Ollama,
entre ellas:

-   tokens generados;
-   tiempo de generación;
-   tokens por segundo;
-   tokens del prompt;
-   tiempo total;
-   tamaño del contexto;
-   utilización de CPU y GPU;
-   calidad y precisión de la respuesta;
-   comportamiento ante tareas técnicas.

También se ha estudiado el efecto de modificar `num_ctx`, especialmente
con modelos de mayor tamaño.

Por ejemplo, `qwen3:30b` se probó inicialmente con un contexto de 4096
tokens y posteriormente con 16384 tokens. El incremento del contexto
produjo un aumento apreciable del tiempo de ejecución y una mayor
participación de la CPU.

Los resultados completos se documentan en:

-   [Metodología de pruebas](docs/08-metodologia-de-pruebas.md)
-   [Pruebas de rendimiento y
    calidad](docs/09-pruebas-de-rendimiento-y-calidad.md)
-   [Modelos y recursos](docs/07-modelos-y-recursos.md)

## 7. RAG y persistencia

La plataforma también se ha utilizado para experimentar con RAG
(Retrieval-Augmented Generation).

El flujo conceptual es:

``` text
Documento
   │
   ▼
Fragmentación
   │
   ▼
Embeddings
   │
   ▼
Almacenamiento vectorial
   │
   ▼
Consulta del usuario
   │
   ▼
Recuperación de información
   │
   ▼
Contexto + consulta
   │
   ▼
LLM
   │
   ▼
Respuesta
```

Las pruebas realizadas previamente demostraron un funcionamiento
adecuado de RAG y de la persistencia para el escenario estudiado.

Esta parte se documenta en
[docs/10-rag-y-persistencia.md](docs/10-rag-y-persistencia.md).

## 8. Integración con otras aplicaciones

Una característica fundamental de la arquitectura es que Ollama no está
limitado a OpenWebUI.

Al disponer de una API REST, otros programas pueden enviar peticiones
directamente al servidor de inferencia:

``` text
Aplicación
    │
    │ HTTP
    ▼
Ollama API
    │
    ▼
Modelo local
```

Esto permite integrar la IA con herramientas de automatización,
administración, análisis y otros servicios.

Entre las integraciones estudiadas se encuentra OpenCode.

## 9. Prueba con OpenCode

Se evaluó la posibilidad de utilizar los modelos locales como agentes de
programación mediante OpenCode.

La prueba no se limitó a solicitar código. Se intentó comprobar la
capacidad de completar una tarea utilizando herramientas:

``` text
Crear archivo
     ↓
Comprobar archivo
     ↓
Compilar
     ↓
Analizar errores
     ↓
Corregir
     ↓
Volver a compilar
     ↓
Ejecutar
     ↓
Comprobar salida
```

Durante las pruebas aparecieron problemas relacionados con:

-   utilización de rutas ficticias;
-   uso de directorios que no correspondían al workspace real;
-   errores derivados de permisos;
-   respuestas explicativas en lugar de ejecución efectiva;
-   comportamiento inconsistente en el uso de herramientas;
-   intentos de delegación en subagentes;
-   dificultad para completar de forma fiable una tarea sencilla de
    extremo a extremo.

La prueba permitió diferenciar entre dos capacidades:

``` text
Generar código
       ↓
     viable

Actuar como agente autónomo sobre un entorno real
       ↓
     no fiable en las pruebas realizadas
```

Dentro del alcance del laboratorio, los modelos evaluados no se
consideran actualmente adecuados para sustituir un flujo de desarrollo
tradicional mediante agentes autónomos de programación.

El análisis detallado se encuentra en
[docs/11-integracion-con-opencode.md](docs/11-integracion-con-opencode.md).

## 10. Acceso desde otros equipos

La infraestructura permite consumir la API de Ollama desde el equipo
personal utilizado para administrar y probar el laboratorio.

Una comprobación habitual de disponibilidad es:

``` bash
curl -s http://127.0.0.1:11434/api/tags
```

La configuración de Docker, el puerto utilizado y las comprobaciones
realizadas se documentan en
[docs/12-acceso-remoto-y-red.md](docs/12-acceso-remoto-y-red.md).

El acceso a la API se mantiene controlado. El objetivo es proporcionar
acceso a los servicios necesarios sin convertir el servidor de
inferencia en un servicio público expuesto directamente a Internet.

## 11. Estado actual

| Área | Estado | Observaciones |
|---|---|---|
| Infraestructura Proxmox | Implementado | Servidor de IA operativo |
| Docker | Implementado | Servicios desplegados mediante Compose |
| Ollama | Implementado | Inferencia local funcionando |
| OpenWebUI | Implementado | Interfaz web operativa |
| Modelos locales | Implementado | Varios modelos disponibles |
| Evaluación de modelos | Probado | Se han realizado pruebas comparativas |
| RAG | Probado | Funcionamiento correcto en las pruebas realizadas |
| Persistencia | Probado | Funcionamiento correcto en las pruebas realizadas |
| API de Ollama | Implementado | Utilizable por aplicaciones externas |
| OpenCode | Probado | Integración experimental no fiable como agente |
| n8n | Pendiente | Próxima línea de experimentación |
| IA aplicada a OPNsense | Pendiente | Posible integración futura |
| Automatización mediante IA | Pendiente | Se estudiará principalmente con n8n |
| Nuevas aplicaciones de IA | Pendiente | Parte del roadmap |

## 12. Documentación

### Fundamentos y arquitectura

-   [01. Introducción y objetivos](docs/01-introduccion-y-objetivos.md)
-   [02. Arquitectura general](docs/02-arquitectura-general.md)
-   [03. Infraestructura Proxmox](docs/03-infraestructura-proxmox.md)
-   [04. Despliegue de la plataforma de
    IA](docs/04-despliegue-de-la-plataforma-ia.md)

### Plataforma

-   [05. Ollama](docs/05-ollama.md)
-   [06. OpenWebUI](docs/06-openwebui.md)
-   [07. Modelos y recursos](docs/07-modelos-y-recursos.md)

### Experimentación

-   [08. Metodología de pruebas](docs/08-metodologia-de-pruebas.md)
-   [09. Pruebas de rendimiento y
    calidad](docs/09-pruebas-de-rendimiento-y-calidad.md)
-   [10. RAG y persistencia](docs/10-rag-y-persistencia.md)

### Integración y evolución

-   [11. Integración con OpenCode](docs/11-integracion-con-opencode.md)
-   [12. Acceso remoto y red](docs/12-acceso-remoto-y-red.md)
-   [13. Resultados y
    conclusiones](docs/13-resultados-y-conclusiones.md)
-   [14. Líneas futuras](docs/14-lineas-futuras.md)

## 13. Próximas líneas de trabajo

El siguiente paso será estudiar aplicaciones de IA distintas del uso
convencional de un chatbot.

La primera tecnología prevista es **n8n**, utilizándola como plataforma
de automatización y como punto de integración entre diferentes
servicios.

Una arquitectura inicial prevista podría ser:

``` text
             ┌──────────────┐
             │     n8n      │
             │ Automatizador│
             └──────┬───────┘
                    │
          ┌─────────┴─────────┐
          │                   │
          ▼                   ▼
      Ollama               Servicios
          │                   │
          ▼                   ▼
      Modelo IA        APIs / sistemas
```

Una línea de experimentación será estudiar la interacción entre n8n,
Ollama y servicios de infraestructura ya existentes.

En ciberseguridad, OPNsense constituye actualmente una fuente potencial
de información para futuras pruebas. El objetivo inicial será estudiar
flujos de lectura, análisis, clasificación y generación de informes
antes de permitir acciones automáticas sobre infraestructura.

## 14. Principios de seguridad

El hecho de ejecutar los modelos localmente no significa que el sistema
sea automáticamente seguro.

Se tendrán en cuenta los siguientes principios:

-   No exponer innecesariamente Ollama a Internet.
-   Limitar el acceso a las APIs a las redes y equipos que realmente lo
    necesiten.
-   Separar servicios mediante redes Docker cuando sea apropiado.
-   Evitar introducir credenciales o secretos en prompts.
-   No proporcionar a un agente permisos de escritura sobre sistemas
    críticos sin una necesidad concreta.
-   Comenzar las integraciones con operaciones de solo lectura.
-   Revisar manualmente las acciones generadas por agentes antes de
    permitir cambios sobre infraestructura.
-   Mantener copias de seguridad de configuraciones y datos relevantes.
-   Documentar cualquier modificación de red o infraestructura.
-   Diferenciar una prueba experimental de un servicio preparado para
    producción.

La automatización mediante IA se introducirá progresivamente, aumentando
los permisos únicamente cuando el comportamiento haya sido
suficientemente validado.

## 15. Estructura del repositorio

``` text
.
├── README.md
└── docs/
    ├── 01-introduccion-y-objetivos.md
    ├── 02-arquitectura-general.md
    ├── 03-infraestructura-proxmox.md
    ├── 04-despliegue-de-la-plataforma-ia.md
    ├── 05-ollama.md
    ├── 06-openwebui.md
    ├── 07-modelos-y-recursos.md
    ├── 08-metodologia-de-pruebas.md
    ├── 09-pruebas-de-rendimiento-y-calidad.md
    ├── 10-rag-y-persistencia.md
    ├── 11-integracion-con-opencode.md
    ├── 12-acceso-remoto-y-red.md
    ├── 13-resultados-y-conclusiones.md
    └── 14-lineas-futuras.md
```

El README proporciona una visión general del proyecto. Los documentos de
`docs/` contienen el desarrollo técnico detallado.

## 16. Criterio de documentación

Toda nueva funcionalidad incorporada seguirá, siempre que sea posible,
este ciclo:

``` text
Planificación
     ↓
Implementación
     ↓
Prueba
     ↓
Medición
     ↓
Análisis
     ↓
Decisión
     ↓
Documentación
```

La documentación distinguirá entre funcionalidades implementadas,
funcionalidades probadas experimentalmente y funcionalidades previstas.

Esto permitirá conservar un historial técnico fiable del laboratorio y
relacionar las conclusiones con experimentos concretos.

## 17. Conclusión

Este proyecto constituye un laboratorio para estudiar la aplicación
práctica de modelos de inteligencia artificial en una infraestructura
informática local.

La primera fase se ha centrado en construir la plataforma básica de
inferencia y comprobar sus capacidades mediante Ollama, OpenWebUI,
diferentes modelos de lenguaje, RAG y persistencia.

Las pruebas realizadas muestran que la utilidad de un modelo depende
considerablemente de la tarea. La generación conversacional y el
análisis técnico pueden ofrecer resultados útiles, mientras que la
utilización de los modelos como agentes autónomos que interactúan
directamente con un entorno de desarrollo requiere un nivel de
fiabilidad que no se ha alcanzado en las pruebas realizadas con
OpenCode.

El proyecto continuará evolucionando hacia aplicaciones orientadas a
automatización y administración de sistemas. La incorporación de n8n
constituye el siguiente paso, con el objetivo de estudiar cómo combinar
modelos locales con servicios y fuentes de información reales de una
infraestructura informática.

El propósito final no es únicamente disponer de una IA funcionando
localmente, sino comprender qué puede hacer de forma fiable, cuáles son
sus limitaciones y cómo integrarla de manera controlada dentro de una
infraestructura real.
