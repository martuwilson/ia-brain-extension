# Templates de Código — Referencia Completa

Estos son los templates base. Claude los adapta según el proyecto del usuario
(nombre, entidades, base de datos elegida). Los comentarios didácticos son OBLIGATORIOS.

---

## `package.json`

El `package.json` generado NO incluye números de versión fijos.
Las versiones las resuelve `npm install` en tiempo real, trayendo siempre la última estable.

```json
{
  "name": "{{PROJECT_NAME}}",
  "version": "1.0.0",
  "description": "REST API — Node.js 20 + Express",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js",
    "dev": "nodemon src/index.js",
    "audit": "npm audit --audit-level=moderate"
  },
  "dependencies": {
    "cors": "latest",
    "dotenv": "latest",
    "express": "latest",
    "mongoose": "latest",
    "morgan": "latest"
  },
  "devDependencies": {
    "nodemon": "latest"
  },
  "engines": {
    "node": ">=20.0.0"
  }
}
```

Agregar según el caso:
- JWT + bcrypt: `"jsonwebtoken": "latest"` y `"bcryptjs": "latest"`
- PostgreSQL: `"pg": "latest"` en lugar de `"mongoose": "latest"`

NUNCA poner versiones hardcodeadas — pueden tener vulnerabilidades conocidas al momento de generar.

---

## `src/index.js`

```js
/**
 * 📁 src/index.js — Entry Point
 *
 * Este es el archivo que Node.js ejecuta primero cuando corres `node src/index.js`.
 * Su única responsabilidad es:
 *   1. Cargar las variables de entorno desde el archivo .env
 *   2. Conectarse a la base de datos
 *   3. Levantar el servidor HTTP en el puerto configurado
 *
 * La configuración de Express (rutas, middlewares, etc.) está en app.js.
 * Separar index.js de app.js facilita hacer tests unitarios de la app
 * sin tener que levantar un servidor real.
 */

// dotenv debe cargarse ANTES que cualquier otra importación que use process.env
require('dotenv').config();

const app = require('./app');
const connectDB = require('./config/db');

const PORT = process.env.PORT || 3000;

// Conectar a la base de datos y luego levantar el servidor.
// Si la DB falla, el servidor NO arranca — mejor fallar rápido que funcionar a medias.
connectDB()
  .then(() => {
    app.listen(PORT, () => {
      console.log(`✅ Servidor corriendo en http://localhost:${PORT}`);
      console.log(`📖 Entorno: ${process.env.NODE_ENV || 'development'}`);
    });
  })
  .catch((err) => {
    console.error('❌ Error al conectar a la base de datos:', err.message);
    process.exit(1); // Salir con código de error
  });
```

---

## `src/app.js`

```js
/**
 * 📁 src/app.js — Configuración de Express
 *
 * Aquí se configura la instancia de Express:
 *   - Middlewares globales (aplican a TODAS las rutas)
 *   - Montaje del router principal
 *   - Middleware de manejo de errores (siempre al final)
 *
 * ¿Qué es un middleware?
 * Es una función que se ejecuta ENTRE que llega el request y que se envía la respuesta.
 * Puede modificar el request, la respuesta, o terminar el ciclo.
 * Express los ejecuta en orden, de arriba hacia abajo.
 */

const express = require('express');
const cors = require('cors');
const morgan = require('morgan');

const routes = require('./routes');
const errorMiddleware = require('./middlewares/error.middleware');

const app = express();

// ─── Middlewares Globales ──────────────────────────────────────────────────────

// CORS: permite que otras aplicaciones (ej: un frontend React, un notebook Jupyter)
// puedan hacer requests a esta API. Sin esto, el browser bloquea las requests.
app.use(cors());

// express.json(): parsea el body de los requests con Content-Type: application/json
// Sin esto, req.body sería undefined en los POST/PUT.
app.use(express.json());

// Morgan: logger de requests. En 'dev' mode imprime algo como:
// GET /api/v1/users 200 12.345 ms - 256
app.use(morgan('dev'));

