---
name: node-backend-scaffold
description: >
  Genera un esqueleto de backend Node.js / Express completo, didáctico y con buenas prácticas,
  orientado a equipos de Data Science o Data Engineering que necesitan exponer APIs rápidamente
  sin experiencia previa en backend. Usa esta skill siempre que el usuario pida: crear un backend,
  armar una API REST, generar endpoints, estructurar un proyecto Node/Express, crear rutas o
  controladores, hacer un servidor con Express, o cualquier variante de "quiero levantar un back".
  También úsala si mencionan JWT, middlewares, conexión a base de datos con Node, o si preguntan
  cómo organizar un proyecto backend en JavaScript/TypeScript.
---

# Node Backend Scaffold Skill

Genera proyectos backend Node.js 20+ / Express completamente funcionales, con estructura
profesional, comentarios educativos en cada archivo, y un script generador listo para correr.

## Público objetivo

Equipos de Data Science / Data Engineering con conocimientos sólidos de Python/SQL pero
sin experiencia real en arquitectura backend. El código generado debe ser:

- **Funcional**: corre sin modificaciones después de `npm install`
- **Didáctico**: cada archivo explica su propio rol con comentarios
- **Escalable**: la estructura permite crecer sin refactorizar
- **Profesional**: sigue convenciones reales de la industria

---

## Proceso de generación

### Paso 1 — Entender el dominio del proyecto

Si el usuario no lo especificó, pregunta UNA SOLA VEZ:
- ¿Cuál es el nombre del proyecto? (para nombrar carpetas y package.json)
- ¿Qué recursos/entidades principales va a manejar? (ej: "usuarios, modelos ML, predicciones")
- ¿Base de datos? (MongoDB con Mongoose / PostgreSQL con pg / ninguna por ahora)

Si el usuario quiere arrancar rápido con un ejemplo genérico, NO preguntes — generá el
proyecto con entidades de ejemplo (`users`, `predictions`) y avisá que puede cambiarlo.

### Paso 2 — Generar el script

Lee `references/structure.md` para entender la arquitectura completa.
Lee `references/templates.md` para obtener el código de cada archivo.

Genera UN ÚNICO archivo JavaScript: `generate-backend.js`.

Este script, al correrlo con `node generate-backend.js`, debe:
1. Crear toda la estructura de carpetas
2. Escribir todos los archivos con su contenido completo
3. Imprimir instrucciones claras al final ("Próximos pasos")

### Paso 3 — Presentar al usuario

Entregá el archivo `generate-backend.js` para descargar.

Luego escribí en el chat una guía paso a paso completa, como si el usuario nunca tocó
una terminal de Node.js. Usar este formato exacto:

---

**Descargaste `generate-backend.js`. Ahora hacé esto:**

**Paso 1 — Abrí una terminal en la carpeta donde descargaste el archivo**
> En VS Code: menú Terminal → New Terminal
> En Mac: buscá "Terminal" en Spotlight
> En Windows: click derecho en la carpeta → "Abrir en Terminal"

**Paso 2 — Corré el script generador**
```bash
node generate-backend.js
```
Esto crea la carpeta `<nombre-proyecto>/` con todos los archivos adentro.

**Paso 3 — Entrá a la carpeta del proyecto**
```bash
cd <nombre-proyecto>
```

**Paso 4 — Copiá el archivo de variables de entorno**
```bash
cp .env.example .env
```
Después abrí el `.env` con cualquier editor y completá los datos de tu base de datos
(host, nombre de la DB, usuario, password).

**Paso 5 — Instalá las dependencias**
```bash
npm install
```
Esto descarga los paquetes que necesita el proyecto (Express, mysql2, etc.).
Solo hace falta correrlo una vez.

**Paso 6 — Levantá el servidor**
```bash
npm run dev
```
Si todo está bien, vas a ver en la terminal:
```
✅ MySQL conectado: localhost/tu_base
✅ Servidor corriendo en http://localhost:3000
```

**Para probar que funciona:**
Abrí el browser y entrá a: `http://localhost:3000/health`
Deberías ver: `{ "status": "ok" }`

Si ves eso, el servidor está corriendo. 🎉

---

Después de la guía, agregá UNA sola línea indicando qué archivo editar primero
según el caso de uso (generalmente el model para adaptar el nombre de la tabla).

### Regla crítica — el script también se explica solo

El script `generate-backend.js` debe imprimir en la terminal, al finalizar,
la misma guía de pasos pero en formato consola. Así el usuario tiene el contexto
tanto en el chat de Claude como en su propia terminal.

El bloque final del script debe verse así:

```
╔══════════════════════════════════════════════════════════════╗
║  ✅ 11 archivos generados en: ./data-api
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  QUÉ HACER AHORA:                                            ║
║                                                              ║
║  1. cd data-api                                              ║
║  2. cp .env.example .env   ← completar con tus credenciales  ║
║  3. npm install            ← instalar dependencias (1 vez)   ║
║  4. npm run dev            ← levantar el servidor            ║
║                                                              ║
║  Para verificar que funciona:                                ║
║  → Abrí http://localhost:3000/health en el browser           ║
║                                                              ║
║  ⚠️  Antes de correr: editá src/models/records.model.js      ║
║     y cambiá "records" por el nombre real de tu tabla        ║
╚══════════════════════════════════════════════════════════════╝
```

---

