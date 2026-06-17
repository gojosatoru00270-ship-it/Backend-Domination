# 🟢 Node.js Backend Foundation — Complete Documentation

> A beginner-friendly, from-scratch guide to building a backend with **Node.js** and **Express** — written in documentation style.

---

## 📌 Table of Contents

1. [Overview & Core Flow](#overview)
2. [Part 1 — Node.js Basics](#part-1-nodejs)
   - What is Node.js?
   - How Node.js Works
   - Module System (require / exports)
   - Built-in Modules
3. [Part 2 — npm & package.json](#part-2-npm)
   - What is npm?
   - Initializing a Project
   - Installing Packages
   - package.json Explained
   - devDependencies vs dependencies
   - Scripts
4. [Part 3 — Express.js](#part-3-express)
   - What is Express?
   - Installation & Setup
   - Creating a Server
   - Routing
   - Route Parameters & Query Strings
   - Request Object (req)
   - Response Object (res)
5. [Part 4 — Middleware](#part-4-middleware)
   - What is Middleware?
   - Built-in Middleware
   - Third-party Middleware
   - Custom Middleware
   - Middleware Execution Order
6. [Part 5 — REST API Design](#part-5-rest)
   - What is a REST API?
   - HTTP Methods
   - Status Codes
   - Building CRUD Routes
7. [Part 6 — Environment Variables](#part-6-env)
   - What is .env?
   - dotenv Setup
   - Best Practices
8. [Part 7 — Project Structure](#part-7-structure)
   - Folder Structure
   - Full Working Example
   - Common Errors & Fixes
9. [Quick Reference Cheatsheet](#cheatsheet)

---

## 🗺️ Overview & Core Flow {#overview}

**Node.js** lets you run JavaScript on the server — outside the browser. Before Node.js, JavaScript only ran in browsers. Now you can use it to build APIs, servers, and command-line tools.

**Express.js** is a minimal web framework built on top of Node.js. It makes it easy to handle HTTP requests, define routes, and send responses.

### How a Backend Request Works

```
Client (Browser / Postman / Mobile App)
              ↓  sends HTTP Request
         Express Server
              ↓
         Middleware runs
         (auth check, body parsing, logging...)
              ↓
         Matching Route found
              ↓
         Controller / Handler runs
         (reads DB, processes logic...)
              ↓
         Response sent back to Client
```

Think of **Node.js** as the engine, **Express** as the steering wheel, and **Middleware** as the checkpoints the request passes through before reaching its destination.

---

## 🟩 Part 1 — Node.js Basics {#part-1-nodejs}

### What is Node.js?

Node.js is a **JavaScript runtime** built on Chrome's V8 engine. It allows you to execute JavaScript code on a server.

Key characteristics:

| Feature | Meaning |
|---------|---------|
| **Non-blocking** | Node doesn't wait for one task to finish before starting another |
| **Event-driven** | Everything revolves around events and callbacks |
| **Single-threaded** | One thread handles all requests using an event loop |
| **Fast** | Built on the V8 engine — same engine that powers Chrome |

> 💡 **Real World Analogy:** Node.js is like a waiter who takes multiple orders and passes them to the kitchen, instead of standing and waiting for one order to be prepared before taking the next one.

---

### How Node.js Works — The Event Loop

```
   Incoming Request
         ↓
    Event Loop
   ↙          ↘
Sync tasks   Async tasks
(run now)    (DB query, file read, API call...)
                  ↓
           Callback Queue
                  ↓
           Event Loop picks it up
           when call stack is empty
                  ↓
           Send Response
```

> ⚠️ **Important:** Never run heavy CPU tasks (like image processing in a loop) synchronously in Node.js — it will block the entire server for all users.

---

### Module System

Node.js uses the **CommonJS** module system. Every file is its own module.

#### Exporting from a file

```js
// math.js

function add(a, b) {
  return a + b;
}

function subtract(a, b) {
  return a - b;
}

module.exports = { add, subtract }; // export both functions
```

#### Importing in another file

```js
// app.js

const math = require('./math'); // ./ means same folder

console.log(math.add(5, 3));      // 8
console.log(math.subtract(5, 3)); // 2
```

**Key Terms:**

| Term | Meaning |
|------|---------|
| `module.exports` | What this file exposes to other files |
| `require('./file')` | Import from a local file (use `./` for relative path) |
| `require('package')` | Import an installed npm package (no `./`) |
| `exports` | Shorthand for `module.exports` (avoid mixing both) |

---

### Built-in Modules

Node.js comes with built-in modules — no installation needed.

```js
const fs   = require('fs');    // File System — read/write files
const path = require('path');  // File paths — join, resolve, extname
const os   = require('os');    // OS info — platform, memory
const http = require('http');  // Create raw HTTP servers
```

#### `path` — Most commonly used

```js
const path = require('path');

path.join('/users', 'john', 'photo.jpg')
// → /users/john/photo.jpg

path.extname('photo.jpg')
// → .jpg

path.basename('/users/john/photo.jpg')
// → photo.jpg

path.dirname('/users/john/photo.jpg')
// → /users/john
```

#### `fs` — Reading files

```js
const fs = require('fs');

// Synchronous (blocks the thread — avoid in production)
const data = fs.readFileSync('file.txt', 'utf8');

// Asynchronous (correct way)
fs.readFile('file.txt', 'utf8', (err, data) => {
  if (err) throw err;
  console.log(data);
});

// Modern async/await way
const { promises: fsPromises } = require('fs');
const data = await fsPromises.readFile('file.txt', 'utf8');
```

---

## 📦 Part 2 — npm & package.json {#part-2-npm}

### What is npm?

**npm** (Node Package Manager) is the world's largest software registry. It lets you install, share, and manage packages (libraries) for your Node.js project.

It comes automatically installed with Node.js.

```bash
node --version   # check Node version
npm --version    # check npm version
```

---

### Initializing a Project

```bash
mkdir my-project
cd my-project
npm init -y      # -y skips questions and uses defaults
```

This creates a `package.json` file in your project.

---

### Installing Packages

```bash
npm install express          # install one package
npm install express dotenv   # install multiple packages
npm install -D nodemon       # install as dev dependency
npm install                  # install all deps from package.json
npm uninstall express        # remove a package
```

| Flag | Meaning |
|------|---------|
| (no flag) | Install as production dependency |
| `-D` or `--save-dev` | Install as development-only dependency |
| `-g` | Install globally on your machine |

---

### package.json Explained

```json
{
  "name": "my-project",
  "version": "1.0.0",
  "description": "My Node.js backend",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "dotenv": "^16.0.3"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

**Key Terms:**

| Field | Meaning |
|-------|---------|
| `name` | Project name (lowercase, no spaces) |
| `version` | Current version (follows semver: major.minor.patch) |
| `main` | Entry point file |
| `scripts` | Shortcut commands you can run with `npm run <name>` |
| `dependencies` | Packages needed in production |
| `devDependencies` | Packages needed only during development |

---

### devDependencies vs dependencies

| Type | When used | Examples |
|------|-----------|---------|
| `dependencies` | Production + Development | `express`, `mongoose`, `dotenv` |
| `devDependencies` | Only during Development | `nodemon`, `jest`, `eslint` |

> 💡 **Pro Tip:** When you deploy to a server, run `npm install --production` — it skips devDependencies and keeps the server lean.

---

### Scripts

Scripts let you run commands with shortcuts.

```json
"scripts": {
  "start": "node server.js",
  "dev": "nodemon server.js",
  "test": "jest"
}
```

```bash
npm start        # runs "node server.js"
npm run dev      # runs "nodemon server.js"
npm test         # runs "jest"
```

> ⚠️ `npm start` and `npm test` are special — they don't need `run`. Everything else needs `npm run <name>`.

---

### nodemon — Auto Restart on Save

Without nodemon, you have to manually restart your server after every change.

```bash
npm install -D nodemon
```

```json
"scripts": {
  "dev": "nodemon server.js"
}
```

```bash
npm run dev
# Server auto-restarts every time you save a file
```

---

## 🚂 Part 3 — Express.js {#part-3-express}

### What is Express?

Express is a **minimal web framework** for Node.js. It wraps Node's raw `http` module and gives you a clean API to:

- Define routes (`GET /users`, `POST /login`)
- Parse request bodies
- Send responses (JSON, HTML, files)
- Use middleware

---

### Installation & Setup

```bash
npm install express
```

---

### Creating a Server

```js
const express = require('express');
const app = express();

const PORT = 3000;

app.listen(PORT, () => {
  console.log(`Server is running on http://localhost:${PORT}`);
});
```

**What's happening:**

| Line | Meaning |
|------|---------|
| `require('express')` | Import the Express package |
| `express()` | Create an Express application instance |
| `app.listen(PORT, cb)` | Start the server on the given port, run callback once ready |

---

### Routing

A **route** tells the server: "When a request comes in at THIS path with THIS method, run THIS function."

```js
// Syntax:
// app.METHOD(PATH, HANDLER)

app.get('/', (req, res) => {
  res.send('Hello World!');
});

app.get('/users', (req, res) => {
  res.json({ users: ['Alice', 'Bob'] });
});

app.post('/users', (req, res) => {
  res.status(201).json({ message: 'User created' });
});

app.put('/users/:id', (req, res) => {
  res.json({ message: `User ${req.params.id} updated` });
});

app.delete('/users/:id', (req, res) => {
  res.json({ message: `User ${req.params.id} deleted` });
});
```

---

### Route Parameters & Query Strings

#### Route Parameters — part of the URL path

```js
// URL: /users/42
app.get('/users/:id', (req, res) => {
  console.log(req.params.id); // "42"
  res.json({ userId: req.params.id });
});

// URL: /posts/2024/nodejs
app.get('/posts/:year/:slug', (req, res) => {
  console.log(req.params.year); // "2024"
  console.log(req.params.slug); // "nodejs"
});
```

#### Query Strings — key=value after `?`

```js
// URL: /search?keyword=node&page=2
app.get('/search', (req, res) => {
  console.log(req.query.keyword); // "node"
  console.log(req.query.page);    // "2"
  res.json(req.query);
});
```

**Key Terms:**

| Term | URL Example | Access |
|------|-------------|--------|
| Route param | `/users/42` | `req.params.id` |
| Query string | `/search?q=node` | `req.query.q` |
| Request body | POST body JSON | `req.body` |

---

### Request Object (`req`)

The `req` object represents the incoming HTTP request.

```js
app.post('/example', (req, res) => {
  req.body         // parsed request body (JSON or form data)
  req.params       // URL route parameters → { id: '42' }
  req.query        // query string params → { page: '2' }
  req.headers      // request headers → { authorization: 'Bearer ...' }
  req.method       // HTTP method → 'GET', 'POST', etc.
  req.url          // full URL path → '/users?page=1'
  req.ip           // client IP address
});
```

---

### Response Object (`res`)

The `res` object is used to send a response back to the client.

```js
res.send('Hello')                          // send plain text
res.json({ message: 'ok' })               // send JSON (sets Content-Type automatically)
res.status(404).json({ error: 'Not found' }) // send with status code
res.redirect('/new-url')                   // redirect to another URL
res.sendFile('/path/to/file.html')         // send a file
res.end()                                  // end response with no data
```

**Common status codes used with `res.status()`:**

| Code | Meaning |
|------|---------|
| `200` | OK — success |
| `201` | Created — resource created |
| `400` | Bad Request — invalid input |
| `401` | Unauthorized — not logged in |
| `403` | Forbidden — logged in but no permission |
| `404` | Not Found |
| `500` | Internal Server Error |

---

## 🔁 Part 4 — Middleware {#part-4-middleware}

### What is Middleware?

Middleware is a function that runs **between** the request arriving and the response being sent. It has access to `req`, `res`, and a `next` function.

```
Request → [Middleware 1] → [Middleware 2] → [Route Handler] → Response
```

```js
// Middleware signature
function myMiddleware(req, res, next) {
  // do something with req or res
  next(); // pass control to the next middleware/route
}
```

> ⚠️ **Important:** If you forget to call `next()`, the request will hang forever — the client never gets a response.

---

### Built-in Middleware

Express ships with built-in middleware for the most common tasks:

#### `express.json()` — Parse JSON request bodies

```js
app.use(express.json());

// Now req.body works for JSON requests
app.post('/users', (req, res) => {
  console.log(req.body); // { name: "Alice", age: 25 }
});
```

#### `express.urlencoded()` — Parse HTML form data

```js
app.use(express.urlencoded({ extended: true }));

// Now req.body works for form submissions
```

#### `express.static()` — Serve static files

```js
app.use(express.static('public'));
// Files in /public folder are now accessible:
// /public/index.html → http://localhost:3000/index.html
// /public/style.css  → http://localhost:3000/style.css
```

---

### Third-party Middleware

```bash
npm install cors morgan helmet
```

#### `cors` — Allow cross-origin requests

```js
const cors = require('cors');

app.use(cors());
// Now frontend on localhost:5173 can call your API on localhost:3000

// Restrict to specific origin
app.use(cors({ origin: 'https://mywebsite.com' }));
```

#### `morgan` — HTTP request logger

```js
const morgan = require('morgan');

app.use(morgan('dev'));
// Logs: GET /users 200 5.123 ms - 83
```

#### `helmet` — Set security headers

```js
const helmet = require('helmet');

app.use(helmet());
// Adds headers like X-Frame-Options, X-XSS-Protection etc.
```

---

### Custom Middleware

You can write your own middleware for any purpose:

#### Request Logger

```js
function requestLogger(req, res, next) {
  console.log(`[${new Date().toISOString()}] ${req.method} ${req.url}`);
  next(); // must call next() to continue
}

app.use(requestLogger); // applies to ALL routes
```

#### Auth Check (Route-level)

```js
function isAuthenticated(req, res, next) {
  const token = req.headers.authorization;

  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }

  // verify token logic here...
  next();
}

// Apply only to specific routes
app.get('/dashboard', isAuthenticated, (req, res) => {
  res.json({ message: 'Welcome to dashboard' });
});
```

---

### Middleware Execution Order

Order matters. Middleware runs **top to bottom**.

```js
app.use(express.json());      // 1st — parse body
app.use(morgan('dev'));        // 2nd — log request
app.use(cors());              // 3rd — handle CORS

app.get('/users', handler);   // 4th — route matched

app.use(errorHandler);        // LAST — error handling middleware
```

> 💡 **Pro Tip:** Always put `express.json()` before your routes, or `req.body` will be `undefined`.

**Error-handling middleware** has 4 parameters — Express identifies it by the `err` argument:

```js
// Must have exactly 4 params: err, req, res, next
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: err.message });
});
```

---

## 🌐 Part 5 — REST API Design {#part-5-rest}

### What is a REST API?

**REST** (Representational State Transfer) is a set of rules for designing APIs. A REST API uses HTTP methods and URLs to let clients perform operations on resources.

**Resources** = the "things" in your app (users, posts, products, orders)

---

### HTTP Methods

| Method | Purpose | Example |
|--------|---------|---------|
| `GET` | Fetch data | Get all users / Get one user |
| `POST` | Create new data | Create a user |
| `PUT` | Replace entire resource | Replace user info |
| `PATCH` | Update part of a resource | Update only user email |
| `DELETE` | Remove data | Delete a user |

---

### Building CRUD Routes

**CRUD** = Create, Read, Update, Delete — the 4 basic operations.

For a `users` resource:

```js
const express = require('express');
const router = express.Router();

// READ ALL   — GET /users
router.get('/', (req, res) => {
  res.json({ users: [] });
});

// READ ONE   — GET /users/:id
router.get('/:id', (req, res) => {
  res.json({ userId: req.params.id });
});

// CREATE     — POST /users
router.post('/', (req, res) => {
  const { name, email } = req.body;

  if (!name || !email) {
    return res.status(400).json({ error: 'Name and email are required' });
  }

  res.status(201).json({ message: 'User created', user: { name, email } });
});

// UPDATE     — PUT /users/:id
router.put('/:id', (req, res) => {
  res.json({ message: `User ${req.params.id} updated` });
});

// DELETE     — DELETE /users/:id
router.delete('/:id', (req, res) => {
  res.json({ message: `User ${req.params.id} deleted` });
});

module.exports = router;
```

Mount the router in `server.js`:

```js
app.use('/api/users', require('./routes/users'));

// This maps:
// GET    /api/users       → router.get('/')
// GET    /api/users/42    → router.get('/:id')
// POST   /api/users       → router.post('/')
// PUT    /api/users/42    → router.put('/:id')
// DELETE /api/users/42    → router.delete('/:id')
```

---

### Standard API Response Format

Always return consistent JSON so the frontend knows what to expect:

```js
// Success
res.status(200).json({
  success: true,
  data: { user }
});

// Error
res.status(400).json({
  success: false,
  error: 'Validation failed',
  message: 'Email is required'
});
```

---

## 🔐 Part 6 — Environment Variables {#part-6-env}

### What is `.env`?

A `.env` file stores **secret configuration** that should not be hardcoded in your code — like database passwords, API keys, and port numbers.

```bash
npm install dotenv
```

---

### Setup

```js
// server.js — must be the VERY FIRST line
require('dotenv').config();

// Now you can access variables from .env
const PORT = process.env.PORT || 3000;
const DB_URL = process.env.DB_URL;
```

```
# .env file
PORT=5000
DB_URL=mongodb://localhost:27017/mydb
JWT_SECRET=mysupersecretkey123
NODE_ENV=development
```

**Key Terms:**

| Term | Meaning |
|------|---------|
| `process.env` | Global object in Node.js that holds environment variables |
| `.env` | File where you define variables (never commit to Git) |
| `|| 3000` | Fallback value if the variable is not set |
| `NODE_ENV` | Convention to define environment: `development`, `production`, `test` |

---

### Best Practices

```
# .env — your actual secrets (never commit this)
PORT=3000
DB_URL=mongodb+srv://user:pass@cluster.mongodb.net/mydb

# .env.example — template with no real values (commit this for teammates)
PORT=
DB_URL=
JWT_SECRET=
```

**`.gitignore`** — always add `.env`:

```
node_modules/
.env
```

> ⚠️ **Critical:** Never push `.env` to GitHub. Your API keys can be stolen and misused within minutes by bots that scan public repos.

---

## 🗂️ Part 7 — Project Structure {#part-7-structure}

### Recommended Folder Structure

```
my-project/
├── server.js            ← entry point, starts the server
├── app.js               ← Express app setup (middleware, routes)
├── .env                 ← environment variables (never commit)
├── .env.example         ← template for teammates
├── .gitignore
├── package.json
│
├── config/
│   └── db.js            ← database connection
│
├── routes/
│   ├── users.js         ← /api/users routes
│   └── posts.js         ← /api/posts routes
│
├── controllers/
│   ├── userController.js  ← logic for user routes
│   └── postController.js
│
├── middleware/
│   ├── auth.js          ← authentication middleware
│   └── errorHandler.js  ← global error handler
│
└── models/              ← database models (Mongoose schemas etc.)
    └── User.js
```

---

### Full Working Example

#### `server.js`

```js
require('dotenv').config();
const app = require('./app');

const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
  console.log(`✅ Server running on http://localhost:${PORT}`);
});
```

---

#### `app.js`

```js
const express = require('express');
const cors = require('cors');
const morgan = require('morgan');

const app = express();

// ─── Middleware ─────────────────────────────────────────────
app.use(cors());
app.use(morgan('dev'));
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// ─── Routes ─────────────────────────────────────────────────
app.use('/api/users', require('./routes/users'));

// ─── 404 Handler ────────────────────────────────────────────
app.use((req, res) => {
  res.status(404).json({ error: 'Route not found' });
});

// ─── Global Error Handler ───────────────────────────────────
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(err.status || 500).json({
    success: false,
    error: err.message || 'Internal Server Error'
  });
});

