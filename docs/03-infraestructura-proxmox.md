# 03. Infraestructura Proxmox

## 1. Introducción

La plataforma de inteligencia artificial se ejecuta sobre una infraestructura de virtualización basada en Proxmox. Esta capa constituye la base física y virtual sobre la que se despliegan posteriormente Linux, Docker, Ollama, OpenWebUI y el resto de servicios del laboratorio.

La finalidad de este documento es describir la infraestructura desde el punto de vista operativo: dónde se ejecuta la plataforma, qué recursos tiene disponibles, cómo se proporciona acceso a la GPU y cómo se organiza la red y el almacenamiento.

Es importante distinguir este documento del capítulo anterior.

En `02-arquitectura-general.md` se describe la arquitectura lógica de la plataforma y la relación entre sus componentes. En este documento se describe la infraestructura concreta que permite ejecutar dicha arquitectura.

La relación puede resumirse como:

```text
┌──────────────────────────────────────────┐
│                HARDWARE                  │
│                                          │
│ CPU · RAM · GPU · VRAM · SSD/HDD · NIC  │
└────────────────────┬─────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────┐
│                  PROXMOX                 │
│                                          │
│ Virtualización · almacenamiento · red    │
└────────────────────┬─────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────┐
│                  AI01                    │
│                                          │
│ Sistema Linux                            │
│ Docker                                   │
└────────────────────┬─────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────┐
│             SERVICIOS IA                 │
│                                          │
│ Ollama · OpenWebUI · SearXNG · ...       │
└──────────────────────────────────────────┘
```

---

## 2. Objetivos de la infraestructura

La infraestructura se ha diseñado para proporcionar un entorno local en el que sea posible:

- ejecutar modelos de lenguaje sin depender de una API externa;
- probar diferentes modelos y configuraciones;
- medir su rendimiento;
- proporcionar una interfaz gráfica de acceso;
- integrar aplicaciones externas mediante API;
- conservar los modelos y configuraciones entre reinicios;
- incorporar progresivamente nuevos servicios de IA;
- experimentar con automatización, RAG y otras aplicaciones;
- mantener separado el laboratorio de IA de otros servicios de infraestructura.

El objetivo no es construir inicialmente una plataforma de producción de alta disponibilidad, sino un laboratorio suficientemente estable y reproducible para estudiar las posibilidades de la IA local.

---

## 3. Nodo de IA

El servidor utilizado para la plataforma se identifica como:

```text
ai01
```

Este equipo actúa como host de los servicios de inteligencia artificial.

Durante las pruebas realizadas se ha comprobado que `ai01` proporciona acceso a una GPU desde el entorno Docker y que Ollama puede utilizarla durante la inferencia.

La cadena de ejecución es:

```text
Proxmox
   │
   ▼
ai01
   │
   ▼
Linux
   │
   ▼
Docker
   │
   ▼
Ollama
   │
   ▼
GPU
```

El nombre `ai01` se utilizará a lo largo de la documentación para referirse al servidor que ejecuta la plataforma.

---

## 4. Ficha técnica

Los datos que se han podido verificar durante el desarrollo del proyecto se separan de aquellos que todavía deben incorporarse a partir de la información directa del servidor.

### Datos confirmados

| Elemento | Valor |
|---|---|
| Host de IA | `ai01` |
| Plataforma de virtualización | Proxmox |
| Sistema de contenedores | Docker |
| Orquestación | Docker Compose |
| Motor de inferencia | Ollama |
| GPU disponible para IA | 1 GPU |
| VRAM disponible | 8 GB |
| Acceso de Docker a GPU | `gpus: all` |
| Red Docker de IA | `local-ai_ai-backend` |
| Red Docker interna | `172.18.0.0/16` |
| IP interna de Ollama en Docker | `172.18.0.4` |
| Puerto interno Ollama | `11434/tcp` |
| Puerto publicado de Ollama | `127.0.0.1:11434` |
| Interfaz publicada de OpenWebUI | `10.50.50.10:3000` |
| Persistencia de Ollama | Volumen Docker `local-ai_ollama-data` |
| Ruta del proyecto Compose | `/opt/local-ai` |
| Fichero Compose | `/opt/local-ai/compose.yaml` |

