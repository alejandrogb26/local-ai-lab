# 01. Introducción y objetivos

## 1. Introducción

La inteligencia artificial ha pasado de ser una tecnología
principalmente experimental a convertirse en una herramienta que puede
integrarse en numerosos ámbitos de la informática: generación y análisis
de texto, búsqueda semántica, asistencia técnica, automatización,
clasificación de información, análisis de registros, generación de
documentación y apoyo a la administración de sistemas, entre otros.

Una parte importante de las soluciones actuales de inteligencia
artificial se consume como un servicio externo. En este modelo, el
usuario o la aplicación envía información a una plataforma de terceros,
donde se ejecuta el modelo, y recibe posteriormente el resultado.

Aunque este enfoque resulta muy accesible, presenta una serie de
condicionantes cuando se pretende utilizar inteligencia artificial de
forma continuada dentro de una infraestructura informática propia. Entre
ellos se encuentran la dependencia de servicios externos, el tratamiento
de información fuera de la infraestructura local, la conectividad
necesaria, las políticas de uso de cada proveedor, los costes asociados
al consumo y la dificultad para controlar completamente el entorno de
ejecución.

Este proyecto parte de una alternativa diferente: construir un
laboratorio de inteligencia artificial local en el que los modelos se
ejecuten dentro de una infraestructura propia y puedan ser consumidos
tanto por usuarios como por otras aplicaciones.

El objetivo no es únicamente instalar un modelo de lenguaje y utilizarlo
como chatbot. Se pretende construir una plataforma sobre la que sea
posible experimentar con diferentes modelos, medir su comportamiento,
estudiar sus limitaciones e integrarlos progresivamente con otros
servicios informáticos.

La plataforma se ha desarrollado sobre infraestructura de virtualización
propia y utiliza Docker para desplegar los diferentes componentes.
Ollama actúa como motor de inferencia y OpenWebUI proporciona
inicialmente la interfaz principal de interacción. Sobre esta base se
han realizado pruebas con diferentes modelos y se han estudiado
funcionalidades como RAG, persistencia, acceso mediante API e
integración con herramientas externas.

Posteriormente, el proyecto se orienta hacia aplicaciones más amplias de
la inteligencia artificial dentro de la informática. En particular, se
plantea experimentar con automatización mediante n8n y estudiar posibles
integraciones con servicios de infraestructura y seguridad.

Por tanto, el proyecto debe entenderse como un laboratorio tecnológico
en evolución. No se pretende construir desde el principio una plataforma
cerrada y definitiva, sino establecer una base técnica que permita
incorporar nuevas capacidades y evaluar de forma objetiva su utilidad.

------------------------------------------------------------------------

## 2. Motivación

La decisión de construir una plataforma de IA local responde a varias
motivaciones técnicas.

### 2.1. Conocer el funcionamiento de los modelos locales

Utilizar un servicio de IA como usuario permite aprovechar sus
capacidades, pero oculta gran parte de la infraestructura necesaria para
ejecutar un modelo.

En un entorno local es posible observar y controlar directamente
aspectos como:

-   almacenamiento de los modelos;
-   consumo de memoria;
-   utilización de CPU;
-   utilización de GPU;
-   tamaño del contexto;
-   velocidad de generación;
-   tiempos de respuesta;
-   configuración del servidor de inferencia;
-   persistencia de los datos;
-   comunicación mediante API;
-   integración con otros servicios.

Esto convierte la plataforma en una herramienta de aprendizaje además de
una herramienta de productividad.

### 2.2. Evaluar la utilidad real de la IA

La calidad de un modelo no puede determinarse únicamente observando una
respuesta aislada.

Un modelo puede producir respuestas excelentes en determinadas tareas y,
al mismo tiempo, presentar limitaciones importantes en otras. La
evaluación debe considerar tanto la calidad de la respuesta como los
recursos necesarios para obtenerla y la fiabilidad del proceso completo.

Por este motivo, el proyecto utiliza pruebas prácticas y comparativas.

La pregunta principal no es:

> ¿Cuál es el mejor modelo?

Sino:

> ¿Qué modelo y qué configuración resultan adecuados para una
> determinada tarea dentro de los recursos disponibles?

