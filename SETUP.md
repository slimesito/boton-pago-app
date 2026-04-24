# Guía de Instalación — Botón de Pago IVSS

Instrucciones paso a paso para levantar el proyecto desde cero después de clonar el repositorio.

## Requisitos previos

| Herramienta | Versión mínima | Notas |
|---|---|---|
| Docker | 24+ | Con Docker Compose v2 (`docker compose`) |
| Node.js | 20+ | Solo para el frontend |
| npm | 10+ | Incluido con Node.js |
| Acceso a Oracle DB | — | Servidor externo, no se incluye en Docker |

---

## Estructura del proyecto

```
boton-pago/
├── backend/          ← API Laravel 13 (PHP 8.3 + Apache, corre en Docker)
│   ├── Dockerfile
│   ├── docker-compose.yml
│   └── oracle-client/   ← ZIPs del Oracle Instant Client (NO están en el repo, ver Paso 2.1)
└── frontend/         ← SPA React 19 + TypeScript (corre con Node.js local)
```

El backend expone la API en `http://localhost:8000`.  
El frontend corre en `http://localhost:5173` y redirige `/api` al backend automáticamente via proxy Vite.

---

## Paso 1 — Clonar el repositorio

```bash
git clone <url-del-repositorio>
cd boton-pago
```

---

## Paso 2 — Obtener los archivos del Oracle Instant Client

El ZIP del cliente Oracle pesa 129 MB y no puede estar en el repositorio. Debes colocarlo manualmente antes de construir la imagen Docker.

**Opción A — Desde Oracle (descarga oficial):**
1. Ir a https://www.oracle.com/database/technologies/instant-client/linux-x86-64-downloads.html
2. Descargar los paquetes de la versión **23.x** (misma que usa el `Dockerfile`):
   - `instantclient-basic-linux.x64-23.x.x.x.x.zip`
   - `instantclient-sdk-linux.x64-23.x.x.x.x.zip`
3. Colocar ambos archivos en `backend/oracle-client/`.

**Opción B — Desde almacenamiento compartido del equipo:**  
Copiar los ZIPs directamente desde la ubicación compartida del equipo a `backend/oracle-client/`.

Verificar que la carpeta quede así:

```
backend/oracle-client/
├── instantclient-basic-linux.x64-23.26.1.0.0.zip
└── instantclient-sdk-linux.x64-23.26.1.0.0.zip
```

> Los nombres exactos deben coincidir con los del `Dockerfile`. Si descargas una versión diferente, actualiza las líneas `unzip` y `instantclient_23_xx` en `backend/Dockerfile`.

---

## Paso 3 — Configurar el backend

### 3.1 Crear el archivo `.env`

```bash
cd backend
cp .env.example .env
```

### 3.2 Editar `.env` con las credenciales reales

Abrir `backend/.env` y completar las variables obligatorias:

```env
# ── Base de datos Oracle ─────────────────────────────────────
DB_HOST=<IP o hostname del servidor Oracle>
DB_PORT=1521
DB_DATABASE=<nombre del schema/SID>
DB_USERNAME=<usuario>
DB_PASSWORD=<contraseña>

# ── Biopago ─────────────────────────────────────────────────
# Sin credenciales reales → usar modo demo (ver sección "Modo demo" más abajo)
BIOPAGO_ENV=demo
BIOPAGO_BASE_URL=https://biodemo.ex-cle.com:4443/Biopago2/IPG2
BIOPAGO_TOKEN_URL=https://biodemo.ex-cle.com:4443/Biopago2/IPG2/connect/token
BIOPAGO_CLIENT_ID=
BIOPAGO_CLIENT_SECRET=

# ── URLs ─────────────────────────────────────────────────────
APP_URL=http://localhost:8000
BIOPAGO_RETURN_URL=http://localhost:8000/api/payments/return
FRONTEND_URL=http://localhost:5173
```

> **Oracle en el host local:** Si la base de datos corre en la misma máquina que Docker, usa `DB_HOST=host.docker.internal` — ya está configurado en `docker-compose.yml` como alias para el host.

> **Drivers de sesión y caché:** Si no quieres crear tablas adicionales en Oracle, deja `SESSION_DRIVER=file`, `CACHE_STORE=file` y `QUEUE_CONNECTION=sync`.

---

## Paso 4 — Construir e iniciar el contenedor Docker

Todos los comandos siguientes se ejecutan desde `backend/`:

```bash
# Construir la imagen (tarda varios minutos la primera vez — compila OCI8)
docker compose build

# Iniciar el contenedor en segundo plano
docker compose up -d
```

Verificar que el contenedor esté corriendo:

```bash
docker compose ps
```

Deberías ver `boton-pago-api` con estado `Up`.

---

## Paso 5 — Instalar dependencias PHP y configurar Laravel

```bash
# Instalar paquetes Composer dentro del contenedor
docker compose exec laravel-api composer install --no-dev --optimize-autoloader

# Generar la clave de la aplicación
docker compose exec laravel-api php artisan key:generate

# Compilar assets CSS/JS del backend (Laravel Vite)
docker compose exec laravel-api npm install
docker compose exec laravel-api npm run build
```

---

## Paso 6 — Ejecutar migraciones en Oracle

```bash
docker compose exec laravel-api php artisan migrate --force
```

Esto crea la tabla `PAYMENTS` y otras tablas auxiliares en el schema Oracle configurado.

> Si la conexión falla, verificar que `DB_HOST`, `DB_PORT`, `DB_DATABASE`, `DB_USERNAME` y `DB_PASSWORD` en `.env` sean correctos y que el servidor Oracle sea accesible desde el contenedor.

---

## Paso 7 — Configurar e iniciar el frontend

Desde la raíz del proyecto:

```bash
cd ../frontend

# Crear archivo de entorno (opcional — el proxy de Vite funciona sin él)
cp .env.example .env
# El archivo .env.example incluye instrucciones; en desarrollo no es necesario editar nada

# Instalar dependencias
npm install

# Iniciar servidor de desarrollo
npm run dev
```

El frontend quedará disponible en **http://localhost:5173**.

---

## Verificación

1. Abrir `http://localhost:5173` en el navegador.
2. Completar el formulario de pago con datos de prueba.
3. Confirmar que en la base de datos Oracle aparece un registro en la tabla `PAYMENTS` con `STATUS = 'pending'`.

Para comprobar la conectividad con Oracle desde el contenedor:

```bash
docker compose exec laravel-api php artisan tinker --execute 'echo json_encode(DB::select("SELECT 1 val FROM DUAL"));'
# Resultado esperado: [{"val":"1"}]
```

---

## Modo demo (sin credenciales Biopago)

Con `BIOPAGO_ENV=demo` en `.env`, el backend **simula** todas las llamadas a Biopago:
- `POST /api/payments/init` → genera un `paymentId` ficticio (`DEMO-xxxx`) y registra el pago en Oracle.
- `POST /api/payments/send-token` → responde éxito sin enviar OTP real.
- `POST /api/payments/process` → devuelve pago aprobado con código de autorización aleatorio.
- `GET /api/payments/groups` → retorna dos bancos ficticios (BDV y Banesco).

Los datos **sí se persisten en Oracle** incluso en modo demo.

Para activar el modo real cuando ya se tengan las credenciales:

```env
BIOPAGO_ENV=quality          # o 'production'
BIOPAGO_CLIENT_ID=<id real>
BIOPAGO_CLIENT_SECRET=<secret real>
```

Luego limpiar la caché de configuración:

```bash
docker compose exec laravel-api php artisan config:clear
```

---

## Comandos útiles del día a día

```bash
# Ver logs de la aplicación en tiempo real
docker compose exec laravel-api tail -f storage/logs/laravel.log

# Entrar al shell del contenedor
docker compose exec laravel-api bash

# Detener el contenedor
docker compose down

# Reconstruir desde cero (si cambia el Dockerfile o el oracle-client)
docker compose down && docker compose build --no-cache && docker compose up -d
```

---

## Flujo de pago (resumen técnico)

```
Frontend                     Backend (API)                  Biopago / Oracle
────────                     ─────────────                  ────────────────
1. Datos del pagador  →  POST /api/payments/init        →  crea registro en Oracle
                         ←  { payment_id, url_payment }
2. Selecciona banco   →  GET  /api/payments/groups      →  obtiene métodos disponibles
3. Solicita OTP       →  POST /api/payments/send-token  →  envía token al celular
4. Ingresa OTP        →  POST /api/payments/process     →  procesa en Biopago
                         ←  { approved: true/false }
5. Biopago redirige   →  GET  /api/payments/return      →  verifica y actualiza Oracle
6. Consulta estado    →  GET  /api/payments/{ref}/status ← status, auth_code, etc.
```