### Datos pendientes de documentar con exactitud

Los siguientes datos deben obtenerse directamente del entorno Proxmox/`ai01` antes de considerarlos definitivos:

| Elemento | Estado |
|---|---|
| Modelo exacto de CPU | AMD Ryzen 5 5600X |
| Número de núcleos/hilos | 8 |
| RAM total | 48 GB |
| Modelo exacto de GPU | Pendiente |
| VRAM exacta de la GPU | 8 GB |
| Tipo de almacenamiento | VirtIO SCSI Single |
| Capacidad de almacenamiento | 300 GB |
| Versión exacta de Proxmox | 9.2.21 |
| Tipo de recurso Proxmox (VM/LXC) | VM |
| ID de VM/CT | 100 |
| Método de asignación de GPU | Pendiente de documentar |

No se deben completar estos campos mediante estimaciones. La información definitiva debe proceder de la configuración real del servidor.

---

## 5. Por qué es importante documentar el hardware

En una plataforma de IA convencional puede resultar suficiente conocer el modelo del servidor y sus recursos generales.

En inferencia local, sin embargo, el hardware tiene una relación directa con el comportamiento del modelo.

El principal ejemplo de este proyecto es la GPU.

La plataforma dispone de una única GPU con 8 GB de VRAM. Esta capacidad determina, en gran medida, qué modelos pueden ejecutarse completamente acelerados por GPU.

Por tanto, la siguiente relación es fundamental:

```text
Hardware disponible
        │
        ├── CPU
        ├── RAM
        ├── GPU
        └── VRAM
             │
             ▼
      Modelos ejecutables
             │
             ▼
     Distribución CPU/GPU
             │
             ▼
       Rendimiento final
```

La documentación del hardware permite interpretar correctamente los resultados de las pruebas realizadas posteriormente.

Por ejemplo, si un modelo genera menos tokens por segundo que otro, no basta con comparar únicamente sus parámetros. Es necesario conocer también cómo se está distribuyendo el procesamiento entre CPU y GPU.

---

## 6. La GPU y sus 8 GB de VRAM

La principal restricción de hardware del laboratorio es disponer de una única GPU con 8 GB de VRAM.

La VRAM es la memoria de alta velocidad integrada o asociada a la GPU que se utiliza durante la ejecución de los modelos.

En términos simplificados:

```text
                GPU
        ┌─────────────────┐
        │                 │
        │     VRAM        │
        │      8 GB       │
        │                 │
        └─────────────────┘
```

Un modelo que puede mantenerse completamente en VRAM puede beneficiarse de una ejecución fuertemente acelerada por GPU.

Cuando el modelo requiere más memoria de la disponible, parte de la carga puede ejecutarse utilizando CPU y RAM.

Esto permite ejecutar modelos más grandes, pero normalmente con una reducción importante del rendimiento.

Por este motivo, los 8 GB de VRAM son una de las variables más importantes de todo el proyecto.

---

## 7. GPU compartida con Docker

La configuración de Docker Compose utilizada por Ollama incluye:

```yaml
ollama:
  image: ollama/ollama:latest
  container_name: ollama
  restart: unless-stopped
  gpus: all
```

La directiva:

```yaml
gpus: all
```

permite al contenedor utilizar las GPU disponibles en el host que Docker tiene configuradas para acceso por contenedores.

Esto es lo que permite que Ollama pueda ejecutar inferencia utilizando la GPU.

La arquitectura correspondiente es:

```text
                 Host
                   │
                   ▼
              Docker Engine
                   │
                   │ GPU access
                   ▼
          ┌─────────────────┐
          │     Ollama      │
          │   container     │
          └────────┬────────┘
                   │
                   ▼
                  GPU
                   │
                   ▼
                VRAM
```

