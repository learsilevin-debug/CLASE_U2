# API de Tareas (Aplicaciones Web – UPS, Unidad 2)

API REST de gestión de tareas con **Node.js + Express 5**, persistencia en **SQLite**, **sesiones** con cookie,
**archivos adjuntos** y **eventos en tiempo real** (Socket.IO).

## Requisitos y ejecución

- Node.js 22 o superior (`node -v` para comprobarlo).

```bash
npm install        # el .npmrc evita compilar: better-sqlite3 ya incluye binarios para Windows, macOS y Linux
npm start          # http://localhost:3000  (en "/" hay una página demo con eventos en vivo)
npm test           # 65 pruebas automáticas (node:test), sin servicios externos
```

> Si `npm install` intenta compilar y falla por falta de compilador, confirma que existe el archivo `.npmrc`
> (`ignore-scripts=true`) o usa `npm install --ignore-scripts`. Variables de ejemplo en `.env.example`.

Variables de entorno opcionales: `PORT`, `DB_PATH`, `UPLOAD_DIR`, `SESSION_SECRET` (obligatoria con `NODE_ENV=production`),
`MAX_FILE_BYTES` (2 MB por defecto), `MAX_LOGIN_ATTEMPTS`, `LOGIN_WINDOW_MS`.

## Arquitectura en capas

```
src/
  domain/        funciones PURAS: validación, DTOs, reglas (sin E/S, sin mutar entradas)
  repositories/  única capa con SQL (consultas parametrizadas)
  services/      casos de uso: orquestan dominio + repositorio + archivos + eventos
  controllers/   adaptan HTTP <-> servicios (delgados)
  routes/        recursos y métodos, 405 con cabecera Allow
  middleware/    sesión, JSON, subida, límite de intentos, manejador de errores
  storage/       acceso al sistema de archivos
  realtime/      notificaciones Socket.IO por usuario
  app.js         fábrica que ensambla todo (permite pruebas aisladas)
```

## Endpoints

| Método | Ruta | Descripción | Éxito |
|---|---|---|---|
| POST | `/api/auth/registro` | Crea usuario e inicia sesión | 201 |
| POST | `/api/auth/login` | Inicia sesión | 200 |
| POST | `/api/auth/logout` | Cierra sesión | 204 |
| GET | `/api/auth/yo` | Usuario de la sesión | 200 |
| GET | `/api/tareas` | Lista con filtros, búsqueda, orden y paginación | 200 |
| POST | `/api/tareas` | Crea tarea | 201 + `Location` |
| GET | `/api/tareas/:id` | Obtiene una tarea | 200 |
| PUT | `/api/tareas/:id` | Reemplazo completo (exige `version`) | 200 |
| PATCH | `/api/tareas/:id` | Modificación parcial (exige `version`) | 200 |
| DELETE | `/api/tareas/:id` | Elimina (y borra su adjunto) | 204 |
| POST | `/api/tareas/:id/archivo` | Adjunta archivo (`multipart`, campo `archivo`) | 201 |
| GET | `/api/tareas/:id/archivo` | Descarga el adjunto | 200 |
| DELETE | `/api/tareas/:id/archivo` | Quita el adjunto | 204 |
| GET | `/api/salud` | Comprobación de estado | 200 |

Filtros de `GET /api/tareas`: `estado`, `prioridad`, `q` (búsqueda literal en título/descripción),
`ordenar` (`creado|titulo|fecha_limite|prioridad`), `direccion` (`asc|desc`), `pagina`, `limite` (1–50).

### Códigos HTTP usados

`200, 201, 204` éxito · `400` JSON/id inválido · `401` sin sesión o credenciales erróneas · `404` no existe (o es de otro usuario) ·
`405` método no permitido · `409` conflicto (título duplicado, usuario duplicado, versión obsoleta) · `413` archivo/cuerpo muy grande ·
`415` tipo no soportado · `422` validación · `429` demasiados intentos de login · `500` error interno.

### Formato uniforme de errores

```json
{
  "error": {
    "codigo": "VALIDACION",
    "mensaje": "Los datos enviados no son válidos.",
    "detalles": [{ "campo": "titulo", "mensaje": "Debe tener entre 3 y 120 caracteres." }],
    "ruta": "POST /api/tareas",
    "marcaTiempo": "2026-11-02T20:30:30.808Z"
  }
}
```

## Ejemplos de petición / respuesta

Usa `curl -c cookies.txt` al registrarte/entrar y `-b cookies.txt` en las siguientes peticiones.

**Registro**
```bash
curl -i -c cookies.txt -X POST localhost:3000/api/auth/registro \
  -H 'content-type: application/json' -d '{"nombre":"ana","clave":"clave-segura-1"}'
```
```
HTTP/1.1 201 Created
Set-Cookie: sid=...; Path=/; HttpOnly; SameSite=Lax
{"usuario":{"id":1,"nombre":"ana"}}
```