// ─── Rutas ────────────────────────────────────────────────────────────────────

// Todas las rutas de la API están bajo /api/v1
// El prefijo /v1 permite versionar la API en el futuro (v2, v3...)
// sin romper clientes que usan la versión anterior.
app.use('/api/v1', routes);

// Ruta de health check — útil para monitoreo y para verificar que el server responde
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok', timestamp: new Date().toISOString() });
});

// ─── Error Middleware ─────────────────────────────────────────────────────────

// IMPORTANTE: el error middleware DEBE ir al final, después de todas las rutas.
// Express lo identifica como error middleware porque tiene 4 parámetros: (err, req, res, next).
// Se activa cuando cualquier controller hace: next(error)
app.use(errorMiddleware);

module.exports = app;
```

---

## `src/routes/index.js`

```js
/**
 * 📁 src/routes/index.js — Router Raíz
 *
 * Este archivo es el "panel de control" de todas las rutas de la API.
 * Cada vez que se agrega un nuevo recurso al proyecto, se registra aquí.
 *
 * Flujo hasta llegar aquí:
 *   Request → index.js → app.js → [este archivo]
 *
 * Desde aquí, el request se redirige al router específico del recurso.
 */

const { Router } = require('express');

// Importar los routers de cada recurso
const usersRouter = require('./users.routes');
// const predictionsRouter = require('./predictions.routes'); // ← agregar nuevos recursos acá

const router = Router();

// Montar cada router en su path correspondiente.
// Cuando llegue una request a /api/v1/users/..., Express la delegará a usersRouter.
router.use('/users', usersRouter);
// router.use('/predictions', predictionsRouter);

module.exports = router;
```

---

## `src/routes/users.routes.js`

```js
/**
 * 📁 src/routes/users.routes.js — Rutas del recurso "users"
 *
 * Define el mapeo entre cada URL + método HTTP y la función del controller
 * que debe manejarla. Este archivo NO contiene lógica de negocio.
 *
 * Flujo hasta llegar aquí:
 *   Request → index.js → app.js → routes/index.js → [este archivo]
 *
 * Desde aquí, el request pasa al controller (y antes, si aplica, por middlewares).
 *
 * Rutas definidas:
 *   GET    /api/v1/users          → getAll     (listar todos los usuarios)
 *   GET    /api/v1/users/:id      → getById    (obtener un usuario por ID)
 *   POST   /api/v1/users          → create     (crear un usuario)
 *   PUT    /api/v1/users/:id      → update     (actualizar un usuario)
 *   DELETE /api/v1/users/:id      → remove     (eliminar un usuario)
 */

const { Router } = require('express');
const {
  getAll,
  getById,
  create,
  update,
  remove,
} = require('../controllers/users.controller');
const authMiddleware = require('../middlewares/auth.middleware');

const router = Router();

// ─── Rutas públicas (no requieren token) ──────────────────────────────────────

// POST /api/v1/users → Registro de usuario (público: el usuario aún no tiene token)
router.post('/', create);

// ─── Rutas protegidas (requieren token JWT válido) ────────────────────────────

// 🔒 Todo lo que esté debajo de esta línea requiere autenticación.
// authMiddleware verifica el token y adjunta req.user antes de llegar al controller.
router.use(authMiddleware);

router.get('/', getAll);          // GET /api/v1/users
router.get('/:id', getById);      // GET /api/v1/users/64a1b2c3d4e5f6a7b8c9d0e1
router.put('/:id', update);       // PUT /api/v1/users/64a1b2c3d4e5f6a7b8c9d0e1
router.delete('/:id', remove);    // DELETE /api/v1/users/64a1b2c3d4e5f6a7b8c9d0e1

