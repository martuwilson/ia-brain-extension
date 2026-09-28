# Arquitectura del Backend — Referencia

Este documento explica el "por qué" de cada decisión de arquitectura.
Claude debe leerlo para entender el diseño y poder responder preguntas del equipo.

---

## Flujo de un Request

```
Cliente (browser/Python/Postman)
    │
    ▼
src/index.js          ← Inicia el servidor, llama a app.js
    │
    ▼
src/app.js            ← Configura middlewares globales (cors, json parser, morgan)
    │
    ▼
src/routes/index.js   ← Router raíz: monta /api/v1/users, /api/v1/predictions, etc.
    │
    ▼
src/routes/users.routes.js    ← Define GET /users, POST /users, GET /users/:id ...
    │
    ├── [Si la ruta requiere auth] src/middlewares/auth.middleware.js
    │
    ▼
src/controllers/users.controller.js   ← Lógica del endpoint
    │
    ├── [Si hay lógica de negocio compleja] src/services/users.service.js
    │
    ▼
src/models/user.model.js   ← Interacción con la base de datos
    │
    ▼
Respuesta JSON al cliente
```

---

## Responsabilidad de cada capa

### `index.js` — Entry Point
- **Solo hace una cosa**: levantar el servidor en el puerto configurado
- Importa `app.js` y llama a `app.listen()`
- No configura rutas ni middlewares — eso es trabajo de `app.js`
- Maneja el evento de conexión a base de datos antes de levantar el servidor

### `app.js` — Configuración de Express
- Crea la instancia de Express (`const app = express()`)
- Registra middlewares GLOBALES (aplican a todas las rutas):
  - `cors()` — permite requests desde otros orígenes (frontend, Jupyter, etc.)
  - `express.json()` — parsea el body de los requests como JSON
  - `morgan('dev')` — imprime cada request en consola para debugging
- Monta el router raíz: `app.use('/api/v1', routes)`
- Registra el error middleware AL FINAL (importante: debe ser el último)
- Exporta `app` para que `index.js` lo use y para los tests

### `routes/index.js` — Router Raíz
- Único punto donde se registran todos los sub-routers
- Si se agrega un nuevo recurso, solo se toca este archivo
- Ejemplo: `router.use('/users', usersRouter)`

### `routes/<resource>.routes.js` — Rutas de un recurso
- Define qué función del controller maneja cada combinación URL + método HTTP
- NO contiene lógica de negocio, solo el mapeo
- Puede aplicar middlewares específicos de esa ruta (ej: solo algunas rutas requieren auth)

### `controllers/<resource>.controller.js` — Controladores
- Recibe `(req, res, next)` de Express
- Extrae datos del request: `req.params`, `req.body`, `req.query`, `req.user`
- Llama al service o model correspondiente
- Maneja la respuesta HTTP: status code + JSON
- Usa siempre try/catch y pasa errores a `next(error)`
- NO contiene SQL, queries de Mongoose, ni lógica de negocio compleja

### `middlewares/auth.middleware.js` — Autenticación JWT
- Intercepta requests antes de llegar al controller
- Verifica que el header `Authorization: Bearer <token>` exista y sea válido
- Si el token es válido, adjunta el usuario decodificado a `req.user`
- Si no, responde 401 sin llegar al controller

### `middlewares/error.middleware.js` — Manejo Centralizado de Errores
- Express lo llama automáticamente cuando un controller hace `next(error)`
- Evita repetir el manejo de errores en cada controller
- Loguea el error y responde con un formato JSON consistente

### `models/<resource>.model.js` — Modelos
- Define el esquema de datos (con Mongoose) o las queries (con pg)
- Es la única capa que sabe cómo se guardan los datos
- Los controllers no deberían importar nada de `db` directamente

### `config/db.js` — Conexión a Base de Datos
- Centraliza la lógica de conexión
- Se llama una sola vez desde `index.js` al arrancar
- Si la conexión falla, el servidor no debería levantarse

### `.env.example` — Variables de Entorno
- Lista TODAS las variables que necesita la app, con valores de ejemplo o descripción
- El `.env` real nunca se sube al repo (está en `.gitignore`)
- Cada dev crea su propio `.env` copiando el `.env.example`

---

## Patrones de respuesta HTTP

### Success responses
```js
// GET (colección)
res.status(200).json({ data: [...], total: n });

// GET (uno), PUT
res.status(200).json({ data: {...} });

// POST (creación)
res.status(201).json({ data: {...}, message: 'Created successfully' });

// DELETE
res.status(204).send(); // Sin body
```

### Error responses
```js
// Recurso no encontrado
res.status(404).json({ error: 'User not found' });

// Datos inválidos
res.status(400).json({ error: 'Email is required' });

// Sin autorización
res.status(401).json({ error: 'Invalid or missing token' });
```

---

## Estructura de JWT

El payload del token debería contener solo lo necesario:
```js
{
  id: user._id,       // Para buscar el usuario en la DB
  email: user.email,  // Para logging/auditoría
  role: user.role     // Para autorización por rol (si aplica)
}
```

El token se firma con `JWT_SECRET` (variable de entorno) y tiene expiración (`JWT_EXPIRES_IN`).

---

## Variables de entorno estándar

```env
# Servidor
PORT=3000
NODE_ENV=development

# Base de datos
DB_URI=mongodb://localhost:27017/myapp
# o para PostgreSQL:
# DB_HOST=localhost
# DB_PORT=5432
# DB_NAME=myapp
# DB_USER=postgres
# DB_PASSWORD=secret

# JWT
JWT_SECRET=una-clave-muy-larga-y-aleatoria-minimo-32-chars
JWT_EXPIRES_IN=7d
```

---

## Preguntas frecuentes del equipo

**¿Por qué separar routes y controllers?**
Un route es solo el "mapa de rutas". El controller es la lógica. Si mañana cambian las URLs
o el framework, solo tocás los routes — los controllers no cambian.

**¿Cuándo usar services?**
Cuando la lógica de un controller se vuelve larga o se repite en varios controllers.
Por ejemplo, "enviar email de confirmación" puede necesitarse en POST /users y POST /auth/register.
Se extrae a `services/email.service.js` y ambos controllers lo importan.

**¿Por qué async/await y no callbacks?**
Los callbacks llevan a "callback hell" y son difíciles de debuggear. async/await hace que
el código asincrónico parezca sincrónico — mucho más familiar para alguien que viene de Python.

**¿Cómo agrego un nuevo recurso?**
1. Crear `routes/predictions.routes.js`
2. Crear `controllers/predictions.controller.js`
3. Crear `models/prediction.model.js` (si aplica)
4. Agregar `router.use('/predictions', predictionsRouter)` en `routes/index.js`
Solo 4 pasos, siempre los mismos.
