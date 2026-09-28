# 🛠️ node-backend-scaffold

Skill para Claude que genera un esqueleto de backend **Node.js 20 + Express** completo, didáctico y listo para correr en un solo comando.

Orientada a equipos de Data Science / Data Engineering que necesitan exponer APIs rápidamente sin experiencia previa en backend.

---

## ¿Qué genera?

Al activarse, Claude pregunta el nombre del proyecto, las entidades/colecciones principales y la base de datos. Con esa info genera un único archivo `generate-backend.js` que al correrlo produce:

```
mi-proyecto/
├── src/
│   ├── index.js              ← Entry point
│   ├── app.js                ← Configuración de Express
│   ├── routes/               ← Mapeo de URLs a controllers
│   ├── controllers/          ← Lógica de cada endpoint
│   ├── models/               ← Schemas de base de datos
│   ├── middlewares/          ← Auth, manejo de errores
│   └── config/
│       └── db.js             ← Conexión a la DB
├── .env.example
├── .gitignore
└── package.json
```

Cada archivo tiene comentarios que explican su rol, su responsabilidad y el flujo completo desde el request hasta la respuesta.

---

## Stack soportado

| Base de datos | Paquete |
|---------------|---------|
| MongoDB | Mongoose |
| PostgreSQL | pg |
| Ninguna (en memoria) | — |

Dependencias core: `express`, `dotenv`, `cors`, `morgan`, `nodemon`.

---

## Cómo instalar la skill

1. Descargá `node-backend-skill.skill` desde este repo
2. En Claude → **Settings → Skills → Add skill**
3. Subí el archivo `.skill`

> Si tu equipo usa una organización de Claude (Team / Enterprise), el admin puede subirla una sola vez desde Settings y queda disponible para todos.

---

## Cómo usarla

Una vez instalada, simplemente describile a Claude lo que necesitás en lenguaje natural:

> *"Tengo una base de datos en MongoDB con clientes y transacciones, necesito un backend para leer y editar esos datos"*

Claude va a hacer un par de preguntas (nombre del proyecto, colecciones) y entregará el script generador.

**Para correr el script:**

```bash
node generate-backend.js   # genera archivos + npm install + npm audit
cd mi-proyecto
cp .env.example .env       # completar credenciales
npm run dev                # levantar el servidor
```

---

## Notas importantes

- Las dependencias siempre se instalan en su **última versión estable** (no versiones hardcodeadas)
- El script corre `npm audit` automáticamente y reporta vulnerabilidades si las hay
- El `npm install` usa `--registry https://registry.npmjs.org` para evitar conflictos con proxies o mirrors corporativos (Nexus, Artifactory, etc.)

---

## V2 — Próximos pasos

- [ ] **Schema desde campos reales** — que Claude pregunte los campos de cada colección y genere los models con la estructura exacta, sin que el usuario tenga que editar nada a mano
- [ ] **Archivo de requests listos** — generar un `requests.http` (VS Code REST Client) o colección de Postman con ejemplos de cada endpoint, listo para importar y probar
- [ ] **Validación de `.env` al arrancar** — si falta alguna variable de entorno, el servidor muestra exactamente cuál falta con un mensaje claro, en lugar de un error críptico de Mongoose o pg

---

## Archivos de la skill

```
node-backend-scaffold/
├── README.md          ← este archivo
├── SKILL.md           ← instrucciones para Claude
└── references/
    ├── structure.md   ← arquitectura y decisiones de diseño
    └── templates.md   ← código completo de cada archivo generado
```