module.exports = router;
```

---

## `src/controllers/users.controller.js`

```js
/**
 * 📁 src/controllers/users.controller.js — Controlador de "users"
 *
 * Los controllers son el corazón de cada endpoint. Su trabajo es:
 *   1. Extraer los datos del request (body, params, query, user autenticado)
 *   2. Llamar al model o service que hace el trabajo real
 *   3. Enviar la respuesta HTTP correcta (status code + JSON)
 *   4. Si algo falla, pasar el error a next() para que lo maneje el error middleware
 *
 * Flujo hasta llegar aquí:
 *   Request → ... → routes/users.routes.js → [este archivo]
 *
 * REGLA: un controller NUNCA debería tener más de ~30 líneas por función.
 * Si crece más, extraer lógica a un service.
 */

const User = require('../models/user.model');
const bcrypt = require('bcryptjs');

// ─── GET /api/v1/users ────────────────────────────────────────────────────────
const getAll = async (req, res, next) => {
  try {
    // req.query permite filtros opcionales: GET /users?role=admin
    const users = await User.find().select('-password'); // nunca devolver el password
    res.status(200).json({ data: users, total: users.length });
  } catch (error) {
    // next(error) pasa el error al errorMiddleware, evitando repetir código de manejo de errores
    next(error);
  }
};

// ─── GET /api/v1/users/:id ────────────────────────────────────────────────────
const getById = async (req, res, next) => {
  try {
    // req.params.id es el valor que viene en la URL: /users/64a1b2c3...
    const user = await User.findById(req.params.id).select('-password');

    if (!user) {
      // Responder 404 si no existe, NO dejar que el catch lo maneje
      return res.status(404).json({ error: 'Usuario no encontrado' });
    }

    res.status(200).json({ data: user });
  } catch (error) {
    next(error);
  }
};

// ─── POST /api/v1/users ───────────────────────────────────────────────────────
const create = async (req, res, next) => {
  try {
    const { name, email, password } = req.body;

    // Validación básica — en producción conviene usar una librería como Joi o Zod
    if (!name || !email || !password) {
      return res.status(400).json({ error: 'name, email y password son requeridos' });
    }

    // Nunca guardar el password en texto plano. bcrypt lo hashea con salt.
    const hashedPassword = await bcrypt.hash(password, 10);

    const newUser = await User.create({ name, email, password: hashedPassword });

    // Crear una versión del usuario sin el password para devolverla
    const userResponse = newUser.toObject();
    delete userResponse.password;

    // 201 Created — no 200 — cuando se crea un recurso nuevo
    res.status(201).json({ data: userResponse, message: 'Usuario creado correctamente' });
  } catch (error) {
    // Mongoose lanza error con code 11000 cuando hay un campo único duplicado (ej: email)
    if (error.code === 11000) {
      return res.status(400).json({ error: 'El email ya está registrado' });
    }
    next(error);
  }
};

// ─── PUT /api/v1/users/:id ────────────────────────────────────────────────────
const update = async (req, res, next) => {
  try {
    const { name, email } = req.body;

    // new: true → devuelve el documento actualizado (no el original)
    // runValidators: true → aplica las validaciones del schema de Mongoose
    const updated = await User.findByIdAndUpdate(
      req.params.id,
      { name, email },
      { new: true, runValidators: true }
    ).select('-password');

    if (!updated) {
      return res.status(404).json({ error: 'Usuario no encontrado' });
    }

    res.status(200).json({ data: updated });
  } catch (error) {
    next(error);
  }
};

// ─── DELETE /api/v1/users/:id ─────────────────────────────────────────────────
const remove = async (req, res, next) => {
  try {
    const deleted = await User.findByIdAndDelete(req.params.id);

    if (!deleted) {
      return res.status(404).json({ error: 'Usuario no encontrado' });
    }

    // 204 No Content — el estándar para DELETE exitoso, no lleva body
    res.status(204).send();
  } catch (error) {
    next(error);
  }
};

