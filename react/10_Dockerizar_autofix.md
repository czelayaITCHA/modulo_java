# 10 — Dockerizar AutoFix (autofix-api + autofix-app)

Guía para estudiantes y docente — WSL2 y Docker Desktop ya instalados en las
laptops. Cubre **qué hace cada línea** de los archivos, no solo cómo
copiarlos.

## Arquitectura: 2 contenedores + PostgreSQL de tu propia PC

```
┌─────────────────┐      ┌──────────────────┐      ┌──────────────────────┐
│  autofix-app    │      │  autofix-api     │      │  PostgreSQL          │
│  React + Vite   │───▶ |  Spring Boot 4.1 │ ────▶│  instalado en TU PC  │
│  servido por    │ HTTP │  Java 21         │ JDBC │  (fuera de Docker)   │
│  Nginx, puerto  │      │  puerto 8080     │      │  puerto 5432         │
│  5173           │      │                  │      │                      │
└─────────────────┘      └──────────────────┘      └──────────────────────┘
     Contenedor              Contenedor                  Proceso normal
      Docker                  Docker                     de Windows
```

**La base de datos NO se dockeriza en esta guía** — cada estudiante usa el
PostgreSQL que ya tenga instalado directo en su Windows (no dentro de un
contenedor). Esto simplifica el aprendizaje: menos piezas moviéndose a la
vez, y los datos no desaparecen cada vez que reconstruyes los contenedores.

**Nota importante sobre esta arquitectura**: `autofix-api` y `autofix-app`
son **dos aplicaciones totalmente independientes** — el navegador del
habla directo con cada una por separado (el JavaScript de React
hace peticiones `axios` directo a `http://localhost:8080`, no "a través"
del contenedor de Nginx). Por eso son 2 contenedores separados, cada uno
con su propio `Dockerfile`, en vez de uno solo.

---

## Prerrequisito: PostgreSQL instalado en tu PC

Si todavía no lo tiene:
1. Descargue el instalador desde <https://www.postgresql.org/download/windows/>
2. Durante la instalación, anote la contraseña que le ponga al usuario `postgres`
3. Cree la base de datos y el usuario que usará la app:

```sql
CREATE DATABASE autofix_db # ya esta creada;
CREATE USER autofix WITH PASSWORD 'autofix123';
GRANT ALL PRIVILEGES ON DATABASE autofix_db TO autofix;
```

**Importante**: el nombre `autofix_db` no es arbitrario — es el valor por
defecto que ya trae `application.properties` (`${DB_NAME:autofix_db}`). Si
usa otro nombre, usuario o contraseña, solo asegúrase de reflejarlo en las
variables `DB_NAME`/`DB_USER`/`DB_PASSWORD` del `docker-compose.yml` más
abajo.

**Un detalle que suele fallar la primera vez**: por defecto, PostgreSQL solo
acepta conexiones desde `localhost` — y un contenedor Docker, aunque corra
en la misma PC, técnicamente es una "red" distinta. Hay que decirle a
Postgres que acepte conexiones desde ahí también:

1. Busque el archivo `postgresql.conf` (con el instalador oficial, suele
   estar en `C:\Program Files\PostgreSQL\<version>\data\postgresql.conf`).
   Busque la línea `listen_addresses` y confirme que diga `'*'` (todas las
   interfaces), no solo `'localhost'`.
2. En el mismo directorio, edite `pg_hba.conf` y agrega esta línea al
   final (permite conexiones desde la red interna de Docker):
   ```
   host    all             all             172.17.0.0/16           scram-sha-256
   ```
3. Reinicie el servicio de PostgreSQL (desde "Servicios" de Windows, o
   `net stop postgresql-x64-16` / `net start postgresql-x64-16`, ajustando
   el nombre exacto del servicio a su versión).

---

## Estructura de carpetas esperada

```
autofix/
├── docker-compose.yml
├── autofix-api/
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/
└── autofix-app/
    ├── Dockerfile
    ├── nginx.conf
    ├── package.json
    └── src/
```