La configuración exacta del acceso de la GPU desde Proxmox hasta el sistema operativo y posteriormente Docker debe documentarse con las características concretas del entorno.

---

## 8. Proxmox como capa de virtualización

Proxmox proporciona la capa situada entre el hardware físico y el sistema que ejecuta la plataforma de IA.

Conceptualmente:

```text
┌─────────────────────────────┐
│       Hardware físico       │
├─────────────────────────────┤
│          Proxmox            │
├─────────────────────────────┤
│          ai01               │
├─────────────────────────────┤
│          Linux              │
├─────────────────────────────┤
│          Docker             │
├─────────────────────────────┤
│   Ollama / OpenWebUI / ...  │
└─────────────────────────────┘
```

El uso de Proxmox proporciona una separación entre:

- infraestructura física;
- recursos virtualizados;
- servicios de IA.

Esta separación facilita la administración y permite modificar los recursos asignados sin alterar directamente los servicios de aplicación.

---

## 9. Acceso de la GPU desde Proxmox

La GPU debe recorrer varias capas antes de llegar al proceso de inferencia:

```text
GPU física
    │
    ▼
Proxmox
    │
    ▼
ai01
    │
    ▼
Docker
    │
    ▼
Ollama
    │
    ▼
Modelo
```

Por tanto, cuando la GPU deja de funcionar correctamente dentro de Ollama, el problema puede encontrarse en cualquiera de estas capas.

Una metodología adecuada de diagnóstico debe comprobarlas en orden.

### Nivel 1: Proxmox

Comprobar que el dispositivo físico es visible.

### Nivel 2: sistema operativo

Comprobar que `ai01` detecta la GPU.

### Nivel 3: runtime de contenedores

Comprobar que Docker puede acceder a ella.

### Nivel 4: Ollama

Comprobar que Ollama detecta y utiliza la GPU.

### Nivel 5: modelo

Comprobar la distribución CPU/GPU durante una inferencia.

Esta separación facilita enormemente el diagnóstico.

---

## 10. Comprobación de la GPU en el sistema operativo

El procedimiento exacto depende del fabricante de la GPU.

Como primera comprobación general puede utilizarse:

```bash
lspci | grep -Ei 'vga|3d|display'
```

Para una GPU NVIDIA, una comprobación habitual es:

```bash
nvidia-smi
```

La salida permite verificar, entre otros aspectos:

- modelo de GPU;
- memoria total;
- memoria utilizada;
- procesos que utilizan la GPU;
- versión del controlador.

---

## 11. Comprobación de GPU desde Docker

Una vez comprobado el sistema operativo, es necesario comprobar que Docker puede acceder a la GPU.

En un entorno NVIDIA, una prueba típica es:

```bash
docker run --rm --gpus all nvidia/cuda:12.*/base-ubuntu* nvidia-smi
```

La imagen concreta utilizada debe adaptarse a la versión de driver/runtime disponible.

No debe asumirse que cualquier versión de CUDA es compatible con cualquier controlador. La comprobación debe hacerse contra la configuración real del host.

Una vez validado el acceso, la configuración de Ollama utiliza:

```yaml
gpus: all
```

---

## 12. Docker Compose

La plataforma se gestiona mediante Docker Compose.

El proyecto se encuentra en:

```text
/opt/local-ai
```

y el fichero principal es:

```text
/opt/local-ai/compose.yaml
```

La configuración puede validarse mediante:

```bash
cd /opt/local-ai
docker compose config
```

Durante la instalación se utilizó esta comprobación para verificar que la configuración era sintácticamente válida:

```bash
docker compose config >/dev/null && echo "Compose OK"
```

Una salida:

```text
Compose OK
```

indica que Docker Compose ha podido procesar correctamente la configuración.

---

## 13. Servicios desplegados

En el estado actual del proyecto existen tres servicios principales dentro del Compose:

```text
local-ai
│
├── ollama
├── open-webui
└── searxng
```

El estado observado durante las pruebas fue equivalente a:

```text
NAME         IMAGE                                PORTS
ollama       ollama/ollama:latest                 ...
open-webui   ghcr.io/open-webui/open-webui:main  ...
searxng      searxng/searxng:latest               ...
```

Cada servicio tiene una función diferente.

### Ollama

Motor de inferencia y API de modelos.

### OpenWebUI

Interfaz web para interactuar con los modelos.

### SearXNG

Servicio de búsqueda utilizado como componente auxiliar.

---

## 14. Proyecto Docker Compose

La estructura lógica del proyecto es:

```text
/opt/local-ai/
│
├── compose.yaml
├── compose.yaml.bak-*
├── searxng/
│   └── ...
└── ...
```

Durante la modificación de la configuración se utilizó una copia de seguridad del fichero Compose:

```bash
cp -a compose.yaml "compose.yaml.bak.$(date +%Y%m%d-%H%M%S)"
```

Este procedimiento es recomendable antes de realizar cambios estructurales en la infraestructura.

Permite recuperar rápidamente una configuración anterior en caso de error.

---

## 15. Persistencia

Ollama utiliza un volumen Docker para almacenar sus datos.

El volumen identificado es:

```text
local-ai_ollama-data
```

y está montado en el contenedor como:

```text
/root/.ollama
```

La relación es:

```text
Docker volume
local-ai_ollama-data
        │
        ▼
/root/.ollama
        │
        ▼
Ollama
```

La ruta física del volumen en el host Docker se observó como:

```text
/var/lib/docker/volumes/local-ai_ollama-data/_data
```

No se recomienda modificar directamente el contenido interno del volumen salvo que exista una razón concreta y se conozca el formato utilizado por Ollama.

La administración normal debe realizarse a través de Ollama y Docker.

---

## 16. Modelos almacenados

Gracias a la persistencia del volumen, los modelos descargados permanecen disponibles después de reiniciar el contenedor.

Durante las pruebas se llegó a disponer de los siguientes modelos:

```text
qwen2.5-coder:7b
qwen2.5-coder:14b
qwen3:8b
qwen3:14b
qwen3:30b
qwen3.5:9b
gemma3:12b
```

La lista puede comprobarse mediante:

```bash
docker exec ollama ollama list
```

o, mediante la API:

```bash
curl -s http://127.0.0.1:11434/api/tags | jq '.models[].name'
```

La segunda opción resulta especialmente útil para comprobar que la API está operativa.

---

## 17. Red Docker

Los servicios de la plataforma están conectados a una red Docker denominada:

```text
local-ai_ai-backend
```

La red proporciona comunicación interna entre los contenedores.

Durante las pruebas, el contenedor de Ollama tenía:

```text
IP:      172.18.0.4
Gateway: 172.18.0.1
```

y estaba conectado a:

```text
local-ai_ai-backend
```

La red interna debe considerarse diferente de la red física o lógica de la infraestructura.

```text
Red física / Proxmox
          │
          ▼
       ai01
          │
          ▼
     Docker bridge
          │
          ▼
  172.18.0.0/16
          │
     ┌────┴────┐
     ▼         ▼
  Ollama   OpenWebUI
```

La dirección `172.18.0.4` es una dirección interna del entorno Docker y no debe utilizarse como dirección de acceso desde otros equipos de la red.

---

## 18. Publicación de Ollama

En la configuración inicial, Ollama no estaba publicado directamente sobre todas las interfaces del servidor.

La configuración final observada fue:

```text
127.0.0.1:11434 -> 11434/tcp
```

Esto significa que el puerto del servicio queda asociado a la interfaz loopback del host.

La ventaja principal de este diseño es reducir la superficie de exposición del servicio.

El acceso externo puede realizarse mediante un mecanismo de acceso controlado, en lugar de exponer directamente la API de Ollama a toda la red.

El procedimiento concreto utilizado para acceder desde el equipo personal se documentará en:

```text
12-acceso-remoto-y-red.md
```

---

## 19. OpenWebUI