module.exports = { getAll, getById, create, update, remove };
```

---

## `src/middlewares/auth.middleware.js`

```js
/**
 * 📁 src/middlewares/auth.middleware.js — Verificación de JWT
 *
 * Este middleware intercepta requests que requieren autenticación.
 * Se ejecuta ANTES del controller en las rutas protegidas.
 *
 * ¿Cómo funciona JWT?
 *   1. El cliente hace login → el server genera un token firmado y lo devuelve
 *   2. El cliente guarda el token y lo envía en cada request posterior:
 *      Header: Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI...
 *   3. Este middleware verifica que el token sea válido y no haya expirado
 *   4. Si es válido, adjunta el payload decodificado a req.user
 *   5. El controller puede entonces usar req.user.id, req.user.email, etc.
 *
 * Si el token no existe o es inválido: responde 401 y el controller NO se ejecuta.
 */

const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  // El token viene en el header: "Authorization: Bearer <token>"
  const authHeader = req.headers['authorization'];

  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Token no proporcionado' });
  }

  // Extraer solo el token (sin el prefijo "Bearer ")
  const token = authHeader.split(' ')[1];

  try {
    // jwt.verify lanza una excepción si el token es inválido o expiró
    const decoded = jwt.verify(token, process.env.JWT_SECRET);

    // Adjuntar el payload decodificado al objeto request
    // A partir de acá, cualquier controller puede acceder a req.user
    req.user = decoded;

    // next() le dice a Express que continúe al siguiente middleware o controller
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Token inválido o expirado' });
  }
};

module.exports = authMiddleware;
```

---

## `src/middlewares/error.middleware.js`

```js
/**
 * 📁 src/middlewares/error.middleware.js — Manejo Centralizado de Errores
 *
 * Cuando un controller hace next(error), Express busca el middleware con
 * 4 parámetros (err, req, res, next) y lo ejecuta. Este es ese middleware.
 *
 * Beneficio: un solo lugar para manejar todos los errores.
 * Sin esto, cada controller debería tener su propio bloque de respuesta de error.
 *
 * IMPORTANTE: debe registrarse en app.js AL FINAL, después de todas las rutas.
 */

const errorMiddleware = (err, req, res, next) => {
  // Loguear el error completo (stack trace) para debugging
  console.error('❌ Error:', err.message);
  if (process.env.NODE_ENV === 'development') {
    console.error(err.stack);
  }

  // Usar el status code del error si fue definido, o 500 por defecto
  const statusCode = err.statusCode || err.status || 500;

  res.status(statusCode).json({
    error: err.message || 'Error interno del servidor',
    // Solo mostrar el stack en desarrollo, nunca en producción
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack }),
  });
};

module.exports = errorMiddleware;
```

---

## `src/models/user.model.js` (Mongoose)

```js
/**
 * 📁 src/models/user.model.js — Schema de Mongoose para "User"
 *
 * Un Schema de Mongoose define la estructura de los documentos en MongoDB:
 *   - Qué campos tiene cada documento
 *   - De qué tipo es cada campo
 *   - Qué validaciones aplican
 *   - Qué índices existen (para búsquedas rápidas)
 *
 * El Model es la interfaz para interactuar con la colección:
 *   User.find(), User.create(), User.findById(), etc.
 */

const mongoose = require('mongoose');

const userSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: [true, 'El nombre es requerido'],
      trim: true, // elimina espacios al inicio y final
    },
    email: {
      type: String,
      required: [true, 'El email es requerido'],
      unique: true,       // índice único en MongoDB
      lowercase: true,    // guarda siempre en minúsculas
      trim: true,
    },
    password: {
      type: String,
      required: [true, 'El password es requerido'],
      minlength: [6, 'El password debe tener al menos 6 caracteres'],
    },
    role: {
      type: String,
      enum: ['user', 'admin'], // solo estos valores son válidos
      default: 'user',
    },
  },
  {
    // timestamps: true agrega automáticamente createdAt y updatedAt a cada documento
    timestamps: true,
  }
);

// Model.find(), Model.create(), etc. usan este schema.
// 'User' es el nombre del modelo → MongoDB crea la colección 'users' (en plural, automáticamente)
const User = mongoose.model('User', userSchema);