module.exports = app;
```

---

#### `routes/users.js`

```js
const express = require('express');
const router = express.Router();
const { getAllUsers, createUser, deleteUser } = require('../controllers/userController');
const isAuthenticated = require('../middleware/auth');

router.get('/', getAllUsers);
router.post('/', createUser);
router.delete('/:id', isAuthenticated, deleteUser);

module.exports = router;
```

---

#### `controllers/userController.js`

```js
// Keeping route logic separate from route definitions
const getAllUsers = (req, res) => {
  res.json({ success: true, data: [] });
};

const createUser = (req, res) => {
  const { name, email } = req.body;

  if (!name || !email) {
    return res.status(400).json({ success: false, error: 'Name and email required' });
  }

  res.status(201).json({ success: true, data: { name, email } });
};

const deleteUser = (req, res) => {
  res.json({ success: true, message: `User ${req.params.id} deleted` });
};

module.exports = { getAllUsers, createUser, deleteUser };
```

---

#### `middleware/auth.js`

```js
const isAuthenticated = (req, res, next) => {
  const authHeader = req.headers.authorization;

  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Unauthorized — no token' });
  }

  const token = authHeader.split(' ')[1]; // extract token after "Bearer "

  // In real app: verify JWT token here
  if (!token) {
    return res.status(401).json({ error: 'Invalid token' });
  }

  next();
};