Esta diferencia es importante porque un modelo de mayor tamaño puede
producir respuestas más elaboradas, pero también puede requerir
considerablemente más memoria y tiempo de procesamiento.

### 2.3. Experimentar con IA como componente de infraestructura

Otro objetivo fundamental es dejar de considerar la IA exclusivamente
como una aplicación independiente.

En una infraestructura informática, un modelo puede convertirse en un
servicio consumido por otras aplicaciones:

``` text
Aplicación
    │
    │ HTTP / API
    ▼
Servidor de inferencia
    │
    ▼
Modelo de IA
```

Este enfoque permite estudiar la IA como un componente más de la
infraestructura, de forma similar a como se utilizan otros servicios
mediante APIs.

Esto abre la puerta a integrar modelos locales con automatizadores,
sistemas de monitorización, herramientas de administración, sistemas de
documentación o plataformas de análisis.

### 2.4. Control de la infraestructura

Una instalación local proporciona un mayor control sobre el entorno de
ejecución.

En este proyecto, los modelos se almacenan en la propia infraestructura
y el servidor de inferencia se ejecuta localmente. Esto permite decidir
cómo se exponen los servicios, qué aplicaciones pueden acceder a ellos y
qué información se envía a los modelos.

Este control no debe confundirse con una garantía automática de
seguridad o privacidad. Una infraestructura local también debe
configurarse y protegerse correctamente.

Por ello, la seguridad de las integraciones será una consideración
transversal del proyecto.

### 2.5. Estudiar los límites de la tecnología

El proyecto no está planteado únicamente para demostrar las capacidades
de la IA.

También se pretende identificar sus limitaciones.

Esta cuestión resultó especialmente relevante durante las pruebas
realizadas con herramientas de agentes de programación. La generación de
código y la capacidad de actuar de forma autónoma sobre un entorno real
son problemas diferentes.

Un modelo puede generar una implementación aparentemente correcta y, sin
embargo, no ser capaz de ejecutar de forma fiable una secuencia de
operaciones sobre un sistema real.

Esta diferencia entre generación y ejecución es una de las ideas
fundamentales que se pretende conservar a lo largo del proyecto:

``` text
Generar una respuesta
        ≠
Ejecutar una tarea de forma fiable
```

Por ello, las pruebas negativas o los experimentos que no cumplen los
objetivos también se consideran resultados válidos.

------------------------------------------------------------------------

## 3. Propósito general

El propósito general del proyecto es diseñar, desplegar y estudiar una
plataforma de inteligencia artificial local que permita ejecutar modelos
de lenguaje, evaluar su comportamiento e integrarlos progresivamente con
otros servicios informáticos.

La plataforma debe servir simultáneamente para tres finalidades:

1.  **Experimentación:** probar modelos, configuraciones y aplicaciones.
2.  **Aprendizaje:** comprender los componentes y procesos necesarios
    para utilizar IA local.
3.  **Integración:** estudiar cómo convertir los modelos en servicios
    utilizables por otras aplicaciones.

Estas tres finalidades están relacionadas.

La experimentación permite obtener conocimiento sobre los modelos; ese
conocimiento permite determinar en qué tareas pueden resultar útiles; y
la integración permite comprobar si esa utilidad se mantiene cuando la
IA se incorpora a un flujo de trabajo real.

------------------------------------------------------------------------

## 4. Objetivo general

El objetivo general del proyecto es:

> Construir y documentar un laboratorio de inteligencia artificial local
> basado en modelos de lenguaje ejecutados sobre infraestructura propia,
> evaluando su rendimiento, calidad, persistencia e integración con
> aplicaciones y servicios informáticos.

Este objetivo se divide en varios objetivos específicos.

------------------------------------------------------------------------

## 5. Objetivos específicos

### 5.1. Desplegar una infraestructura de IA local

Construir una plataforma capaz de ejecutar modelos de lenguaje
localmente mediante un servidor de inferencia.

La infraestructura debe permitir:

-   instalar modelos;
-   eliminarlos o sustituirlos;
-   iniciar y detener modelos;
-   consultar los modelos disponibles;
-   ejecutar inferencias;
-   modificar parámetros de ejecución;
-   monitorizar el consumo de recursos;
-   acceder a los modelos mediante API.

