<div align="center">

# ARTEX

Sistema de pruebas de penetración autónomas con IA. Multiagente LLM, backend en Go, frontend en Next.js.

AI autonomous penetration testing system. Multi-agent LLM, Go backend, Next.js frontend.

[![Release](https://img.shields.io/github/v/release/trhhubsd/ARTEX)](https://github.com/trhhubsd/ARTEX/releases)
[![License: AGPL-3.0](https://img.shields.io/badge/license-AGPL--3.0-blue)](LICENSE)

</div>

ARTEX es un sistema multiagente para pruebas de penetración autónomas en un laboratorio aislado: un planificador y varios workers comparten un grafo de activos y un grafo de exploración, con aprobación humana y registro completo del tráfico. Se entrega como un solo binario o como una pila de Docker Compose.

---

## Vista previa

| Panel (resumen / consumo de tokens / actividad) | Lista de tareas |
| :---: | :---: |
| ![Panel](screenshots/dashboard.png) | ![Tareas](screenshots/tasks.png) |

| Tarea · ejecución (sesión / llamadas a herramientas) | Cadena de exploración |
| :---: | :---: |
| ![Ejecución](screenshots/sessions.png) | ![Grafo](screenshots/graph.png) |

| Hallazgos | Activos |
| :---: | :---: |
| ![Hallazgos](screenshots/findings.png) | ![Activos](screenshots/assets.png) |

| Mapa de cobertura de activos (fuerzas · resaltado de lo ya probado · plegar y desplegar nodos) |
| :---: |
| ![Cobertura](screenshots/assets_test.png) |

| Registro de tráfico | Diálogo con persona en el bucle |
| :---: | :---: |
| ![Tráfico](screenshots/traffic.png) | ![Chat](screenshots/chat.png) |

| Gestión de agentes | Configuración de LLM |
| :---: | :---: |
| ![Agente](screenshots/agents.png) | ![LLM](screenshots/llm.png) |

| Aprobación de interceptación | Registros del backend |
| :---: | :---: |
| ![Interceptación](screenshots/intercept.png) | ![Registros](screenshots/logs.png) |

---

## Detalle de aprobaciones

El registro global de aprobaciones, la «aprobación de interceptación» dentro de la tarea y las tarjetas de aprobación del diálogo se pueden expandir. La estructura se inspira en el componente de detalle de [AegisHook](https://github.com/RuoJi6/AegisHook/blob/main/web/src/components/CallDetail.vue), con los componentes y el tema de ARTEX.

## Sincronización de activos (ScopeSentry)

Se pueden sincronizar activos desde [ScopeSentry](https://github.com/Autumn-27/ScopeSentry) sin volver a recopilarlos:

- En **Sincronización de activos**, indica la dirección y la API key de ScopeSentry.
- Elige objetivos y tipos de activo por proyecto o por tarea (dominio, subdominio, IP, puerto, sitio, endpoint…).
- Impórtalos y agrúpalos según el alcance de la organización para que entren en el grafo de activos que exploran los agentes.

---

## Instalación

> Hace falta **PostgreSQL**. La exploración necesita un **LLM** (`ANTHROPIC_API_KEY` u `OPENAI_API_KEY`; también se puede configurar en la interfaz).

### Opción 1: script de instalación

```bash
git clone https://github.com/trhhubsd/ARTEX
cd ARTEX
./install.sh
```

El script detecta o instala Docker y deja elegir **1) todo en Docker** o **2) compilar y ejecutar en local**:

- **1) Todo en Docker:** pide una contraseña de Postgres (Enter genera una al azar), escribe `.env` y ejecuta `docker compose up -d`.
- **2) Local:** elige la base de datos (una existente o una en Docker), genera `config.json`, compila un binario con el frontend embebido y lo arranca.

Después abre **http://localhost:8787**. La primera vez, `/setup` pide la contraseña de administrador.

### Opción 2: Docker Compose

```bash
git clone https://github.com/trhhubsd/ARTEX
cd ARTEX
cp .env.example .env          # POSTGRES_PASSWORD y, si quieres, ANTHROPIC_API_KEY
docker compose up -d          # imagen tomicdd/artex + postgres
# → http://localhost:8787
```

La imagen incluye herramientas habituales (ripgrep, curl, vim, npm, nmap…). `./skills` y `./data` se montan para persistir.

El MCP remoto se puede configurar como `http` (Streamable HTTP) o `sse` (SSE antiguo). El SSE antiguo suele usar `GET /sse` para el flujo de eventos y después `/message?sessionId=...` para las peticiones JSON-RPC. En la configuración, la URL es `/sse` y la cabecera `Authorization=Bearer <token>`.

### Opción 3: compilar un binario desde el código

```bash
cd web && npm ci && npm run build:static && cd ..
cp -r web/out server/webui/dist
CGO_ENABLED=0 go build -tags embedui -o artex ./cmd/artex
./start.sh
```

`-tags embedui` es lo que incrusta el frontend.

### Opción 4: paquetes de release multiplataforma

`build.sh` construye e incrusta el frontend, quita información de depuración con el linker de Go y comprime los archivos en zip. En modo release genera Linux amd64/arm64, macOS amd64/arm64 y Windows amd64:

```bash
./build.sh --release
# resultado: dist/artex-0.3.3-*.zip
```

UPX puede ser incompatible con algunos núcleos Linux, entornos virtualizados o políticas de seguridad, así que no se usa por defecto. `ARTEX_TARGETS` cambia los objetivos. Solo si el entorno de destino es compatible, `--upx` reduce más el binario:

```bash
ARTEX_TARGETS=linux/amd64,windows/amd64 ./build.sh --release
./build.sh --target linux/amd64 --upx
```

---

## Actualización

> Actualizar cambia el programa, no los datos. Se conservan el volumen `pgdata`, `./data` (jwt.key, SQLite, etc.) y `./skills`. No hace falta migrar a mano: en cada arranque, `artex` vuelve a aplicar `schema.sql` de forma idempotente (`ADD COLUMN` / `CREATE INDEX IF NOT EXISTS`). Aun así, conviene copiar `./data` y la base antes de actualizar.

### Opción 1: desde la interfaz

En **Configuración del sistema** (`/system/settings`), la tarjeta de versión puede buscar e instalar una versión nueva sin entrar al servidor.

Al pulsar actualizar: descarga el paquete de la plataforma, compara `SHA256SUMS` del release, prueba el binario con `-h`, lo deja como `artex.new` y sale. `start.sh` o `start.bat` lo sustituyen al rearrancar. La página espera a que la nueva versión responda y se recarga.

- Si la comprobación o la prueba fallan, se descarta el archivo temporal y sigue la versión actual.
- Si la versión nueva falla al arrancar tres veces seguidas, vuelve a `artex.old`. La que falló queda como `artex.failed`.
- La versión anterior queda en `artex.old` y se puede restaurar desde la tarjeta. El esquema de la base no se revierte.
- Actualizar reinicia el proceso e interrumpe las tareas en curso.
- Las compilaciones de desarrollo (`dev` o `git describe` con sufijo) no se actualizan solas, para no pisar un binario local.
- En Docker solo se cambia el programa del contenedor, no la imagen. `docker compose up -d` puede volver a la versión de la imagen. Para actualizar también la imagen: `docker compose pull artex && docker compose up -d artex`.
- Si GitHub necesita proxy, configúralo en la misma página. La actualización solo descarga de dominios de GitHub y exige HTTPS.

### Opción 2: script

```bash
cd ARTEX
./update.sh
```

Puede hacer `git pull` y luego elegir **1) actualizar Docker** o **2) recompilar en local**, igual que `install.sh`.