module.exports = isAuthenticated;
```

---

#### `middleware/errorHandler.js`

```js
const errorHandler = (err, req, res, next) => {
  const statusCode = err.statusCode || 500;

  res.status(statusCode).json({
    success: false,
    error: err.message || 'Something went wrong'
  });
};

module.exports = errorHandler;
```

---

#### Install all dependencies

```bash
npm install express dotenv cors morgan helmet
npm install -D nodemon
```

---

#### `.gitignore`

```
node_modules/
.env
```

---

### Testing with Postman / Thunder Client

```
GET    http://localhost:3000/api/users
POST   http://localhost:3000/api/users    Body → JSON → { "name": "Alice", "email": "alice@example.com" }
DELETE http://localhost:3000/api/users/1  Headers → Authorization: Bearer mytoken
```

---

### Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Cannot GET /` | No route defined for that path | Add `app.get('/', ...)` or check URL spelling |
| `req.body is undefined` | `express.json()` not added | Add `app.use(express.json())` before routes |
| `listen EADDRINUSE :::3000` | Port 3000 already in use | Change PORT in `.env` or kill the process using the port |
| `Cannot find module './routes/users'` | Wrong file path | Check the path and filename (case-sensitive) |
| `next is not a function` | Forgot to pass `next` in middleware params | Middleware must have `(req, res, next)` |
| `process.env.PORT is undefined` | `.env` file not loaded | Add `require('dotenv').config()` as the first line in `server.js` |
| `SyntaxError: Unexpected token` | Invalid JSON in request body | Check Postman body — must be valid JSON |
| `router.get is not a function` | Forgot `express.Router()` | Use `const router = express.Router()` |

