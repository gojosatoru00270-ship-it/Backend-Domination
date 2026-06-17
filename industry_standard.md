# 📁 Backend Folder Structure — Reference Guide

> Apne future projects ke liye ek baar padh ke samajh lo, phir jab bhi naya backend banao — isi structure ko copy-paste karke follow kar sakte ho. Node.js + Express + MongoDB stack ke liye banaya gaya hai.

---

## 📌 Table of Contents

1. [Overview — Kyun Alag Folders Banate Hain](#overview)
2. [Har Folder Ka Matlab — One by One](#folders)
3. [Do Variants — Simple vs Scalable](#variants)
4. [Request Lifecycle Diagram](#lifecycle)
5. [Naming Conventions](#naming)
6. [Full Reference Structure](#structure)
7. [Quick Cheatsheet](#cheatsheet)

---

## 🗺️ Overview — Kyun Alag Folders Banate Hain {#overview}

Agar sab kuch ek `server.js` mein likh do, toh shuru mein chalta hai, lekin project badhne pe problem aati hai:

- DB logic, validation, response formatting — sab ek dusre se chipak jaata hai
- Ek cheez change karne pe (jaise Cloudinary se S3 shift karna) pura file khojna padta hai
- Reuse karna mushkil ho jaata hai — same logic do jagah copy-paste karna padta hai

**Solution:** Har folder ek hi responsibility nibhaye, aur sirf apne "neeche" wale layer ko call kare.

```
Request
  ↓
routes/        →  konsa URL kis controller ko jaayega
  ↓
middlewares/   →  route tak pahunchne se pehle ek check (guard)
  ↓
controllers/   →  request lo, kaam karwao, response do
  ↓
services/      →  asli business logic + DB query (optional layer)
  ↓
models/        →  DB mein data ka shape
```

Rule: **upar wala neeche wale ko call kar sakta hai, neeche wala upar wale ko nahi jaanta.** Isi se code reusable aur testable banta hai.

---

## 📂 Har Folder Ka Matlab — One by One {#folders}

### `config/`
**Kya hai:** Connections aur third-party services ka setup.
**Use:** Database connect karna (`db.js`), Cloudinary jaisi service configure karna (`cloudinary.config.js`).
**Rule:** Yahan sirf *setup* hota hai — actual kaam (query, upload) yahan nahi hota.

```js
// config/db.js
export const connectDB = async () => {
  await mongoose.connect(process.env.MONGO_URI);
};
```

---

### `models/`
**Kya hai:** Mongoose Schema — DB mein document ka shape.
**Use:** Fields, validation rules (`required`, `unique`), aur relationships (`ref`) define karna.
**Yaad rakho:** Model ko HTTP ka kuch pata nahi hota — sirf data ka structure jaanta hai.

```js
// models/post.model.js
const postSchema = new mongoose.Schema({
  title: { type: String, required: true },
  author: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
});
```

---

### `services/`
**Kya hai:** Asli business logic — DB queries, third-party API calls (jaise Cloudinary delete).
**Use:** Controller se kaam alag karna taaki logic reusable rahe (CLI script, doosra route — kahin se bhi call ho sake).
**Yaad rakho:** Service ko `req`/`res` ka kuch pata nahi hota — sirf data leke kaam karta hai, data return karta hai.

```js
// services/post.service.js
export const createPost = async (data) => {
  const post = new Post(data);
  return await post.save();
};
```

> 💡 Chhote projects mein (jaise simple Multer+Cloudinary upload feature) ye layer skip bhi kiya ja sakta hai — [Variant comparison](#variants) neeche dekho.

---

### `controllers/`
**Kya hai:** Traffic cop — request handle karta hai.
**Use:** `req` se data nikalna (`req.body`, `req.file`, `req.user`), service ko call karna, response format karke bhejna.
**Yaad rakho:** Controller khud DB ko directly touch nahi karta — wo sirf HTTP layer handle karta hai (status codes, error format).

```js
// controllers/post.controller.js
export const create = async (req, res) => {
  try {
    const post = await postService.createPost(req.body);
    res.status(201).json({ success: true, data: post });
  } catch (err) {
    res.status(400).json({ success: false, message: err.message });
  }
};
```

---

### `middlewares/`
**Kya hai:** Guard functions — request route tak pahunchne se pehle koi check.
**Use:** Auth check (`protect`), file upload parsing (`multer`), validation, logging.
**Yaad rakho:** `next()` call karna zaroori hai, warna request wahin atak jaayegi.

```js
// middlewares/auth.middleware.js
export const protect = (req, res, next) => {
  const decoded = jwt.verify(token, process.env.JWT_SECRET);
  req.user = decoded;   // aage wale functions ke liye attach kar diya
  next();
};
```

---

### `routes/`
**Kya hai:** URL → function mapping.
**Use:** Kaunsa HTTP method + path, kin middlewares se hoke, kis controller tak jaayega — sirf ye define karta hai.
**Yaad rakho:** Yahan koi logic nahi likhte, sirf "wiring" hoti hai.

```js
// routes/post.routes.js
router.post('/', protect, upload.single('image'), create);
//              ↑ guard      ↑ file parse          ↑ actual handler
```

---

### `utils/`
**Kya hai:** Chhoti, reusable helper functions jo kisi ek layer se bandhi nahi hoti.
**Use:** Response formatter (`apiResponse.js`), third-party SDK setup jo multiple jagah use hota hai (`cloudinary.js`).

```js
// utils/apiResponse.js
export const successResponse = (message, data) => ({ success: true, message, data });
```

---

### `.env`
**Kya hai:** Secret/config values — kabhi Git mein commit nahi hota.
**Use:** DB URI, JWT secret, API keys. `.gitignore` mein zaroor add karo.

---

### `server.js` (ya `app.js` + `server.js` alag)
**Kya hai:** App ka entry point.
**Use:** Express app banana, middlewares register karna (`cors`, `express.json`), routes mount karna, aur DB connect hone ke **baad** server start karna.

```js
connectDB().then(() => {
  app.listen(PORT, () => console.log(`Server running on ${PORT}`));
});
```

> ⚠️ Logic: server tab tak requests accept na kare jab tak DB se connection na ban jaaye — warna pehli request pe hi crash ho sakta hai.

---

## ⚖️ Do Variants — Simple vs Scalable {#variants}

Har feature ke liye pura `service` layer zaroori nahi. Project ke size ke hisaab se decide karo:

| | **Variant A — Simple** (controller-only) | **Variant B — Scalable** (with services) |
|---|---|---|
| Logic kahan likhte ho | Seedha controller mein | Service mein, controller sirf call karta hai |
| Kab use karo | Chhota feature, ek-baar ka script, fast prototype (jaise Multer+Cloudinary direct upload) | Full app jahan multiple models, multiple routes same logic use karte hain |
| Reusability | Kam — logic controller se chipka hota hai | High — service kahin bhi import ho sakti hai |
| Example | `uploadPost` controller jisme seedha `streamifier.pipe(cloudinary)` chal raha ho | `auth.service.js` jisme `registerUser`/`loginUser` defined ho, multiple controllers use karein |

> 💡 Rule of thumb: **agar future mein wahi logic 2+ jagah se call hoga, ya complex DB operations hain, service layer banao.** Agar ek simple, self-contained kaam hai (jaise sirf image upload), controller mein hi rakh sakte ho.

---

## 🔄 Request Lifecycle Diagram {#lifecycle}

Example: `POST /api/posts` — image ke saath naya post banana (auth protected)

```
Client (FormData + Bearer token)
        │
        ▼
server.js / app.js  ──→  app.use('/api/posts', postRoutes)
        │
        ▼
routes/post.routes.js
        │  router.post('/', protect, upload.single('image'), create)
        ▼
middlewares/auth.middleware.js  ──→  token verify, req.user set, next()
        │
        ▼
utils/cloudinary.js (multer middleware)  ──→  file Cloudinary pe upload, req.file set
        │
        ▼
controllers/post.controller.js  ──→  req.body + req.file + req.user nikala
        │
        ▼
services/post.service.js  ──→  DB mein document banaya
        │
        ▼
models/post.model.js  ──→  schema validate, MongoDB mein save
        │
        ▼
Response wapas: Model → Service → Controller → res.json()
```

---

## 🏷️ Naming Conventions {#naming}

| Pattern | Example | Kyun |
|---|---|---|
| `*.model.js` | `user.model.js` | Schema files clearly identify ho jaayein |
| `*.service.js` | `auth.service.js` | Business logic files |
| `*.controller.js` | `post.controller.js` | Request handlers |
| `*.middleware.js` | `auth.middleware.js` | Guard functions |
| `*.routes.js` | `post.routes.js` | Route definitions |
| `*.config.js` | `cloudinary.config.js` | Setup/connection files |
| camelCase functions | `createPost`, `getAllPosts` | Functions ke naam |
| PascalCase models | `User`, `Post` | Mongoose model exports |

---

## 🗂️ Full Reference Structure {#structure}

```
backend/
│
├── config/
│   ├── db.js                    ← MongoDB connection
│   └── cloudinary.config.js     ← Cloudinary setup
│
├── models/
│   ├── user.model.js
│   └── post.model.js
│
├── services/                    ← optional — bade features ke liye
│   ├── auth.service.js
│   └── post.service.js
│
├── controllers/
│   ├── auth.controller.js
│   └── post.controller.js
│
├── middlewares/
│   ├── auth.middleware.js       ← JWT guard
│   └── multer.js                ← file parsing guard
│
├── routes/
│   ├── auth.routes.js
│   └── post.routes.js
│
├── utils/
│   ├── apiResponse.js           ← response formatter
│   └── cloudinary.js            ← upload_stream helper (agar utils mein rakhna ho)
│
├── .env
├── .gitignore                   ← .env aur node_modules zaroor add karo
├── package.json
└── server.js
```

---

## ✅ Quick Cheatsheet {#cheatsheet}

| Folder | One-line yaad |
|---|---|
| `config/` | Connection/setup — DB, Cloudinary |
| `models/` | Data ka shape — schema + validation |
| `services/` | Asli logic — DB query, reusable, HTTP se independent |
| `controllers/` | Traffic cop — req lo, service call karo, res do |
| `middlewares/` | Guard — route tak pahunchne se pehle check |
| `routes/` | URL → controller mapping, koi logic nahi |
| `utils/` | Chhoti reusable helpers |
| `.env` | Secrets — Git mein kabhi nahi |
| `server.js` | Entry point — DB connect hone ke baad server start |

**Golden rule:** Upar wala layer neeche wale ko call kar sakta hai, neeche wala upar wale ko nahi jaanta. Isi se separation of concern maintain hota hai.

---

> 📝 Ye guide future projects mein copy-paste karne ke liye banayi gayi hai — jab bhi naya backend shuru karo, yahi structure follow karo aur is file ko apne repo ke root mein `docs/` folder mein rakh do.
