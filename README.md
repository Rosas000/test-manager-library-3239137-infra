# Manager Library — Infra (Docker Compose)

Infraestructura de Manager Library: levanta todos los servicios del proyecto con un solo comando. Incluye PostgreSQL 16, API Laravel, frontend React y sitio estático.

*Prueba técnica SDD · Ficha ADSO 3239137 · SENA · Viernes 23 de octubre de 2026*

## Stack

* Docker
* Docker Compose
* PostgreSQL 16
* Nginx (API y page)
* Node 20 (build de app)

## Arranque

```bash
# 1. Clonar y entrar
git clone <url-del-repo>
cd test-manager-library-3239137-infra

# 2. Copiar entorno
cp .env.example .env
# Editar .env: JWT_SECRET, ADMIN_USERNAME, ADMIN_PASSWORD, CORS_ORIGIN

# 3. Levantar todo
docker compose up -d --build

# 4. Migrar la base
docker compose exec api php artisan migrate --force

# 5. Sembrar datos (opcional)
docker compose exec api php artisan app:seed-demo
```

### Servicios disponibles

| Servicio | URL |
| :--- | :--- |
| **API** | `http://localhost:8000/api/v1/health` |
| **Frontend (app)** | `http://localhost:5173` |
| **Sitio estático (page)** | `http://localhost:3000` |
| **PostgreSQL** | `localhost:5432` |

### Detener

```bash
docker compose down            # detiene sin borrar datos
docker compose down -v         # detiene y borra volúmenes (base vacía)
```

## Estructura

```text
/
├── docker-compose.yml         # Orquestación de servicios
├── .env.example               # Variables de entorno
├── api/
│   ├── Dockerfile             # PHP 8.3 + Laravel
│   └── nginx.conf             # Configuración de Nginx para la API
├── app/
│   ├── Dockerfile             # Node 20 + Vite (build + serve)
│   └── nginx.conf             # Proxy inverso para React
├── page/
│   ├── Dockerfile             # Nginx estático
│   └── nginx.conf
├── db/
│   └── init.sql               # Creación del esquema `library` (vacío)
└── README.md
```

## Servicios

| Servicio | Imagen | Puerto | Descripción |
| :--- | :--- | :---: | :--- |
| **db** | `postgres:16` | 5432 | PostgreSQL con esquema `library` vacío |
| **api** | `library-api` (build) | 8000 | Laravel + PHP-FPM + Nginx |
| **app** | `library-app` (build) | 5173 | React en modo desarrollo (Vite) |
| **page** | `library-page` (build) | 3000 | Sitio estático con Nginx |

## Variables de entorno (.env)

| Variable | Valor por defecto | Nota |
| :--- | :--- | :--- |
| `DB_HOST` | `db` | Nombre del servicio en Docker |
| `DB_PORT` | `5432` | Puerto interno de PostgreSQL |
| `DB_DATABASE` | `library` | Nombre de la base |
| `DB_USERNAME` | `library` | Usuario |
| `DB_PASSWORD` | `library123` | Contraseña |
| `JWT_SECRET` | *(generar)* | `php artisan jwt:secret` |
| `ADMIN_USERNAME` | `admin` | Usuario inicial |
| `ADMIN_PASSWORD` | `admin123` | Clave inicial |
| `CORS_ORIGIN` | `http://localhost:5173` | Origen del frontend |

## Estado de la tarea

| Tarea | Estado |
| :--- | :--- |
| **T-20** `docker compose up` levanta motor vacío, API y frontend | hecha |

### Evidencia

```bash
$ docker compose up -d --build

[+] Building 4/4
 ✔ db      Pulled
 ✔ api     Built
 ✔ app     Built
 ✔ page    Built

[+] Running 4/4
 ✔ Container library-db    Started
 ✔ Container library-api   Started
 ✔ Container library-app   Started
 ✔ Container library-page  Started
```

**Verificación de salud:**

```bash
$ curl http://localhost:8000/api/v1/health
{"status":"ok"}

$ curl -I http://localhost:3000
HTTP/1.1 200 OK
Server: nginx

$ curl -I http://localhost:5173
HTTP/1.1 200 OK
```

**La base arranca sin tablas hasta correr las migraciones (ADR-001):**

```bash
$ docker compose exec db psql -U library -d library -c "\dt library.*"
Did not find any relations.

$ docker compose exec api php artisan migrate --force
Migration: 2026_10_22_000001_create_library_schema ... DONE

$ docker compose exec db psql -U library -d library -c "\dt library.*"
             List of relations
 Schema |    Name    | Type  |  Owner
--------+------------+-------+----------
 library| book       | table | library
 library| category   | table | library
 library| loan       | table | library
 library| loan_item  | table | library
 library| member     | table | library
 library| user       | table | library
(6 rows)
```

## Desviaciones del spec

| # | Situación | Decisión |
| :-: | :--- | :--- |
| **DV-I1** | Nginx para la API en lugar de `php artisan serve` | Más cercano a producción; maneja mejor concurrencia |
| **DV-I2** | Frontend en modo desarrollo (Vite) en lugar de build estático | Permite hot-reload durante la prueba; para producción se cambiaría a `npm run build` + Nginx |
| **DV-I3** | Volúmenes nombrados para persistencia de la base | Los datos sobreviven a `docker compose down` (sin `-v`) |

## Relación con los demás repositorios

| Repo | Rol |
| :--- | :--- |
| `test-manager-library-3239137-docs` | Spec completo |
| `test-manager-library-3239137-api` | Backend Laravel (imagen `library-api`) |
| `test-manager-library-3239137-app` | Frontend React (imagen `library-app`) |
| `test-manager-library-3239137-page` | Sitio estático (imagen `library-page`) |
| `test-manager-library-3239137-tool` | Sembrador (se ejecuta dentro del contenedor `api`) |
| **Este repo** | Orquestación Docker Compose |

## Flujo completo de arranque

```bash
# 1. Clonar los 6 repos en la misma carpeta
git clone <url-docs>    test-manager-library-3239137-docs
git clone <url-api>     test-manager-library-3239137-api
git clone <url-app>     test-manager-library-3239137-app
git clone <url-page>    test-manager-library-3239137-page
git clone <url-infra>   test-manager-library-3239137-infra
git clone <url-tool>    test-manager-library-3239137-tool

# 2. Entrar a infra y levantar
cd test-manager-library-3239137-infra
cp .env.example .env
docker compose up -d --build

# 3. Migrar y sembrar
docker compose exec api php artisan migrate --force
docker compose exec api php artisan app:seed-demo

# 4. Abrir en el navegador
#    http://localhost:5173  → aplicación React
#    http://localhost:3000  → sitio de presentación
```