### 5.2. Integrar una interfaz de usuario

Desplegar OpenWebUI como interfaz para interactuar con los modelos.

La interfaz debe facilitar la experimentación sin necesidad de realizar
manualmente todas las peticiones HTTP contra la API de Ollama.

Además, permitirá estudiar funcionalidades adicionales relacionadas con
documentos y RAG.

### 5.3. Evaluar diferentes modelos

Instalar y probar diferentes modelos con características y tamaños
distintos.

Entre los modelos evaluados durante las pruebas se encuentran:

-   `qwen3:30b`;
-   `qwen3:14b`;
-   `qwen3:8b`;
-   `qwen3.5:9b`;
-   `qwen2.5-coder:14b`;
-   `qwen2.5-coder:7b`;
-   `gemma3:12b`.

La comparación se realizará considerando tanto métricas cuantitativas
como aspectos cualitativos.

### 5.4. Medir el rendimiento

Registrar métricas que permitan relacionar las características del
modelo con el rendimiento obtenido.

Entre las métricas de interés se encuentran:

-   tokens generados;
-   tokens por segundo;
-   tiempo de generación;
-   tiempo total de la petición;
-   tokens procesados en el prompt;
-   tamaño del contexto;
-   utilización de CPU;
-   utilización de GPU.

Estas métricas permiten estudiar el compromiso entre tamaño del modelo,
calidad y rendimiento.

### 5.5. Estudiar el tamaño de contexto

Analizar cómo afecta `num_ctx` al comportamiento de los modelos.

El contexto es especialmente importante en tareas donde es necesario
proporcionar una cantidad elevada de información al modelo.

Sin embargo, aumentar el contexto también puede incrementar el consumo
de recursos y el tiempo necesario para procesar una petición.

Por este motivo, se estudiará el contexto como un parámetro de
rendimiento y no únicamente como una característica funcional.

### 5.6. Experimentar con RAG

Estudiar el uso de Retrieval-Augmented Generation para complementar la
información generada por el modelo con información procedente de
documentos.

El objetivo es comprender el flujo completo:

``` text
Documentos
    ↓
Procesamiento
    ↓
Fragmentación
    ↓
Embeddings
    ↓
Almacenamiento
    ↓
Consulta
    ↓
Recuperación
    ↓
Contexto
    ↓
Modelo
    ↓
Respuesta
```

### 5.7. Comprobar la persistencia

Verificar que los modelos, configuraciones y datos necesarios sobreviven
a reinicios de contenedores y del entorno de ejecución.

La persistencia es especialmente importante en una infraestructura que
debe mantenerse operativa durante periodos prolongados.

El objetivo no es únicamente comprobar que el modelo funciona, sino que
el estado necesario para volver a utilizarlo se conserva correctamente.

### 5.8. Exponer la IA mediante API

Utilizar la API REST de Ollama para comprobar que la inferencia puede
ser consumida por herramientas externas.

Este objetivo es fundamental para las fases posteriores del proyecto, ya
que permite desacoplar la interfaz de usuario del motor de inferencia.

La arquitectura resultante permite:

``` text
                 ┌──────────────┐
                 │  OpenWebUI   │
                 └──────┬───────┘
                        │
                        ▼
                    ┌───────┐
                    │Ollama │
                    └───────┘
                        ▲
                        │
                 ┌──────┴──────┐
                 │             │
               n8n          Otras apps
```

### 5.9. Evaluar agentes y herramientas externas

Estudiar la integración de los modelos con aplicaciones que permitan
utilizar la IA como parte de un flujo de trabajo más complejo.

OpenCode se utilizó como primer caso experimental de este tipo.

El objetivo de esta prueba no era únicamente medir la capacidad de
generación de código, sino comprobar si el modelo podía:

-   interpretar una tarea;
-   utilizar herramientas;
-   interactuar con archivos;
-   trabajar dentro de un workspace;
-   ejecutar comandos;
-   analizar errores;
-   corregirlos;
-   completar la tarea de extremo a extremo.

Las pruebas mostraron limitaciones importantes en este escenario.

Este resultado forma parte de las conclusiones del proyecto y evita
asumir que una buena capacidad conversacional implica necesariamente una
capacidad equivalente como agente autónomo.