module.exports = User;
```

---

## `src/config/db.js` (MongoDB / Mongoose)

```js
/**
 * 📁 src/config/db.js — Conexión a Base de Datos
 *
 * Centraliza la lógica de conexión a MongoDB.
 * Se llama una sola vez desde index.js al iniciar el servidor.
 *
 * La URI de conexión viene de la variable de entorno DB_URI.
 * Ejemplo: mongodb://localhost:27017/myapp
 * O Atlas: mongodb+srv://user:pass@cluster.mongodb.net/myapp
 */

const mongoose = require('mongoose');

const connectDB = async () => {
  const uri = process.env.DB_URI;

  if (!uri) {
    throw new Error('La variable de entorno DB_URI no está definida');
  }

  await mongoose.connect(uri);
  console.log(`✅ MongoDB conectado: ${mongoose.connection.host}`);
};

module.exports = connectDB;
```

---

## `src/config/db.js` (PostgreSQL / pg)

```js
/**
 * 📁 src/config/db.js — Conexión a PostgreSQL
 *
 * Pool de conexiones a PostgreSQL usando la librería 'pg'.
 * Un Pool reutiliza conexiones en lugar de abrir/cerrar una por request,
 * lo que mejora enormemente el rendimiento.
 */

const { Pool } = require('pg');

const pool = new Pool({
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT || '5432'),
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
});

// Función de conexión para verificar al arrancar
const connectDB = async () => {
  const client = await pool.connect();
  console.log(`✅ PostgreSQL conectado: ${process.env.DB_HOST}/${process.env.DB_NAME}`);
  client.release(); // devolver el cliente al pool
  return pool;
};

// Exportar tanto connectDB como pool
// El pool se usa en los models: const { rows } = await pool.query('SELECT ...')
module.exports = connectDB;
module.exports.pool = pool;
```

---

## `.env.example`

```env
# ─── Servidor ─────────────────────────────────────────────────────────────────
PORT=3000
NODE_ENV=development   # development | production

# ─── Base de Datos ────────────────────────────────────────────────────────────
# MongoDB:
DB_URI=mongodb://localhost:27017/{{PROJECT_NAME}}

# PostgreSQL (descomentar si usás pg):
# DB_HOST=localhost
# DB_PORT=5432
# DB_NAME={{PROJECT_NAME}}
# DB_USER=postgres
# DB_PASSWORD=tu_password_aqui

# ─── JWT ──────────────────────────────────────────────────────────────────────
# IMPORTANTE: Cambiar por una clave aleatoria larga antes de producción
# Generar con: node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
JWT_SECRET=cambiar_esto_por_una_clave_segura_minimo_32_caracteres
JWT_EXPIRES_IN=7d
```

---

## `.gitignore`

```
# Dependencias
node_modules/

# Variables de entorno — NUNCA subir el .env al repositorio
.env

# Logs
*.log
npm-debug.log*

# Sistema operativo
.DS_Store
Thumbs.db

# IDEs
.vscode/
.idea/
```

---

## Template del script generador (`generate-backend.js`)

El script generador tiene tres fases:
1. Crear archivos
2. Correr `npm install` con las últimas versiones estables
3. Correr `npm audit` y reportar el resultado

```js
#!/usr/bin/env node
/**
 * generate-backend.js
 *
 * Corre este script con: node generate-backend.js
 * Genera el proyecto, instala dependencias y verifica vulnerabilidades.
 */

const fs = require('fs');
const path = require('path');
const { execSync } = require('child_process'); // para correr comandos de terminal

const PROJECT_NAME = '{{PROJECT_NAME}}';
const BASE_DIR = path.join(process.cwd(), PROJECT_NAME);

// ── Fase 1: Crear archivos ────────────────────────────────────────────────────

const files = {
  'package.json': `{ ... }`,  // con "latest" en todas las dependencias
  'src/index.js': `...`,
  // ... todos los archivos
};