Cada estudiante copia o clona ambos repos (`autofix-api` y `autofix-app`) como
carpetas **hermanas**, y coloca el `docker-compose.yml` en el nivel de
arriba, junto a ambas.

---

## 1. `autofix-api/Dockerfile` — explicado línea por línea

```dockerfile
# ── Etapa 1: compilar con Maven ──
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app

COPY pom.xml .
RUN mvn dependency:go-offline -B

COPY src ./src
RUN mvn clean package -DskipTests

# ── Etapa 2: imagen final, solo con el JRE ──
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

COPY --from=build /app/target/*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

| Línea | Qué hace | Por qué |
|---|---|---|
| `FROM maven:3.9-eclipse-temurin-21 AS build` | Arranca desde una imagen que ya trae Maven 3.9 + JDK 21 instalados, y le pone el **alias** `build` | Necesitamos el JDK completo (no solo el JRE) y Maven para **compilar** el código — pero esa imagen pesa mucho (~500MB+), no la queremos en el contenedor final |
| `WORKDIR /app` | Crea (si no existe) y se posiciona dentro de `/app` como directorio de trabajo | Todos los comandos siguientes (`COPY`, `RUN`) ocurren relativos a esta carpeta, dentro del contenedor |
| `COPY pom.xml .` | Copia **solo** el `pom.xml` (la lista de dependencias) al contenedor, todavía no el código | Truco clave de cacheo: si tu código cambia pero `pom.xml` no, Docker puede **reutilizar** el resultado del siguiente paso (descargar dependencias) en vez de repetirlo — builds mucho más rápidos |
| `RUN mvn dependency:go-offline -B` | Descarga **todas** las dependencias declaradas en `pom.xml` (Spring, JWT, iTextPDF, etc.), sin compilar nada todavía | Con esto en su propia capa de Docker, si luego solo cambias una clase Java, Docker no vuelve a descargar 200MB de dependencias — usa la capa cacheada |
| `COPY src ./src` | Ahora sí copia el código fuente real (`src/main/java/...`) | Se hace **después** de las dependencias, a propósito (ver cacheo arriba) |
| `RUN mvn clean package -DskipTests` | Compila el proyecto y genera el `.jar` ejecutable en `target/` | `-DskipTests` porque en un build de Docker normalmente no quieres que fallen los tests y bloqueen la imagen — los tests se corren aparte |
| `FROM eclipse-temurin:21-jre-alpine` | **Segunda imagen, desde cero** — solo trae el JRE (Java Runtime, para *ejecutar* Java), no el JDK ni Maven | Esta es la imagen que de verdad vas a usar — mucho más liviana (~150MB en vez de 500MB+), porque no necesitas compilar nada aquí, solo correr el `.jar` ya hecho |
| `COPY --from=build /app/target/*.jar app.jar` | Trae **solo** el archivo `.jar` compilado desde la Etapa 1 (`build`), y lo renombra a `app.jar` | Este es el corazón del *multi-stage build*: todo el peso de Maven/JDK/código fuente/dependencias de la Etapa 1 se **descarta**, solo pasa el resultado final |
| `EXPOSE 8080` | Documenta que este contenedor escucha en el puerto 8080 | Es informativo — no abre el puerto por sí solo (eso lo hace `docker-compose.yml` con `ports:`), pero ayuda a quien lea el Dockerfile a saber qué puerto usa |
| `ENTRYPOINT ["java", "-jar", "app.jar"]` | El comando que se ejecuta cuando arranca el contenedor | Equivale a correr `java -jar app.jar` a mano — así es como cualquier `.jar` de Spring Boot se ejecuta normalmente |

**El concepto más importante aquí es el *multi-stage build*** (dos `FROM` en
el mismo archivo): usas una imagen "de trabajo" pesada para compilar, y
otra imagen "final" liviana para ejecutar — quedándote solo con lo que
realmente necesitas correr, no con las herramientas que usó para
desarrollarlo.

---

## 2. `autofix-app/Dockerfile` — explicado línea por línea

```dockerfile
# ── Etapa 1: compilar con Node ──
FROM node:22-bookworm-slim AS build
WORKDIR /app

COPY package.json ./
RUN npm install

COPY . .

ARG VITE_API_URL=http://localhost:8080/api
ARG VITE_IMAGES_URL=http://localhost:8080/images
ENV VITE_API_URL=$VITE_API_URL
ENV VITE_IMAGES_URL=$VITE_IMAGES_URL

RUN npm run build

# ── Etapa 2: servir los archivos ya compilados con Nginx ──
FROM nginx:alpine

COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
```

| Línea | Qué hace | Por qué |
|---|---|---|
| `FROM node:22-bookworm-slim AS build` | Imagen con Node 22 instalado (basada en Debian/glibc), con alias `build` | Necesitas Node para correr `npm`/`vite build` — pero Node **no** hace falta para servir los archivos ya compilados. Usamos la variante `bookworm-slim` (Debian) y no `alpine` (musl) porque varios paquetes de este proyecto (`rolldown`, el compilador de Vite 8; `lightningcss`, el minificador de CSS) traen binarios nativos por plataforma, y musl da más problemas de compatibilidad con ese tipo de paquete |
| `COPY package.json ./` | Copia **solo** `package.json`, a propósito **sin** `package-lock.json` | El lockfile de este proyecto se generó en Windows — si lo copiamos y usamos `npm ci` (que lo reproduce tal cual), arrastra resoluciones de paquetes nativos pensadas para Windows, no para Linux. Sin el lockfile, `npm install` resuelve todo de nuevo, correctamente, para el Linux del contenedor |
| `RUN npm install` | Instala las dependencias, resolviendo todo desde cero | Se usa `npm install` (no `npm ci`) específicamente por la razón de arriba — es la excepción a la regla general de "usa `npm ci` en Docker", justificada por el problema real que estamos evitando |
| `COPY . .` | Copia todo el resto del proyecto (código React, `index.html`, etc.) | Después de instalar dependencias, por la misma razón de cacheo que en la API |
| `ARG VITE_API_URL=...` / `ARG VITE_IMAGES_URL=...` | Declaran dos variables **solo disponibles durante el build**, con valores por defecto que coinciden con el `.env` real del proyecto (`http://localhost:8080/api` y `.../images`) | `ARG` (no `ENV`) porque esto se necesita únicamente en el momento de compilar — Vite "hornea" estos valores directo dentro del JavaScript final. Si tu proyecto usa más variables `VITE_*` en su `.env`, agrégalas aquí con el mismo patrón (un `ARG` + un `ENV` por cada una) |
| `ENV VITE_API_URL=...` / `ENV VITE_IMAGES_URL=...` | Convierten esos `ARG` en variables de entorno reales, visibles para el proceso de `npm run build` | Vite lee las variables `VITE_*` del entorno específicamente al momento de construir — sin este paso, el build no vería el valor de los `ARG` |
| `RUN npm run build` | Corre el script `"build": "vite build"` de tu `package.json`, generando archivos estáticos en `dist/` (HTML/JS/CSS ya compilados y optimizados) | Este es el resultado final: ya no necesitas Node para nada más, son archivos planos |
| `FROM nginx:alpine` | **Segunda imagen**, un servidor web liviano | React/Vite compilan a archivos **estáticos** — no necesitas un "servidor de aplicación" como con Spring Boot, solo algo que sirva HTML/JS/CSS, y Nginx es el estándar para esto |
| `COPY --from=build /app/dist /usr/share/nginx/html` | Trae **solo** la carpeta `dist/` (el resultado del build) desde la Etapa 1 | Igual que con el `.jar` — te quedas solo con el resultado final, no con Node ni `node_modules` (que puede pesar cientos de MB) |
| `COPY nginx.conf /etc/nginx/conf.d/default.conf` | Reemplaza la configuración por defecto de Nginx con una propia | Necesaria específicamente para que **React Router** funcione (ver más abajo) |
| `EXPOSE 80` | Documenta que Nginx escucha en el puerto 80 (su puerto estándar) dentro del contenedor | Igual que en la API — informativo, el mapeo real lo hace `docker-compose.yml` |

### ¿Por qué hace falta `nginx.conf` aparte?
Debe crearse el archivo en el raíz de autofix-app
```nginx
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

`react-router-dom` maneja la navegación **del lado del navegador** (nunca
hay una petición real al servidor cuando hace clic en un link interno).
Pero si alguien **refresca la página** estando en, por ejemplo,
`/dashboard`, el navegador sí le pide ese archivo a Nginx — y como
`/dashboard` no es un archivo real en `dist/`, Nginx daría un 404 sin esta
configuración. La línea `try_files ... /index.html` le dice: "si no
encuentras ese archivo, sirve `index.html` de todas formas" — y ahí React
Router retoma el control y muestra la ruta correcta.

---

## 3. `docker-compose.yml` — explicado en detalle

```yaml
services:
  api:
    build:
      context: ./autofix-api
      dockerfile: Dockerfile
    container_name: autofix_api
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      DB_HOST: host.docker.internal
      DB_PORT: "5432"
      DB_NAME: autofix_db
      DB_USER: postgres
      DB_PASSWORD: postgres
    extra_hosts:
      - "host.docker.internal:host-gateway"
    volumes:
      - autofix_uploads:/app/uploads

  app:
    build:
      context: ./autofix-app
      dockerfile: Dockerfile
      args:
        VITE_API_URL: http://localhost:8080/api
        VITE_IMAGES_URL: http://localhost:8080/images
    container_name: autofix_app
    restart: unless-stopped
    ports:
      - "5173:80"
    depends_on:
      - api

volumes:
  autofix_uploads:
```

### Servicio `api`

| Línea | Explicación |
|---|---|
| `build: context: ./autofix-api` | Le dice a Docker "el código para esta imagen está en la carpeta `autofix-api/`", y ahí construye la imagen usando el `Dockerfile` que ya explicamos |
| `DB_HOST / DB_PORT / DB_NAME / DB_USER / DB_PASSWORD` | **Este es el detalle más importante de todo el archivo, y hay que tener mucho cuidado aquí**: estos nombres **no** son un estándar de Spring Boot — son variables **propias de este proyecto**, y tienen que coincidir letra por letra con lo que aparece dentro de `${...}` en `application.properties` (`${DB_HOST:localhost}`, `${DB_NAME:autofix_db}`, etc.). Si usaras, por ejemplo, `SPRING_DATASOURCE_URL` en vez de estas, Spring Boot simplemente las **ignoraría** — sin error visible, solo usaría los valores por defecto del archivo (`localhost`, que dentro de un contenedor apunta a sí mismo, no a tu PC) |
| `host.docker.internal` (como valor de `DB_HOST`) | Nombre especial que Docker Desktop resuelve automáticamente hacia tu propia PC — así el contenedor "sale" de la red de Docker y llega al PostgreSQL que instalaste directo en Windows |
| `extra_hosts: host.docker.internal:host-gateway` | En Docker Desktop para Windows, `host.docker.internal` normalmente ya funciona sin esta línea — se agrega como respaldo, por si acaso, sin que cause ningún problema tenerla de más |
| `volumes: autofix_uploads:/app/uploads` | `application.properties` guarda los archivos subidos en una carpeta local (`app.upload.dir=${UPLOAD_DIR:uploads}`) — sin este volumen, esos archivos vivirían **dentro** del contenedor y se perderían cada vez que reconstruyas la imagen. El volumen los guarda aparte, de forma persistente |

### Servicio `app`

| Línea | Explicación |
|---|---|
| `build: args: VITE_API_URL / VITE_IMAGES_URL` | Pasa ambos valores a los `ARG` que definimos en el Dockerfile de `autofix-app` — nota que dicen `localhost`, no `host.docker.internal` |
| `ports: "5173:80"` | Mapea el puerto 80 de **dentro** del contenedor (donde Nginx escucha) al puerto 5173 de tu PC |
| `depends_on: - api` | Espera a que el contenedor de `api` **arranque** antes de levantar `app` — no garantiza que Spring Boot esté 100% listo para recibir peticiones, solo que el contenedor ya inició |

### ¿Por qué `DB_HOST` usa `host.docker.internal` pero `VITE_API_URL` usa `localhost`?

Esto confunde a casi todo el mundo la primera vez, y es clave entenderlo —
son dos conexiones que ocurren en momentos y lugares distintos:

- **`DB_HOST` usa `host.docker.internal`** porque esa
  conexión la hace el contenedor de `api`, que vive **dentro** de Docker —
  necesita ese nombre especial para "salir" y alcanzar tu PC.
- **`VITE_API_URL` usa `localhost`** porque esa conexión la hace el
  **navegador del estudiante** (JavaScript corriendo en Chrome/Firefox en
  su propia PC, no dentro de ningún contenedor) — el navegador no sabe
  nada de Docker ni de `host.docker.internal`; solo conoce
  `localhost:8080`, el puerto que expusiste hacia su propia máquina.

```
Contenedor api ──(host.docker.internal)──▶ PostgreSQL en tu PC
Navegador ──(localhost:8080)──▶ Contenedor api
Navegador ──(localhost:5173)──▶ Contenedor app
```

---

## Cómo levantarlo (para los estudiantes)

**1. Clonar ambos repos como carpetas hermanas:**
```bash
mkdir autofix && cd autofix
git clone <url-autofix-api> autofix-api
git clone <url-autofix-app> autofix-app
```

**2. Colocar los 4 archivos de esta guía:**
- `docker-compose.yml` → en `autofix/` (raíz)
- `Dockerfile` de la API → en `autofix/autofix-api/`
- `Dockerfile` y `nginx.conf` de la app → en `autofix/autofix-app/`

**3. Confirmar que PostgreSQL esté corriendo en tu PC** (ver la sección de
Prerrequisitos más arriba) antes de levantar los contenedores.

**4. Levantar todo:**
```bash
docker compose build
docker compose up -d
```

**5. Verificar que ambos contenedores estén sanos:**
```bash
docker compose ps
```

**6. Probar:**
- API: `http://localhost:8080` (o el endpoint que corresponda)
- App: `http://localhost:5173`

**7. Ver logs si algo falla:**
```bash
docker compose logs -f api
docker compose logs -f app
```

---

## Comandos útiles del día a día

| Comando | Qué hace |
|---|---|
| `docker compose up -d` | Levanta todo en segundo plano |
| `docker compose down` | Apaga y elimina los contenedores |
| `docker compose build --no-cache` | Reconstruye ignorando toda caché — útil si algo raro pasa y sospechas de una capa vieja |
| `docker compose logs -f <servicio>` | Sigue los logs en vivo de un servicio específico |
| `docker compose exec api sh` | Abre una terminal dentro del contenedor de la API (útil para inspeccionar) |
| `docker compose restart api` | Reinicia solo un servicio, sin tocar el otro |

---

## Errores comunes al hacerlo por primera vez

| Síntoma | Causa típica |
|---|---|
| `api` no logra conectarse a la base de datos | Revisa el prerrequisito de `listen_addresses` y `pg_hba.conf` — es el problema más común al conectar un contenedor con una base de datos "de afuera" |
| La app carga pero no trae datos, error de CORS en consola | Spring Security necesita permitir explícitamente el origen `http://localhost:5173` en su configuración de CORS — esto es código de la API, no de Docker |
| Cambios en el código no se reflejan | Olvidaste `docker compose build` de nuevo — a diferencia de correr el proyecto directo con `npm run dev` o desde tu IDE, aquí el código se "hornea" dentro de la imagen en cada build |
| `Cannot find native binding` (rolldown) o `Cannot find module '../lightningcss...'` al hacer `npm run build` dentro del contenedor | Paquetes con binarios nativos por plataforma (Vite 8 usa varios) resueltos incorrectamente por un `package-lock.json` generado en Windows — el Dockerfile ya usa `node:22-bookworm-slim` (no `alpine`) y `npm install` sin copiar el lockfile, para resolver todo de nuevo en Linux |
| `host.docker.internal` no resuelve | Poco común en Docker Desktop para Windows, pero si pasa, revisa que Docker Desktop esté actualizado a una versión reciente |