### 5.10. Estudiar automatización mediante IA

Una de las siguientes líneas de trabajo será integrar n8n con Ollama.

El objetivo será estudiar flujos en los que un modelo local pueda
participar en procesos automáticos de:

-   clasificación;
-   extracción de información;
-   generación de texto;
-   resumen;
-   análisis;
-   transformación de datos;
-   generación de informes;
-   toma de decisiones acotadas dentro de un flujo controlado.

La automatización se desarrollará inicialmente con operaciones de bajo
riesgo y preferentemente de solo lectura.

### 5.11. Explorar aplicaciones relacionadas con informática y ciberseguridad

El proyecto pretende extender progresivamente la IA a otras áreas de la
informática.

Actualmente OPNsense constituye uno de los elementos de infraestructura
disponibles para futuras pruebas relacionadas con seguridad.

La intención inicial será utilizar la IA para tareas como:

-   análisis de información;
-   clasificación de eventos;
-   resumen de registros;
-   generación de informes;
-   explicación de alertas;
-   búsqueda de patrones;
-   asistencia al administrador.

No se plantea inicialmente permitir que un modelo modifique
automáticamente la configuración de sistemas críticos.

### 5.12. Documentar el proceso

Documentar tanto la configuración como los experimentos realizados.

La documentación debe permitir reconstruir el proceso y entender:

-   qué se instaló;
-   cómo se configuró;
-   qué modelos se utilizaron;
-   qué pruebas se realizaron;
-   qué parámetros se utilizaron;
-   qué resultados se obtuvieron;
-   qué problemas aparecieron;
-   qué decisiones se tomaron;
-   qué funcionalidades quedaron pendientes.

La documentación se mantendrá en Markdown dentro del repositorio GitHub.

------------------------------------------------------------------------

## 6. Alcance del proyecto

El proyecto se centra principalmente en modelos de lenguaje y en su
integración con una infraestructura informática local.

Dentro del alcance se encuentran:

-   infraestructura de virtualización;
-   contenedores Docker;
-   servidor de inferencia;
-   modelos de lenguaje;
-   interfaces web;
-   APIs;
-   RAG;
-   persistencia;
-   evaluación de rendimiento;
-   automatización;
-   integración con herramientas externas;
-   experimentación con administración de sistemas;
-   posibles aplicaciones relacionadas con seguridad.

El proyecto no pretende desarrollar desde cero modelos de lenguaje ni
realizar entrenamiento masivo de modelos.

Tampoco se pretende convertir inmediatamente el laboratorio en una
plataforma de producción crítica.

La prioridad inicial es disponer de una infraestructura experimental
estable y suficientemente controlada para realizar pruebas
reproducibles.

------------------------------------------------------------------------

## 7. Fuera de alcance inicial

Quedan fuera del alcance inicial, salvo que posteriormente se decida
incorporarlos como nuevas líneas de trabajo:

-   entrenamiento completo de modelos de lenguaje de gran tamaño;
-   desarrollo de un LLM desde cero;
-   exposición pública de los modelos a Internet;
-   automatización sin supervisión de cambios críticos;
-   sustitución completa de herramientas de administración existentes;
-   utilización de IA como único mecanismo de decisión en sistemas
    críticos;
-   despliegue de agentes con privilegios administrativos sin controles
    adicionales.

Estos elementos podrían estudiarse posteriormente, pero requerirían
criterios de seguridad, validación y control mucho más estrictos.

------------------------------------------------------------------------

## 8. Criterios para considerar útil una solución

La utilidad de una determinada aplicación de IA no se determinará
únicamente por la calidad de la respuesta.

Se considerarán al menos cinco dimensiones:

### Calidad

La respuesta debe ser correcta, relevante y suficientemente completa
para la tarea.

### Rendimiento

El tiempo de respuesta debe ser razonable para el caso de uso.

### Consumo de recursos

El modelo debe poder ejecutarse con los recursos disponibles sin
comprometer innecesariamente el resto de servicios.

### Fiabilidad

Una tarea repetida en condiciones equivalentes debe producir resultados
suficientemente consistentes.

### Integrabilidad

La solución debe poder incorporarse al flujo de trabajo mediante
mecanismos razonables, como APIs, automatizadores o interfaces
existentes.