OpenWebUI tiene una publicación diferente.

La configuración observada fue:

```text
10.50.50.10:3000 -> 8080/tcp
```

Esto significa que el puerto 3000 del host se utiliza para acceder al servicio web, mientras que el contenedor escucha internamente en el puerto 8080.

El flujo es:

```text
Cliente
   │
   │ TCP/3000
   ▼
10.50.50.10
   │
   ▼
OpenWebUI container
   │
   │ TCP/8080
   ▼
Aplicación web
```

La exposición de OpenWebUI y su seguridad de acceso se describirán con mayor detalle en el capítulo de red.

---

## 20. Comunicación entre OpenWebUI y Ollama

Aunque ambos servicios están publicados de forma diferente, dentro de Docker pueden comunicarse mediante la red interna.

El esquema es:

```text
                  Docker network
                local-ai_ai-backend
                         │
             ┌───────────┴───────────┐
             │                       │
             ▼                       ▼
       ┌───────────┐            ┌───────────┐
       │ OpenWebUI │ ─────────► │  Ollama   │
       └───────────┘   HTTP     └───────────┘
                                  │
                                  ▼
                                Modelos
```

La ventaja de utilizar la red interna es que los servicios no necesitan utilizar necesariamente la dirección externa del servidor para comunicarse entre ellos.

Docker proporciona resolución de nombres de servicio, por lo que Ollama puede ser identificado dentro de la red mediante:

```text
ollama
```

y su puerto:

```text
11434
```

---

## 21. Verificación de la configuración actual

La configuración de Docker puede inspeccionarse con:

```bash
docker ps
```