## Reglas de generación — SIEMPRE respetar

### Estructura de carpetas obligatoria

```
<project-name>/
├── src/
│   ├── index.js              ← Entry point. Arranca el servidor
│   ├── app.js                ← Configuración de Express (middlewares globales)
│   ├── routes/               ← Solo define rutas. Nada de lógica
│   │   └── index.js          ← Router raíz que monta todos los sub-routers
│   ├── controllers/          ← Lógica de cada endpoint (un archivo por recurso)
│   ├── middlewares/          ← Funciones que interceptan requests
│   │   ├── auth.middleware.js
│   │   └── error.middleware.js
│   ├── models/               ← Esquemas de base de datos (si aplica)
│   ├── services/             ← Lógica de negocio reutilizable (opcional, avanzado)
│   └── config/               ← Configuración centralizada
│       └── db.js             ← Conexión a base de datos
├── .env.example              ← Variables requeridas (sin valores reales)
├── .gitignore
└── package.json
```

### Convenciones de código — NUNCA violar

- **Un archivo por recurso** en `routes/` y `controllers/`. Si hay `users`, existen
  `routes/users.routes.js` Y `controllers/users.controller.js`
- **Controllers no acceden a `req`/`res` en los services**. Los controllers traducen
  HTTP → lógica; los services son HTTP-agnósticos
- **Siempre async/await**, nunca callbacks
- **Siempre try/catch en controllers**, errores pasan al error middleware con `next(error)`
- **Rutas en plural y kebab-case**: `/api/v1/users`, `/api/v1/ml-predictions`
- **HTTP status codes correctos**: 200 GET, 201 POST, 204 DELETE, 400 bad request, 401 unauth, 404 not found, 500 server error
- **Variables de entorno siempre desde `process.env`**, nunca hardcodeadas
- **El .env nunca va al repo** — solo `.env.example`

### Comentarios didácticos — SIEMPRE incluir

Cada archivo debe tener un bloque de comentario al inicio explicando su rol:

```js
/**
 * 📁 routes/users.routes.js
 * 
 * Define las rutas del recurso "users".
 * Este archivo NO contiene lógica de negocio — solo mapea cada
 * URL + método HTTP a la función correspondiente del controller.
 * 
 * Flujo:  Request → index.js → app.js → routes/index.js
 *                → users.routes.js → users.controller.js
 */
```

Y comentarios inline en puntos clave:

```js
// 🔒 Todas las rutas debajo de esta línea requieren token JWT válido
router.use(authMiddleware);

// GET /api/v1/users → users.controller.getAll
router.get('/', getAll);
```

Los emojis son opcionales pero ayudan a la navegación visual para perfiles no-backend.

---

## Seguridad de dependencias — REGLAS OBLIGATORIAS

### Versiones: siempre `latest`, nunca hardcodeadas

El `package.json` generado usa `"latest"` en TODAS las dependencias, sin excepción:

```json
"dependencies": {
  "express": "latest",
  "mongoose": "latest",
  "cors": "latest"
}
```

**Nunca** poner versiones fijas como `"^4.19.2"` — pueden tener vulnerabilidades conocidas
al momento de generar y se desactualizan sin que la skill lo sepa.

### El script generador corre `npm install` + `npm audit` automáticamente

El script `generate-backend.js` tiene tres fases obligatorias:

1. **Crear archivos** — estructura del proyecto
2. **`npm install --registry https://registry.npmjs.org`** — instala la última versión estable de cada paquete forzando el registry público oficial, para evitar conflictos con proxies o mirrors corporativos (Nexus, Artifactory, etc.)
3. **`npm audit --audit-level=moderate`** — detecta vulnerabilidades conocidas

### Proxies corporativos — manejo de errores

Si `npm install` falla por red, el script imprime un mensaje específico con tres causas posibles:
sin internet, firewall bloqueando registry.npmjs.org, o proxy corporativo requiriendo autenticación.
En ese caso el usuario debe consultar con IT el registry npm habilitado en su empresa.

El usuario no necesita correr `npm install` manualmente — ya lo hace el script.

Si `npm audit` detecta vulnerabilidades, el script **avisa pero no aborta**.
Muestra el reporte y sugiere `npm audit fix`. La decisión final es del usuario.

---

## Módulos incluidos por defecto

Basado en el perfil del equipo, el scaffold SIEMPRE incluye:

| Módulo | Paquete | Uso |
|--------|---------|-----|
| Framework | `express` | Servidor HTTP |
| Variables de entorno | `dotenv` | Leer `.env` |
| CORS | `cors` | Permitir requests cross-origin |
| Logging | `morgan` | Log de requests en consola |
| Auth JWT | `jsonwebtoken` | Generar y verificar tokens |
| Hash passwords | `bcryptjs` | Nunca guardar passwords en texto plano |
| Dev server | `nodemon` (devDep) | Recarga automática en desarrollo |

**Base de datos** — incluir según lo que pida el usuario:
- MongoDB: `mongoose`
- PostgreSQL: `pg` + `pg-pool`
- Ninguna: modelo de ejemplo en memoria (array) con comentario explicativo

---

## Referencias

- `references/structure.md` — Arquitectura detallada y decisiones de diseño
- `references/templates.md` — Código completo de cada archivo del scaffold

Lee ambos archivos antes de generar el script si necesitás el código exacto de algún template.