Estas dimensiones pueden entrar en conflicto.

Por ejemplo:

``` text
Modelo mayor
    ↓
Posible aumento de calidad
    ↓
Mayor consumo
    ↓
Menor velocidad
```

Por tanto, la selección de una solución debe realizarse teniendo en
cuenta el caso de uso concreto.

------------------------------------------------------------------------

## 9. Metodología general

El proyecto seguirá una metodología experimental.

Cada nueva tecnología o aplicación se estudiará mediante las siguientes
fases:

### Fase 1. Definición

Se define qué problema se pretende resolver y qué resultado se considera
satisfactorio.

### Fase 2. Despliegue

Se instala el componente necesario y se integra con la infraestructura
existente.

### Fase 3. Prueba básica

Se verifica que el servicio funciona correctamente en condiciones
normales.

### Fase 4. Prueba controlada

Se ejecutan tareas definidas previamente para poder comparar resultados.

### Fase 5. Medición

Se registran métricas de rendimiento y consumo cuando sea posible.

### Fase 6. Análisis

Se analizan tanto los resultados correctos como los errores.

### Fase 7. Evaluación

Se determina si la tecnología resulta adecuada para el caso de uso.

### Fase 8. Documentación

Se registra el procedimiento, los resultados y las conclusiones.

El flujo completo puede representarse como:

``` text
Problema
   ↓
Objetivo
   ↓
Implementación
   ↓
Prueba
   ↓
Medición
   ↓
Análisis
   ↓
Evaluación
   ↓
Documentación
```

------------------------------------------------------------------------

## 10. Importancia de las pruebas negativas

Una característica importante de la metodología es que los errores no se
consideran simplemente fallos que deben ocultarse.

Si una aplicación no funciona como se esperaba, ese resultado
proporciona información sobre la adecuación de la tecnología.

Por ejemplo, durante la evaluación de agentes de programación se
comprobó que una tarea aparentemente sencilla podía presentar problemas
cuando el sistema tenía que interactuar con el entorno real.

Esto llevó a una distinción importante:

``` text
Capacidad del modelo
        +
Capacidad de las herramientas
        +
Capacidad del agente para utilizarlas
        +
Permisos del entorno
        +
Fiabilidad del flujo
        =
Resultado real
```

Por tanto, una evaluación real debe considerar el sistema completo y no
solamente la respuesta textual del modelo.

------------------------------------------------------------------------

## 11. Reproducibilidad

Siempre que sea posible, las pruebas se realizarán utilizando:

-   el mismo prompt;
-   el mismo modelo;
-   la misma configuración;
-   el mismo tamaño de contexto;
-   condiciones de ejecución comparables;
-   las mismas métricas.

Los comandos utilizados para las pruebas se conservarán o documentarán
en los documentos correspondientes.

Esto permitirá repetir posteriormente un experimento y comprobar si los
resultados se mantienen.

La reproducibilidad es especialmente importante cuando se comparan
modelos de diferente tamaño.

------------------------------------------------------------------------

## 12. Evolución prevista

El proyecto se plantea por etapas.

### Etapa inicial: plataforma de IA

Construcción de la infraestructura básica:

``` text
Proxmox
   ↓
Docker
   ↓
Ollama
   ↓
Modelos
   ↓
OpenWebUI
```

### Etapa de evaluación

Comparación de modelos y configuraciones:

``` text
Modelos
   ↓
Pruebas
   ↓
Métricas
   ↓
Análisis
```

### Etapa de conocimiento

Experimentación con:

-   RAG;
-   persistencia;
-   embeddings;
-   contexto;
-   APIs.

### Etapa de integración

Conexión con aplicaciones externas:

``` text
Aplicaciones
     ↓
     API
     ↓
   Ollama
     ↓
   Modelos
```

### Etapa de automatización

Incorporación de n8n:

``` text
Eventos / Datos
       ↓
      n8n
       ↓
    Ollama
       ↓
   Resultado
       ↓
   Acción / Informe
```

### Etapa de expansión

Exploración de aplicaciones en:

-   administración de sistemas;
-   análisis de información;
-   monitorización;
-   documentación;
-   ciberseguridad;
-   automatización.