Para mostrar los puertos:

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}'
```

Para inspeccionar la red de Ollama:

```bash
docker inspect ollama --format '{{json .NetworkSettings.Networks}}' | jq
```

Para consultar el volumen:

```bash
docker inspect ollama --format '{{json .Mounts}}' | jq
```

Para conocer la ubicación del Compose:

```bash
docker inspect ollama --format '{{index .Config.Labels "com.docker.compose.project.working_dir"}}'
```

Y para identificar el fichero Compose:

```bash
docker inspect ollama --format '{{index .Config.Labels "com.docker.compose.project.config_files"}}'
```

Estas comprobaciones resultan útiles porque permiten reconstruir la configuración real a partir del propio sistema, en lugar de depender exclusivamente de notas manuales.

---

## 22. Estado de Ollama

El servicio puede comprobarse mediante:

```bash
docker exec ollama ollama list
```

Para comprobar los modelos cargados actualmente:

```bash
docker exec ollama ollama ps
```

Un ejemplo observado durante las pruebas:

```text
NAME         ID              SIZE     PROCESSOR          CONTEXT    UNTIL
qwen3:30b    ad815644918f    20 GB    66%/34% CPU/GPU    16384      ...
```

Esta información resulta particularmente importante para el diagnóstico de rendimiento.

La columna `PROCESSOR` permite observar si el modelo está siendo ejecutado principalmente en GPU o si una parte significativa del procesamiento está recayendo en CPU.

---

## 23. Relación entre recursos y rendimiento

La infraestructura debe interpretarse conjuntamente con los resultados obtenidos.

Durante las pruebas se observó una relación clara entre tamaño del modelo, distribución CPU/GPU y velocidad de generación.

Como ejemplo:

```text
Modelo                 CPU/GPU              Rendimiento observado
------------------------------------------------------------------
qwen2.5-coder:7b       100% GPU              ~53 tokens/s
qwen2.5-coder:14b      36/64 CPU/GPU         ~7,6 tokens/s
qwen3:8b               12/88 CPU/GPU         ~27 tokens/s
qwen3:30b              66/34 CPU/GPU         considerablemente menor
```

Estos valores pertenecen a pruebas concretas y no deben interpretarse como benchmarks universales de los modelos.

El resultado depende también de:

- prompt;
- longitud de respuesta;
- contexto;
- temperatura;
- cuantización;
- estado de la GPU;
- carga del sistema;
- versión de Ollama;
- versión del modelo.

No obstante, sirven para demostrar el efecto de la infraestructura sobre la experiencia de uso.

---

## 24. Gestión de modelos

La administración de los modelos se realiza mediante Ollama.

Para descargar un modelo:

```bash
docker exec ollama ollama pull <modelo>
```

Por ejemplo:

```bash
docker exec ollama ollama pull qwen2.5-coder:14b
```

Para ejecutar un modelo:

```bash
docker exec ollama ollama run <modelo>
```

Para detener un modelo cargado:

```bash
docker exec ollama ollama stop <modelo>
```

Para comprobar los modelos disponibles:

```bash
docker exec ollama ollama list
```

Para comprobar cuál está actualmente cargado:

```bash
docker exec ollama ollama ps
```

---

## 25. Configuración de contexto

La infraestructura debe soportar también diferentes tamaños de contexto.

Durante las pruebas se utilizaron:

```text
8192 tokens
16384 tokens
```

Por ejemplo, una petición directa a la API de Ollama puede incluir:

```json
{
  "model": "qwen2.5-coder:7b",
  "prompt": "...",
  "stream": false,
  "options": {
    "num_ctx": 16384
  }
}
```

El tamaño del contexto debe considerarse otro consumidor de recursos.

En una GPU con 8 GB de VRAM, aumentar `num_ctx` puede modificar el comportamiento de memoria y la distribución CPU/GPU.

Por ello, el contexto forma parte de la configuración experimental y no debe compararse entre modelos sin indicar su valor.

---

## 26. Reinicio y mantenimiento

Los servicios se gestionan mediante Docker Compose.

Para comprobar el estado:

```bash
cd /opt/local-ai
docker compose ps
```

Para iniciar los servicios:

```bash
docker compose up -d
```

Para iniciar únicamente Ollama:

```bash
docker compose up -d ollama
```

Para reiniciar Ollama:

```bash
docker compose restart ollama
```

Para detener la plataforma:

```bash
docker compose down
```

Debe distinguirse entre:

```bash
docker compose stop
```

y:

```bash
docker compose down
```

El primero detiene los contenedores, mientras que `down` desmonta la configuración de los servicios gestionados por Compose. La persistencia de los datos dependerá de los volúmenes utilizados y de si estos se eliminan explícitamente.

---

## 27. Seguridad de la infraestructura

La plataforma de IA debe considerarse un servicio de infraestructura y no simplemente una aplicación web.

Existen varios niveles que deben protegerse:

```text
Internet / LAN
       │
       ▼
   Proxmox
       │
       ▼
    ai01
       │
       ▼
    Docker
       │
       ├── OpenWebUI
       ├── Ollama
       └── SearXNG
```

Cada capa introduce una superficie potencial de ataque.

Especialmente importante es Ollama porque su API puede permitir:

- consultar modelos;
- generar contenido;
- ejecutar inferencias potencialmente costosas;
- consumir CPU;
- consumir GPU;
- consumir RAM;
- generar cargas prolongadas.

Por este motivo, la API no debe exponerse directamente a Internet sin controles adecuados.

---

## 28. Disponibilidad

La plataforma actual no está diseñada como un sistema de alta disponibilidad.

Existe un único nodo de IA y una única GPU.

Por tanto:

```text
Fallo de ai01
       ↓
Fallo de la plataforma IA
```

No existe actualmente redundancia de:

- nodo;
- GPU;
- servidor de inferencia;
- almacenamiento;
- OpenWebUI;
- Ollama.

Esto es coherente con el objetivo actual de laboratorio.

En fases posteriores podría estudiarse:

- segundo nodo;
- almacenamiento redundante;
- segunda GPU;
- replicación;
- monitorización;
- mecanismos de recuperación automática.

Estas posibilidades pertenecen a la evolución futura del proyecto y no forman parte de la arquitectura actual.

---

## 29. Monitorización

La monitorización de recursos es especialmente importante en este laboratorio porque el rendimiento depende directamente de los recursos disponibles.

Deben observarse al menos:

```text
CPU
RAM
GPU
VRAM
temperatura
almacenamiento
red
procesos Docker
```

Durante una inferencia puede resultar útil observar:

```bash
docker stats
```

y, en sistemas NVIDIA:

```bash
nvidia-smi
```

También puede comprobarse Ollama mediante:

```bash
docker exec ollama ollama ps
```

La combinación de estas herramientas permite relacionar:

```text
modelo
   ↓