---

## ✅ Quick Reference Cheatsheet {#cheatsheet}

```js
// ─── Setup ──────────────────────────────────────────────────
const express = require('express');
const app = express();
app.listen(3000, () => console.log('Running'));

// ─── Middleware ──────────────────────────────────────────────
app.use(express.json());               // parse JSON body
app.use(express.urlencoded({ extended: true })); // parse form body
app.use(cors());                       // allow cross-origin
app.use(morgan('dev'));                 // log requests
app.use(express.static('public'));     // serve static files

// ─── Routing ────────────────────────────────────────────────
app.get('/path', handler)             // read
app.post('/path', handler)            // create
app.put('/path/:id', handler)         // replace
app.patch('/path/:id', handler)       // partial update
app.delete('/path/:id', handler)      // delete

// ─── Request ────────────────────────────────────────────────
req.body          // { name: 'Alice' }       ← POST body
req.params        // { id: '42' }            ← /users/:id
req.query         // { page: '2' }           ← /users?page=2
req.headers       // { authorization: '...' }

// ─── Response ───────────────────────────────────────────────
res.json({ key: 'value' })            // send JSON
res.status(201).json({ ... })         // with status code
res.send('text')                      // send plain text
res.redirect('/new-route')            // redirect

// ─── Router ─────────────────────────────────────────────────
const router = express.Router();
router.get('/', handler);
module.exports = router;
app.use('/api/users', router);        // mount router

// ─── Environment ────────────────────────────────────────────
require('dotenv').config();           // load .env
process.env.PORT                      // access variable
process.env.NODE_ENV                  // 'development' | 'production'
```

---

> 📝 **Summary:** **Node.js** is the runtime, **Express** is the framework, **Middleware** is the pipeline your request flows through, and **dotenv** keeps your secrets safe. Structure your project with separate `routes/`, `controllers/`, and `middleware/` folders — it keeps your code clean, readable, and scalable.