------------------------------------------------------------------------

## 13. Seguridad como requisito transversal

La seguridad no se considera una fase posterior, sino una condición que
debe acompañar a cada integración.

La introducción de IA en una infraestructura puede ampliar las
capacidades del sistema, pero también puede ampliar su superficie de
riesgo.

Un agente que únicamente genera texto tiene unas consecuencias muy
diferentes de un agente que puede:

``` text
leer archivos
     ↓
ejecutar comandos
     ↓
modificar configuraciones
     ↓
reiniciar servicios
     ↓
cambiar reglas de red
```

Por ello, el proyecto seguirá inicialmente un principio de mínimo
privilegio.

Siempre que sea posible, una nueva integración comenzará con acceso de
solo lectura y posteriormente se evaluará si existe una justificación
para permitir operaciones adicionales.

------------------------------------------------------------------------

## 14. Resultado esperado

Al finalizar las diferentes fases, el proyecto debe proporcionar una
visión práctica de las posibilidades y limitaciones de la IA local
dentro de una infraestructura informática.

El resultado esperado no es únicamente una colección de servicios
instalados.

Se pretende disponer de:

-   una infraestructura funcional;
-   una metodología de evaluación;
-   resultados reproducibles;
-   documentación técnica;
-   criterios para seleccionar modelos;
-   ejemplos de integración;
-   automatizaciones experimentales;
-   identificación de limitaciones;
-   conclusiones basadas en pruebas reales.

De esta forma, el laboratorio podrá utilizarse como base para nuevas
experiencias sin tener que reconstruir desde cero la infraestructura o
repetir las pruebas fundamentales.

------------------------------------------------------------------------

## 15. Organización de la documentación

La documentación del proyecto se divide en varios bloques.

### Arquitectura e infraestructura

Documentan dónde se ejecutan los servicios y cómo se relacionan entre
sí.

### Plataforma de IA

Explican la configuración y utilización de Ollama y OpenWebUI.

### Modelos y evaluación

Recogen los modelos utilizados, los recursos disponibles y los
resultados de las pruebas.

### Integración

Documentan las conexiones con herramientas externas y el acceso a la
API.

### Resultados

Recogen las conclusiones obtenidas a partir de los experimentos.

### Evolución futura

Describe las aplicaciones que se pretenden estudiar posteriormente.

El documento actual establece el contexto y los objetivos. Los detalles
de implementación se desarrollarán en los documentos siguientes.

------------------------------------------------------------------------

## 16. Conclusión

Este proyecto nace con una finalidad fundamentalmente práctica: estudiar
hasta qué punto la inteligencia artificial puede integrarse de forma
útil, controlada y reproducible dentro de una infraestructura
informática local.

La ejecución local de modelos permite experimentar directamente con
aspectos que normalmente quedan ocultos cuando se utiliza una plataforma
externa. Al mismo tiempo, obliga a afrontar problemas reales de
infraestructura, recursos, rendimiento, integración, seguridad y
mantenimiento.

El proyecto no parte de la premisa de que la IA pueda sustituir
automáticamente las herramientas o procesos existentes. Por el
contrario, cada aplicación será evaluada según el valor que aporte en un
escenario concreto.

La experiencia obtenida hasta este momento muestra precisamente la
importancia de este enfoque. Los modelos pueden resultar útiles para
determinadas tareas de generación, análisis y consulta, mientras que
otros escenarios, como la utilización de agentes autónomos sobre un
entorno de desarrollo real, pueden presentar limitaciones importantes.

Por ello, el objetivo final del laboratorio es desarrollar un criterio
técnico para responder a preguntas como:

-   ¿Qué puede hacer un modelo local de forma fiable?
-   ¿Qué recursos necesita?
-   ¿Qué modelo resulta adecuado para cada tarea?
-   ¿Cuándo compensa utilizar IA local?
-   ¿Cómo puede integrarse con otros servicios?
-   ¿Qué nivel de automatización es razonable?
-   ¿Qué controles de seguridad son necesarios?
-   ¿Qué tareas deben seguir requiriendo supervisión humana?

La respuesta a estas preguntas se construirá progresivamente a partir de
experimentos, mediciones y documentación, que constituyen la base
metodológica de todo el proyecto.