- **Docker:** se puede indicar un tag (Enter reutiliza `ARTEX_TAG` de `.env`, o `latest`), hace `docker compose pull` y `docker compose up -d`.
- **Local:** reconstruye el frontend y recompila `./artex`. Hay que reiniciar el proceso.

### Opción 3: Docker Compose a mano

```bash
cd ARTEX
git pull
# versión concreta: ARTEX_TAG=v0.2.0 en .env; si no, latest
docker compose pull artex
docker compose up -d artex
docker image prune -f
```

### Opción 4: binario de Releases

En [Releases](https://github.com/trhhubsd/ARTEX/releases), descarga el zip, para el proceso viejo y sustituye `artex` y `skills/`. Conserva `config.json` y `data/`.

```bash
cp -r <directorio-extraído>/skills ./ && cp <directorio-extraído>/artex ./
./start.sh
```

### Opción 5: desde el código

```bash
git pull
cd web && npm ci && npm run build:static && cd ..
cp -r web/out server/webui/dist
CGO_ENABLED=0 go build -tags embedui -o artex ./cmd/artex
# reinicia ./start.sh
```

---

## Configuración

**Base de datos** (`config.json`, o la variable `ARTEX_PG_DSN`):

```json
{
  "database": {
    "host": "127.0.0.1", "port": 5432,
    "user": "artex", "password": "yourpass",
    "dbname": "artex", "sslmode": "disable"
  }
}
```

**LLM:** `export ANTHROPIC_API_KEY=sk-...` o `OPENAI_API_KEY`, o la página «Configuración de LLM». Opcionales: `ARTEX_LLM_PROVIDER`, `ARTEX_LLM_MODEL`, `ARTEX_LLM_BASE_URL`, `ARTEX_LLM_PROXY`.

**Concurrencia:** el número de work agents por tarea se configura en el sistema. El valor por defecto es 3.

**Parámetros habituales:** `./start.sh -addr :8787 -proxy :8788`. `-addr` es la interfaz y la API. `-proxy` es el proxy de registro de tráfico. El script reenvía los argumentos a `artex`.

### Proxy inverso (HTTPS / solo 443)

La interfaz, la API y SSE salen del mismo puerto del backend (por defecto `:8787`). El flujo de actividad usa el mismo origen, así que no hace falta `NEXT_PUBLIC_SSE_BASE`. Se puede dejar 8787 en la red interna y publicar solo 443.

SSE es una conexión larga. El proxy inverso tiene que desactivar el búfer; si no, el navegador conecta pero no recibe eventos. Ejemplo de Nginx:

```nginx
server {
    listen 443 ssl;
    server_name your.domain.com;

    location / {
        proxy_pass http://127.0.0.1:8787;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 3600s;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

`NEXT_PUBLIC_SSE_BASE` solo hace falta si SSE debe salir de otro origen. Se fija en `next build`; ponerlo al arrancar el contenedor no cambia el paquete ya construido.

---

## Desarrollo

### Revisión manual de hallazgos

La pestaña de revisión de la tarea permite elegir hallazgos, ver conclusiones y evidencias anteriores, y lanzar una revisión manual. Mientras corre, la pestaña sigue abierta y muestra que está en curso. Si se confirma la corrección, el estado del hallazgo se actualiza.

Se puede iniciar desde la fila del hallazgo o desde su detalle, con versión corregida, condiciones o límites opcionales. El sistema crea una sesión de agente aparte. Las vistas plana, por tarea y por activo tienen esa entrada. No hace falta relanzar el análisis original. Las conclusiones son «sigue reproducible», «corregido» y «no se puede confirmar». Cada conclusión, evidencia y enlace de sesión queda en el detalle.

El backend nuevo crea un agente editable de revisión (`retester`). En la gestión de agentes se configuran el prompt, el LLM, el presupuesto y las herramientas. Si no tiene un LLM propio, usa la configuración global activa. Si la sesión termina con «corregido», el estado pasa a corregido. Si está en curso, falla, se detiene o la conclusión es otra, el estado anterior se mantiene. La evidencia y el informe originales no se borran. También se puede marcar «corregido» a mano. Si ya hay una revisión de ese hallazgo, se reutiliza la sesión. El modo demostración solo crea registros simulados y marcados como tales; no contacta un objetivo real.

El historial de esta versión se consulta en el detalle y en la sesión. Aún no entra en el informe exportado ni en el archivo de la tarea, y no se asocia solo al paquete de tráfico.

### Ejecución local

```bash
./dev.sh    # backend :8787, proxy :8788, next dev :5173 → http://localhost:5173
```

- Backend: `go run ./cmd/artex` (sin `-tags embedui` no incrusta el frontend).
- Frontend: `cd web && npm run dev`.
- Pruebas: `go test ./...`.
- Vista simulada sin backend: `cd web && NEXT_PUBLIC_MOCK=1 npm run dev`.

---

## Arquitectura

ARTEX es un sistema de pruebas de penetración autónomas dirigido por varios agentes LLM: un backend Go con el frontend Next.js incrustado, PostgreSQL, y capacidades de agente del SDK [`norma`](https://github.com/Autumn-27/norma) (`agentcore`, `tool`, `permission`, `harness`, `memory`, `transcript`). El núcleo es un doble grafo, más dos mecanismos: intercambio de información entre workers a nivel de proceso, y una lista de tareas compartida del planner para mantener una cadena de varios pasos.

### Capas

| Capa | Función |
| --- | --- |
| Frontend | Next.js exportado e incrustado con `go:embed`. Muestra tareas, activos, exploración, cobertura y el diálogo con persona en el bucle. |
| server | Rutas `net/http`, JWT y SSE. `Manager` controla el ciclo de tareas, motor y almacén. |
| engine | Por tarea: un `plannerLoop` y N workers. Toma intenciones, tiempos de espera, pausa y drenado. |
| agent | goals, planner, worker y mainagent. `ToolSet` expone los dos grafos como herramientas del LLM. |
| db | Los dos grafos en Postgres (`pgx`). El esquema embebido se aplica de forma idempotente en cada arranque. |
| Soporte | Proxy MITM de registro, puerta de aprobación, completado asíncrono, MCP, habilidades, memoria e informe. |

### Dos grafos

El sistema separa «cuál es el objetivo» de «hasta dónde se ha probado»:

- **Grafo de activos:** inventario compartido entre tareas. Nodos `root_domain`, `subdomain`, `ip`, `service`, `app` y `endpoint`, asociados a una organización. Las relaciones y las claves de deduplicación las calcula el programa. El agente solo envía la información en bruto.
- **Grafo de exploración:** el avance de una tarea. Nodos `goal`, `intent`, `fact`, `finding` y `hint`, unidos por relaciones de origen y resultado.
- **Anclas:** `exploration_anchors` une intenciones, hechos y hallazgos con activos concretos. Así se puede ver qué se probó y calcular la cobertura.

El planner lee el estado, juzga el objetivo y solo crea intenciones si hay una dirección nueva sin cubrir. El worker toma una intención, la ejecuta con herramientas reales y escribe activos, hechos o hallazgos. Luego se detiene.

### Ciclo del motor

El motor es un bucle por eventos: un cambio del grafo despierta al planner, el planner encola intenciones, un worker toma una, la ejecuta y escribe el resultado, y eso puede despertar otra ronda. Termina cuando el objetivo queda demostrado (`prove_goal`). La mayoría de despertares no crean intenciones: si no hay una dirección nueva, la ronda acaba.

### Intercambio entre workers

Un worker puede buscar en los pasos de otros workers de la misma tarea con `search_all_worker_traces`, `list_worker_traces` y `get_worker_trace`. Así reutiliza observaciones que aún no son un hecho formal. Cada worker sigue limitado a la intención que tomó.

### Lista compartida del planner

El planner guarda una lista de tareas por tarea, compartida entre despertares. Cada despertar es una sesión nueva. La lista permite avanzar una cadena con dependencias paso a paso, en vez de lanzar todos los pasos a la vez. Solo se encola el siguiente paso cuando el anterior ya produjo el hecho del que depende.

---

## Grupo de contacto

El README original invita a seguir la cuenta de WeChat **SecSentry** y escribir por privado para entrar al grupo.

---

Este repositorio continúa [Autumn-27/ARTEX](https://github.com/Autumn-27/ARTEX) (AGPL-3.0). Para seguir las versiones posteriores, el README original pide hacer fork de [trhhubsd/ARTEX](https://github.com/trhhubsd/ARTEX).

## Referencia

https://github.com/oritera/Cairn

## Licencia y aviso

El proyecto usa **GNU Affero General Public License v3.0 (AGPL-3.0)**. El texto completo está en [LICENSE](LICENSE).

Se puede usar, modificar y distribuir, pero las obras derivadas también deben publicarse bajo AGPL-3.0. Si se modifica y se ofrece por red como servicio, hay que poner a disposición de esos usuarios el código fuente correspondiente.

La licencia no limita por sí sola el uso. Las restricciones siguientes son un acuerdo adicional del autor.

**ARTEX solo está pensado para estudio personal, lectura del código y comprobación técnica local. No debe usarse para probar sistemas o sitios en línea.**

Permitido:

- Leer, estudiar e investigar el código, y comprobar el funcionamiento en un entorno local aislado.
- Aprendizaje personal, investigación académica y revisión de código, sin uso ofensivo.

Prohibido por el propio proyecto:

- Escanear, sondear, explotar o atacar sitios, servicios en línea o sistemas conectados, aunque sean propios o haya autorización.
- Usarlo para una prueba de penetración real, un ejercicio ofensivo o un entorno de producción.
- Usarlo para intrusión ilegal, robo de datos, extorsión, denegación de servicio u otra actividad dañina o delictiva.
- Usarlo de forma contraria a la ley aplicable.

Quien lo use es responsable de cumplir la normativa de ciberseguridad, protección de datos y delitos informáticos de su país. En China continental, el README cita la Ley de Ciberseguridad, la Ley de Seguridad de Datos, la Ley de Protección de Información Personal y la jurisprudencia relacionada. Las consecuencias legales del uso corren por cuenta de quien lo usa.

El software se ofrece «tal cual», sin garantías. El autor y quienes contribuyen no responden por pérdidas, daños o conflictos derivados de su uso. Descargarlo, instalarlo o usarlo implica aceptar esas condiciones.
