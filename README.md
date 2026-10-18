# Catálogo de equipos de fútbol (Proyecto 2)

Proyecto 2: EIF-508 Sistemas Distribuidos.

**Estudiante:** Isaías Víquez Soto

**Tema asignado:** 14. Catálogo de equipos de fútbol

**URL desplegada:** _pendiente_

## Descripción

Continuación del Proyecto 1. El servicio REST pasó de Cloudflare Workers + D1 a **Netlify Functions + MongoDB Atlas** y se le agregó **autenticación con JWT**. El frontend sigue siendo Vue.js.

Entidades, cada una con su colección en MongoDB:

- **equipos** (principal): nombre, ciudad, país, fundación, escudo, estadio (`estadio_id`) y liga (`liga_id`).
- **estadios** (secundaria): nombre, ciudad, país, capacidad, año de inauguración e imagen.
- **ligas** (secundaria): nombre, país, confederación, nivel, fundación y logo.
- **users**: nombre, correo, contraseña (hash bcrypt) y rol (`user` o `admin`).

## Servicio REST

`netlify.toml` redirige `/api/*` a las funciones de `netlify/functions/`. De esa forma se mantienen las rutas del Proyecto 1:

| Operación | Ruta | Autenticación |
|---|---|---|
| Listar | `GET /api/<entidad>` | Pública |
| Consultar | `GET /api/<entidad>/<id>` | Pública |
| Crear | `POST /api/<entidad>` | Token |
| Editar | `POST /api/<entidad>/<id>` | Token |
| Borrar | `POST /api/<entidad>/<id>` con `{"_method":"DELETE"}` | Token con rol `admin` |
| Registro | `POST /api/register` | — |
| Inicio de sesión | `POST /api/login` (devuelve el JWT) | — |

El token se envía en el encabezado `Authorization: Bearer <token>`. Cada función protegida valida la firma y la expiración del token (2 horas). Responde **401** si el token falta, es inválido o expiró, y **403** si un usuario sin rol `admin` intenta borrar. Además, borrar un estadio o una liga que todavía usa algún equipo responde **409**.

El token se guarda en `localStorage`. El material del curso recomienda cookies HTTP-only, pero JavaScript no puede leer esas cookies, y entonces no podría armar el encabezado `Authorization: Bearer` que exige el enunciado.

## Variables de entorno

| Variable | Uso |
|---|---|
| `MONGODB_URI` | Cadena de conexión de MongoDB Atlas (`mongodb+srv://<usuario>:<contraseña>@<cluster>.mongodb.net/`) |
| `TOKEN_KEY` | Clave secreta para firmar los JWT (HS256). Debe ser una cadena aleatoria larga, de al menos 32 caracteres |
| `ADMIN_EMAIL`, `ADMIN_PASSWORD` | Solo para `scripts/seed.js` (desarrollo local): credenciales del administrador inicial |

**En Netlify:** *Site configuration → Environment variables*. Ahí se agregan `MONGODB_URI` y `TOKEN_KEY` y luego se vuelve a desplegar.

**En desarrollo local:** se crea un archivo `.env` en la raíz con las cuatro variables. Git lo ignora y no forma parte de la entrega. Para generar una `TOKEN_KEY`:

```
node -e "console.log(require('crypto').randomBytes(48).toString('base64url'))"
```

**En MongoDB Atlas:**

- *Database Access*: un usuario de base de datos con el permiso `readWrite` sobre `catalogo-futbol`.
- *Network Access*: `0.0.0.0/0`. Netlify Functions no tiene IPs fijas, así que hay que permitir cualquier origen.

## Desarrollo local

Se necesita Netlify CLI (`npm install -g netlify-cli`).

```
npm install
npm run seed      # carga los datos del Proyecto 1 y el usuario admin en Atlas
netlify dev       # SPA + funciones en http://localhost:8888
```

Igual que en el tutorial, el frontend llama al backend publicado: la URL está en `app.config.globalProperties.url` (`src/main.js`) y cada llamada usa `this.url + '/api/...'`. Para probar solo las funciones locales, se pueden llamar con curl en `http://localhost:8888/api/...`.

`npm run build` genera el sitio en `dist/`, que es el directorio que publica Netlify.