Object.entries(files).forEach(([filePath, content]) => {
  const fullPath = path.join(BASE_DIR, filePath);
  fs.mkdirSync(path.dirname(fullPath), { recursive: true });
  fs.writeFileSync(fullPath, content, 'utf8');
  console.log(`  ✅ ${filePath}`);
});

// ── Fase 2: npm install ───────────────────────────────────────────────────────

console.log('\n📦 Instalando dependencias en su última versión estable...\n');
try {
  // --registry fuerza el registry público oficial de npm.
  // Evita que proxies corporativos (Nexus, Artifactory, etc.) interfieran
  // con la instalación de paquetes públicos.
  execSync('npm install --registry https://registry.npmjs.org', {
    cwd: BASE_DIR,
    stdio: 'inherit',
  });
  console.log('\n✅ Dependencias instaladas correctamente.\n');
} catch (err) {
  console.error('❌ Error al instalar dependencias:', err.message);
  console.error('\n   Posibles causas:');
  console.error('   1. Sin conexión a internet');
  console.error('   2. Firewall corporativo bloqueando registry.npmjs.org');
  console.error('      → Intentá conectarte a la VPN o pedile a IT el registry npm correcto');
  console.error('   3. Proxy corporativo (Nexus, Artifactory) requiere autenticación');
  console.error('      → Consultá con IT cuál es el registry npm habilitado en tu empresa\n');
  process.exit(1);
}

// ── Fase 3: npm audit ─────────────────────────────────────────────────────────

console.log('🔍 Verificando vulnerabilidades conocidas (npm audit)...\n');
try {
  // --audit-level=moderate: falla solo si hay vulnerabilidades de severidad media o mayor
  // --registry también acá: npm audit consulta la DB de advisories de npm,
  // que también puede ser bloqueada por proxies corporativos
  execSync('npm audit --audit-level=moderate --registry https://registry.npmjs.org', {
    cwd: BASE_DIR,
    stdio: 'inherit',
  });
  console.log('\n✅ Sin vulnerabilidades moderadas o críticas detectadas.\n');
} catch {
  // npm audit devuelve exit code != 0 si encuentra vulnerabilidades
  // No cortamos el proceso — avisamos y dejamos que el usuario decida
  console.warn('\n⚠️  Se detectaron vulnerabilidades. Revisá el reporte de arriba.');
  console.warn('   Podés correr "npm audit fix" dentro de la carpeta para intentar resolverlas.');
  console.warn('   O "npm audit" para ver el detalle completo.\n');
}

// ── Instrucciones finales ─────────────────────────────────────────────────────

console.log(`
╔═══════════════════════════════════════════════════════════════════╗
║  ✅ Proyecto listo: ./{{PROJECT_NAME}}
╠═══════════════════════════════════════════════════════════════════╣
║                                                                   ║
║  QUÉ HACER AHORA:                                                 ║
║                                                                   ║
║  1. cd {{PROJECT_NAME}}                                           ║
║  2. cp .env.example .env    ← completar credenciales              ║
║  3. npm run dev             ← levantar el servidor                ║
║     (npm install ya se corrió automáticamente)                    ║
║                                                                   ║
║  Para verificar que funciona:                                     ║
║  → Abrí http://localhost:3000/health en el browser                ║
║                                                                   ║
║  Si hubo vulnerabilidades arriba:                                 ║
║  → npm audit fix            ← intenta resolverlas automáticamente ║
║  → npm audit                ← ver detalle completo                ║
╚═══════════════════════════════════════════════════════════════════╝
`);
```

### Reglas para el script generador

- El `package.json` SIEMPRE usa `"latest"` en todas las dependencias, nunca versiones fijas
- `npm install` se corre desde el script — el usuario NO necesita correrlo manualmente
- `npm audit` se corre siempre después del install, sin excepción
- Si `npm audit` encuentra vulnerabilidades, el script AVISA pero NO aborta — deja decidir al usuario
- El bloque de instrucciones finales menciona explícitamente que `npm install` ya se corrió