**Crear tarea**
```bash
curl -i -b cookies.txt -X POST localhost:3000/api/tareas -H 'content-type: application/json' \
  -d '{"titulo":"Entregar informe","prioridad":"alta","fechaLimite":"2026-11-10"}'
```
```
HTTP/1.1 201 Created
Location: /api/tareas/1
{"tarea":{"id":1,"titulo":"Entregar informe","descripcion":"","estado":"pendiente","prioridad":"alta",
 "fechaLimite":"2026-11-10","version":1,"archivo":null,"creadoEn":"...","actualizadoEn":"..."}}
```

**Modificar parcialmente** (la `version` evita pisar cambios ajenos)
```bash
curl -i -b cookies.txt -X PATCH localhost:3000/api/tareas/1 -H 'content-type: application/json' \
  -d '{"estado":"en_progreso","version":1}'
```
```
HTTP/1.1 200 OK
{"tarea":{"id":1,...,"estado":"en_progreso","version":2,...}}
```

**Conflicto de versión** (repetir la misma petición con `version: 1`)
```
HTTP/1.1 409 Conflict
{"error":{"codigo":"CONFLICTO_VERSION","mensaje":"La tarea fue modificada por otra petición. Consúltala de nuevo y reintenta.",
 "detalles":[{"campo":"version","mensaje":"Enviaste la versión 1 pero la actual es 2.","versionActual":2}],
 "ruta":"PATCH /api/tareas/1","marcaTiempo":"..."}}
```

**Validación**
```bash
curl -i -b cookies.txt -X POST localhost:3000/api/tareas -H 'content-type: application/json' \
  -d '{"titulo":"x","estado":"raro"}'
```
```
HTTP/1.1 422 Unprocessable Entity
{"error":{"codigo":"VALIDACION",...,"detalles":[
  {"campo":"titulo","mensaje":"Debe tener entre 3 y 120 caracteres."},
  {"campo":"estado","mensaje":"Debe ser uno de: pendiente, en_progreso, completada."}]}}
```

**Adjuntar archivo** (PNG, JPG, PDF o TXT, máx. 2 MB)
```bash
curl -i -b cookies.txt -X POST localhost:3000/api/tareas/1/archivo -F "archivo=@foto.png;type=image/png"
```
```
HTTP/1.1 201 Created
Location: /api/tareas/1/archivo
{"tarea":{"id":1,...,"version":3,"archivo":{"nombreOriginal":"foto.png","tipo":"image/png","bytes":16,
 "url":"/api/tareas/1/archivo"},...}}
```

**Listado filtrado**
```bash
curl -b cookies.txt "localhost:3000/api/tareas?estado=en_progreso&q=informe&limite=5"
```
```
{"datos":[{"id":1,"titulo":"Entregar informe",...}],"paginacion":{"pagina":1,"limite":5,"total":1,"paginas":1}}
```

## Tiempo real (Socket.IO)

El socket reutiliza la cookie de sesión; sin sesión la conexión se rechaza (`NO_AUTENTICADO`). Cada usuario entra a su
propia sala y **solo recibe sus eventos**. Al cerrar sesión, sus sockets se desconectan.

| Evento | Cuándo | Datos |
|---|---|---|
| `conexion:lista` | Al conectar | `{ usuario }` |
| `tarea:creada` | POST /tareas | `{ tarea, marcaTiempo }` |
| `tarea:actualizada` | PUT / PATCH | `{ tarea, marcaTiempo }` |
| `tarea:eliminada` | DELETE | `{ id, marcaTiempo }` |
| `tarea:archivo` | Adjuntar o quitar | `{ tarea, accion: "adjuntado" \| "eliminado", marcaTiempo }` |

```js
import { io } from 'socket.io-client';
const socket = io('http://localhost:3000'); // en navegador, la cookie viaja sola
socket.on('tarea:creada', ({ tarea }) => console.log('Nueva tarea:', tarea.titulo));
```

## Decisiones de seguridad y conflictos

- **Consultas parametrizadas**; el orden y las columnas editables salen de listas blancas; los comodines de `LIKE` se escapan.
- **Bloqueo optimista**: `UPDATE ... WHERE version = ?`; si no coincide responde `409` con la versión actual.
- **Unicidad** `(usuario, título)` sin distinguir mayúsculas → `409 TITULO_DUPLICADO`.
- **Claves** con `scrypt` + sal; el login tarda igual si el usuario no existe y no revela cuál dato falló; límite de intentos → `429`.
- **Sesión**: cookie `HttpOnly` + `SameSite=Lax`, id regenerado al entrar, destruida al salir.
- **Archivos**: lista de tipos, verificación de firma (bytes mágicos), nombre en disco = UUID, límite de tamaño, `nosniff`,
  limpieza del archivo si algo falla.
- **Aislamiento**: todas las consultas filtran por `usuario_id`; lo ajeno responde `404`.

## Pruebas

`npm test` ejecuta 65 pruebas: reglas de dominio puras, todos los endpoints (códigos de éxito y error), aislamiento entre
usuarios, inyección SQL, subida de archivos (tipo, firma, tamaño, limpieza en disco), sesiones y eventos Socket.IO.
La colección manual para REST Client / IntelliJ está en [`requests.http`](requests.http).