contexto
   ↓
CPU/GPU
   ↓
VRAM/RAM
   ↓
tokens/s
```

Este enfoque será utilizado en la metodología de pruebas.

---

## 30. Inventario actual

El inventario lógico de la infraestructura puede resumirse como:

```text
Proxmox
└── ai01
    └── Docker
        └── local-ai
            ├── ollama
            │   ├── qwen2.5-coder:7b
            │   ├── qwen2.5-coder:14b
            │   ├── qwen3:8b
            │   ├── qwen3:14b
            │   ├── qwen3:30b
            │   ├── qwen3.5:9b
            │   └── gemma3:12b
            │
            ├── open-webui
            │
            └── searxng
```

Este inventario evolucionará conforme se incorporen nuevos componentes.

En particular, n8n está previsto como siguiente ampliación de la plataforma, aunque no forma todavía parte del despliegue actual documentado.

---

## 37. Comandos de inventario recomendados

Antes de cerrar definitivamente la ficha de infraestructura conviene ejecutar en `ai01` y en el nodo Proxmox los siguientes comandos.

### En el nodo Proxmox

```bash
pveversion -v
```

```bash
qm list
```

```bash
pct list
```

```bash
lspci | grep -Ei 'vga|3d|display'
```

### En `ai01`

```bash
hostnamectl
```

```bash
uname -a
```

```bash
lscpu
```

```bash
free -h
```

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

```bash
ip -br addr
```

```bash
ip route
```

```bash
docker version
```

```bash
docker compose version
```

En caso de utilizar NVIDIA:

```bash
nvidia-smi
```

Y para Docker:

```bash
docker info
```

La salida de estos comandos permitirá completar la ficha técnica sin introducir datos no verificados.

---

## 39. Conclusión

La infraestructura Proxmox constituye la base sobre la que se ejecuta todo el laboratorio de inteligencia artificial.

El servidor `ai01` proporciona el entorno de ejecución para Docker, mientras que Docker Compose organiza los principales servicios de la plataforma: Ollama, OpenWebUI y SearXNG.

La característica de hardware más relevante es la disponibilidad de una única GPU con 8 GB de VRAM. Esta limitación condiciona directamente la selección y el comportamiento de los modelos y explica por qué algunos modelos utilizan exclusivamente GPU mientras que otros requieren una combinación de CPU y GPU.

La infraestructura utiliza además persistencia mediante el volumen:

```text
local-ai_ollama-data
```

y mantiene una separación entre la red interna de Docker y los servicios publicados hacia el exterior.

El diseño actual prioriza:

- simplicidad;
- aislamiento;
- reproducibilidad;
- experimentación;
- aprovechamiento de los recursos disponibles;
- posibilidad de ampliación futura.

Al mismo tiempo, existen limitaciones claras: un único nodo, una única GPU, ausencia de alta disponibilidad y capacidad limitada de VRAM.

Estas limitaciones no deben ocultarse, sino formar parte de la documentación porque constituyen el contexto necesario para interpretar todos los resultados posteriores.

En particular, cualquier comparación entre modelos realizada en este laboratorio deberá considerar no solo las características del modelo, sino también el hardware y la configuración con la que se ha ejecutado.

El siguiente capítulo documentará el despliegue concreto de la plataforma de IA sobre esta infraestructura, incluyendo Docker Compose, Ollama, OpenWebUI, persistencia y los procedimientos utilizados para poner en funcionamiento los servicios.
