# 📁 Multer + Cloudinary — Simple Documentation

> Beginner-friendly guide — Frontend form se backend Cloudinary tak, poora flow.

---

## 📌 Table of Contents

1. [Overview & Core Flow](#overview)
2. [Part 1 — Frontend (Form se File Bhejna)](#part-1-frontend)
3. [Part 2 — Multer](#part-2-multer)
4. [Part 3 — Cloudinary](#part-3-cloudinary)
5. [Part 4 — Core Concepts (Deep Dive)](#part-4-concepts)
6. [Part 5 — Full Working Example](#part-5-integration)
7. [Common Errors & Fixes](#errors)
8. [Quick Reference Cheatsheet](#cheatsheet)

---

## 🗺️ Overview & Core Flow {#overview}

**Multer** Node.js middleware hai jo `multipart/form-data` handle karta hai — yeh wahi format hai jo form file ke saath bhejta hai.

**Cloudinary** cloud media service hai — image/video store karta hai aur public URL deta hai.

```
HTML Form (enctype="multipart/form-data")
              ↓
      Browser request bhejta hai
              ↓
      Express Route receives request
              ↓
      Multer parses the file from the request
      (RAM mein Buffer ke roop mein rakhta hai)
              ↓
      Cloudinary receives the file
      → stores it on the cloud
      → returns a public URL
              ↓
      You save that URL in your database
```

Think of **Multer** as the delivery guy who picks up the file, and **Cloudinary** as the warehouse where the file is stored permanently.

---

## 🖥️ Part 1 — Frontend (Form se File Bhejna) {#part-1-frontend}

File upload karne ke liye normal form se kaam nahi chalega — kuch specific settings chahiye.

### `enctype` — Yeh kya hai aur kyun zaroori hai?

Normal HTML form text data `application/x-www-form-urlencoded` format mein bhejta hai — yeh format file bhej hi nahi sakta. File bhejne ke liye `enctype="multipart/form-data"` lagana **zaroori** hai.

```html
<form enctype="multipart/form-data">
  <!-- ❌ Bina enctype ke — file kabhi backend tak nahi pahunchegi -->
</form>
```

| `enctype` value | Use case |
|------------------|----------|
| `application/x-www-form-urlencoded` | Default — sirf text fields |
| `multipart/form-data` | Files + text dono bhejne ke liye |
| `text/plain` | Rarely used |

> ⚠️ **Important:** Agar `enctype="multipart/form-data"` nahi lagaya, toh `<input type="file">` ka data backend pe khaali ya corrupt aayega.

---

### Plain HTML Form (Page Reload Wala Tarika)

Yeh sabse simple tarika hai — bina JavaScript ke, seedha form submit.

```html
<form action="http://localhost:3000/api/posts" method="POST" enctype="multipart/form-data">

  <input type="text" name="title" placeholder="Post Title" required>
  <input type="text" name="description" placeholder="Description">

  <!-- single file -->
  <input type="file" name="image" accept="image/*">

  <!-- multiple files ke liye 'multiple' attribute lagao -->
  <!-- <input type="file" name="image" accept="image/*" multiple> -->

  <button type="submit">Upload</button>

</form>
```

**Key Terms:**

| Attribute | Meaning |
|-----------|---------|
| `method="POST"` | File upload hamesha POST mein hota hai |
| `enctype="multipart/form-data"` | Files bhejne ke liye zaroori |
| `name="image"` | Yahi naam backend pe `upload.array('image')` mein match karega |
| `accept="image/*"` | Browser file picker mein sirf images dikhayega |
| `multiple` | User ek se zyada files select kar sake |

> 💡 Plain form ka downside — submit hote hi page reload ho jaata hai. Modern apps mein JS se fetch use karte hain (neeche dekho) taki page reload na ho.

---

### Vanilla JS — `FormData` + `fetch()` (No Page Reload)

Yeh tarika better hai — form submit hone par page reload nahi hota, aur response ko JS mein handle kar sakte ho (loading spinner, success message, etc).

```html
<form id="uploadForm">
  <input type="text" name="title" placeholder="Post Title" required>
  <input type="text" name="description" placeholder="Description">
  <input type="file" name="image" id="fileInput" accept="image/*" multiple>
  <button type="submit">Upload</button>
</form>

<p id="status"></p>
```

```js
const form = document.getElementById('uploadForm');
const status = document.getElementById('status');

form.addEventListener('submit', async function (e) {
  e.preventDefault();   // page reload roko

  // Step 1 — FormData banao seedha form se
  const formData = new FormData(form);
  // ↑ yeh automatically saare text fields + files uthata hai
  //   form ke 'name' attributes use karke

  status.textContent = 'Uploading...';

  try {
    // Step 2 — fetch se POST request bhejo
    const response = await fetch('http://localhost:3000/api/posts', {
      method: 'POST',
      body: formData
      // ⚠️ Important: 'Content-Type' header MAT lagao manually
      // Browser khud sahi boundary ke saath multipart/form-data set karega
    });

    const result = await response.json();

    if (response.ok) {
      status.textContent = 'Upload successful! ✅';
      console.log(result);
    } else {
      status.textContent = 'Error: ' + result.error;
    }

  } catch (err) {
    status.textContent = 'Upload failed: ' + err.message;
  }
});
```

**Key Terms:**

| Term | Meaning |
|------|---------|
| `new FormData(form)` | Form ke saare inputs (text + files) ko automatically uthata hai |
| `formData.append('key', value)` | Manually koi field add karna ho tab use karo |
| `e.preventDefault()` | Default form submit (page reload) rokta hai |
| `fetch(url, { method, body })` | Request bhejne ka modern tarika |

> ⚠️ **Common Mistake:** `fetch` mein manually `headers: { 'Content-Type': 'multipart/form-data' }` mat lagao. Browser ko khud yeh set karne do — usse ek special `boundary` string milta hai jo Multer ko file parse karne ke liye chahiye. Manually set karoge toh upload fail hoga.

---

### `FormData` Manually Banana (Without a `<form>` Tag)

Kabhi kabhi tumhare paas HTML `<form>` nahi hota — sirf JS se file uthani hoti hai:

```js
const fileInput = document.getElementById('fileInput');

fileInput.addEventListener('change', async function () {
  const formData = new FormData();

  formData.append('title', 'My Post');
  formData.append('description', 'Hello world');

  // single file
  formData.append('image', fileInput.files[0]);

  // multiple files — same key baar baar append karo
  for (const file of fileInput.files) {
    formData.append('image', file);
  }

  const response = await fetch('http://localhost:3000/api/posts', {
    method: 'POST',
    body: formData
  });

  const result = await response.json();
  console.log(result);
});
```

---

## 📦 Part 2 — Multer {#part-2-multer}

### Installation

```bash
npm install multer
```

### What is Multer?

Multer adds a `body` object and a `file` or `files` object to the `request` object. The `body` object contains the text fields of the form, and the `file`/`files` object contains the uploaded files.

> ⚠️ **Important:** Multer only processes `multipart/form-data` requests. It will not work with JSON or URL-encoded bodies.

---

### Storage Engines

#### 1. `diskStorage` — Save file to server's disk

```js
const multer = require('multer');

const storage = multer.diskStorage({
  destination: function (req, file, cb) {
    cb(null, 'uploads/'); // 'uploads' folder must exist
  },
  filename: function (req, file, cb) {
    const uniqueName = Date.now() + '-' + file.originalname;
    cb(null, uniqueName);
  }
});

const upload = multer({ storage: storage });
```

#### 2. `memoryStorage` — Hold file in RAM (as a Buffer)

```js
const storage = multer.memoryStorage();
const upload = multer({ storage: storage });
```

> 💡 **Pro Tip:** Cloudinary ke saath hamesha `memoryStorage` use karo. File `req.file.buffer` mein milegi — no temp files on disk.

---

### File Filter

```js
const upload = multer({
  storage: storage,
  fileFilter: function (req, file, cb) {
    if (file.mimetype.startsWith('image/')) {
      cb(null, true);   // accept
    } else {
      cb(new Error('Only image files are allowed!'), false);  // reject
    }
  }
});
```

---

### Limits

```js
const upload = multer({
  storage: storage,
  limits: {
    fileSize: 5 * 1024 * 1024,  // 5 MB
    files: 3
  }
});
```

| Option | Meaning |
|--------|---------|
| `fileSize` | Max file size in bytes |
| `files` | Max number of files per request |

---

### Upload Methods

#### `upload.single(fieldname)` — One file

```js
app.post('/upload', upload.single('avatar'), (req, res) => {
  console.log(req.file);
  res.send('File uploaded!');
});
```

#### `upload.array(fieldname, maxCount)` — Multiple files, same field

```js
app.post('/upload', upload.array('photos', 5), (req, res) => {
  console.log(req.files);  // array
  res.send('Files uploaded!');
});
```

#### `upload.fields(fields)` — Multiple fields

```js
app.post('/upload',
  upload.fields([
    { name: 'avatar', maxCount: 1 },
    { name: 'gallery', maxCount: 5 }
  ]),
  (req, res) => {
    console.log(req.files['avatar']);
    console.log(req.files['gallery']);
    res.send('Done!');
  }
);
```

`req.file` / `req.files[i]` contains:

| Property | Meaning |
|----------|---------|
| `originalname` | Original filename |
| `mimetype` | MIME type e.g. `image/jpeg` |
| `size` | File size in bytes |
| `buffer` | (memoryStorage only) Raw file data |
| `path` | (diskStorage only) Path where saved |

---

### Error Handling

```js
app.post('/upload', (req, res) => {
  upload.single('file')(req, res, function (err) {

    if (err instanceof multer.MulterError) {
      if (err.code === 'LIMIT_FILE_SIZE') {
        return res.status(400).json({ error: 'File too large. Max 5MB.' });
      }
      return res.status(400).json({ error: err.message });

    } else if (err) {
      return res.status(400).json({ error: err.message });
    }

    res.json({ message: 'Upload successful', file: req.file });
  });
});
```

| MulterError code | Meaning |
|-------------------|---------|
| `LIMIT_FILE_SIZE` | File size limit exceeded |
| `LIMIT_FILE_COUNT` | Too many files |
| `LIMIT_UNEXPECTED_FILE` | Field name mismatch |

---

## ☁️ Part 3 — Cloudinary {#part-3-cloudinary}

### Account Setup

1. [cloudinary.com](https://cloudinary.com) pe free account banao
2. Dashboard se `Cloud Name`, `API Key`, `API Secret` lo

### Installation

```bash
npm install cloudinary
```

### Configuration

```js
// .env
CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret
```

```js
require('dotenv').config();
const cloudinary = require('cloudinary').v2;

cloudinary.config({
  cloud_name: process.env.CLOUDINARY_CLOUD_NAME,
  api_key: process.env.CLOUDINARY_API_KEY,
  api_secret: process.env.CLOUDINARY_API_SECRET
});
```

> 🔒 **Security:** Credentials hardcode mat karo — hamesha `.env` use karo.

---

### Upload from Buffer — `upload_stream`

Multer ka `memoryStorage` file ko `Buffer` deta hai. Cloudinary ka normal `upload()` Buffer accept nahi karta — `upload_stream` use karna padta hai.

```bash
npm install streamifier
```

```js
const streamifier = require('streamifier');

function uploadToCloudinary(buffer) {
  return new Promise((resolve, reject) => {
    const stream = cloudinary.uploader.upload_stream(
      { folder: 'my-app' },
      (error, result) => {
        if (error) reject(error);
        else resolve(result);
      }
    );
    streamifier.createReadStream(buffer).pipe(stream);
  });
}

// Usage
const result = await uploadToCloudinary(req.file.buffer);
console.log(result.secure_url);
```

**Result object:**

| Property | Meaning |
|----------|---------|
| `secure_url` | HTTPS URL — yeh DB mein save karo |
| `public_id` | Unique ID — delete ke liye zaroori |
| `format` | jpg/png/webp etc. |
| `width` / `height` | Image dimensions |
| `bytes` | File size |

---

### Delete a File

```js
const result = await cloudinary.uploader.destroy('my-app/image-public-id');
```

---

### Transformations

```js
const result = await cloudinary.uploader.upload('image.jpg', {
  transformation: [
    { width: 500, height: 500, crop: 'fill' },
    { quality: 'auto' },
    { format: 'webp' }
  ]
});
```

| Option | Meaning |
|--------|---------|
| `width` / `height` | Resize dimensions |
| `crop` | `fill`, `fit`, `thumb`, `scale`, `pad` |
| `gravity` | Focal point e.g. `'face'` |
| `quality` | `'auto'` ya 1-100 |
| `format` | Output format |

---

## 🧠 Part 4 — Core Concepts (Deep Dive) {#part-4-concepts}

### Buffer — Kya hota hai?

`memoryStorage` use karne par file disk pe save nahi hoti — RAM mein **Buffer** ke roop mein rehti hai.

```
File (image.jpg) → Multer (memoryStorage) → req.file.buffer (raw binary data)
```

```js
console.log(req.file.buffer);
// <Buffer ff d8 ff e0 00 10 4a 46 49 46 ...>
```

> ⚠️ `diskStorage` use karoge toh `buffer` nahi milega — sirf `path` milega.

---

### Buffer → Stream → Cloudinary

Cloudinary ka `upload_stream` seedha Buffer accept nahi karta — pehle Buffer ko **Readable Stream** mein convert karna padta hai.

```
Buffer (RAM mein raw data)
      ↓
streamifier.createReadStream() → Readable Stream
      ↓
.pipe() → Cloudinary upload_stream (Writable Stream)
      ↓
Cloudinary servers
```

`streamifier` Buffer ko cleanly Readable Stream mein convert karta hai:

```js
const readableStream = streamifier.createReadStream(file.buffer);
```

`.pipe()` Readable ko Writable se connect karta hai — data automatically flow karta hai:

```js
streamifier.createReadStream(file.buffer).pipe(uploadStream);
```

> ⚠️ Bina `.pipe()` ke data flow hi nahi hoga — request hang ho jayega.

---

### Promise — `upload_stream` ko Wrap Karna

`upload_stream` callback-based hai, directly `await` nahi kar sakte — isliye `Promise` mein wrap karte hain.

```js
function uploadToCloudinary(buffer) {
  return new Promise((resolve, reject) => {
    const uploadStream = cloudinary.uploader.upload_stream(
      { folder: 'posts' },
      (error, result) => {
        if (error) reject(error);   // fail
        else resolve(result);        // success
      }
    );
    streamifier.createReadStream(buffer).pipe(uploadStream);
  });
}

const result = await uploadToCloudinary(file.buffer);
```

---

### `Promise.all` vs `Promise.allSettled` — Multiple Files

#### `Promise.all` — Sab ya kuch nahi

```js
const results = await Promise.all([
  uploadToCloudinary(files[0].buffer),
  uploadToCloudinary(files[1].buffer),
]);
```

Ek bhi fail hua toh **sab kuch reject** ho jaata hai — chahe baaki files Cloudinary pe upload ho chuki hon (woh orphan ban jaati hain, manually delete karni padengi).

#### `Promise.allSettled` — Sabka result lo, fail ho ya pass

```js
const results = await Promise.allSettled(
  files.map(file => uploadToCloudinary(file.buffer))
);

const successful = results
  .filter(r => r.status === 'fulfilled')
  .map(r => r.value);

const failed = results
  .filter(r => r.status === 'rejected')
  .map(r => r.reason);
```

| | `Promise.all` | `Promise.allSettled` |
|--|---------------|----------------------|
| Ek fail ho toh | Sab fail | Baaki ka result phir bhi milta hai |
| File upload mein | ❌ Orphan files ka risk | ✅ Safe — partial save ho jaata hai |

> 💡 Multiple file uploads ke liye **`Promise.allSettled` prefer karo**.

---

### `.map()` — Parallel Uploads

```js
const uploadPromises = files.map(file => uploadToCloudinary(file.buffer));
// Sab Promises ek saath banti hain

await Promise.allSettled(uploadPromises);
// Sab ek saath run hoti hain — parallel, sequential nahi
```

---

## 🔗 Part 5 — Full Working Example {#part-5-integration}

### Project Structure

```
project/
├── server.js
├── app.js
├── routes/
│   └── post.routes.js
├── controllers/
│   └── post.controller.js
├── middleware/
│   └── multer.js
├── config/
│   └── cloudinary.config.js
├── public/
│   └── index.html        ← frontend form
├── .env
└── package.json
```

---

#### `config/cloudinary.config.js`

```js
import { v2 as cloudinary } from 'cloudinary';

cloudinary.config({
  cloud_name: process.env.CLOUDINARY_CLOUD_NAME,
  api_key: process.env.CLOUDINARY_API_KEY,
  api_secret: process.env.CLOUDINARY_API_SECRET
});

export default cloudinary;
```

---

#### `middleware/multer.js`

```js
import multer from 'multer';

const storage = multer.memoryStorage();

const fileFilter = (req, file, cb) => {
  if (file.mimetype.startsWith('image/')) {
    cb(null, true);
  } else {
    cb(new Error('Only image files are allowed'), false);
  }
};

const upload = multer({
  storage,
  fileFilter,
  limits: { fileSize: 5 * 1024 * 1024 }
});

export default upload;
```

---

#### `controllers/post.controller.js`

Yahan seedha controller mein hi Cloudinary call kar rahe hain — koi extra service layer ya utility nahi, simple rakhne ke liye.

```js
import cloudinary from '../config/cloudinary.config.js';
import streamifier from 'streamifier';

function uploadToCloudinary(buffer) {
  return new Promise((resolve, reject) => {
    const uploadStream = cloudinary.uploader.upload_stream(
      { resource_type: 'image', folder: 'posts' },
      (error, result) => {
        if (error) reject(error);
        else resolve(result);
      }
    );
    streamifier.createReadStream(buffer).pipe(uploadStream);
  });
}

export const uploadPost = async function (req, res) {
  try {
    const { title, description } = req.body;
    const files = req.files;   // upload.array() ke baad array milega

    if (!files || files.length === 0) {
      return res.status(400).json({ error: 'No image file provided' });
    }

    // Sab files parallel upload karo — ek fail ho toh baaki ka result bhi mile
    const uploadResults = await Promise.allSettled(
      files.map(file => uploadToCloudinary(file.buffer))
    );

    const successful = uploadResults
      .filter(r => r.status === 'fulfilled')
      .map(r => r.value);

    const failedCount = uploadResults.filter(r => r.status === 'rejected').length;

    if (successful.length === 0) {
      return res.status(400).json({ error: 'All image uploads failed' });
    }

    // Result mein secure_url + public_id wapas bhejo (DB save yahan optional hai)
    const uploaded = successful.map(result => ({
      title,
      description,
      imageUrl: result.secure_url,
      publicId: result.public_id
    }));

    res.json({
      success: true,
      message: 'Post uploaded successfully',
      data: { uploaded, count: successful.length, failed: failedCount }
    });

  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```

> 💡 Agar DB mein save karna ho, toh `uploaded` array ko apne model (jaise `POST.create()`) ke through save kar dena — yeh doc us part ko skip kar raha hai kyunki database setup alag topic hai.

---

#### `routes/post.routes.js`

```js
import express from 'express';
import upload from '../middleware/multer.js';
import { uploadPost } from '../controllers/post.controller.js';

const router = express.Router();

router.post('/', upload.array('image', 10), uploadPost);
//               ↑ field name 'image' frontend form se match hona chahiye

export default router;
```

---

#### `app.js`

```js
import express from 'express';
import cors from 'cors';
import postRouter from './routes/post.routes.js';

const app = express();

app.use(cors());                 // frontend alag origin se request bhej sake
app.use(express.json());
app.use(express.static('public')); // index.html serve karne ke liye

app.use('/api/posts', postRouter);

export default app;
```

---

#### `server.js`

```js
import 'dotenv/config';
import app from './app.js';

const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
```

---

#### `public/index.html` — Frontend

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Image Upload</title>
</head>
<body>

  <form id="uploadForm">
    <input type="text" name="title" placeholder="Post Title" required>
    <input type="text" name="description" placeholder="Description">
    <input type="file" name="image" id="fileInput" accept="image/*" multiple>
    <button type="submit">Upload</button>
  </form>

  <p id="status"></p>

  <script>
    const form = document.getElementById('uploadForm');
    const status = document.getElementById('status');

    form.addEventListener('submit', async function (e) {
      e.preventDefault();

      const formData = new FormData(form);
      status.textContent = 'Uploading...';

      try {
        const response = await fetch('http://localhost:3000/api/posts', {
          method: 'POST',
          body: formData
        });

        const result = await response.json();

        if (response.ok) {
          status.textContent = `Uploaded ${result.data.count} image(s) ✅`;
        } else {
          status.textContent = 'Error: ' + result.error;
        }
      } catch (err) {
        status.textContent = 'Upload failed: ' + err.message;
      }
    });
  </script>

</body>
</html>
```

---

#### `.env`

```
PORT=3000
CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret
```

---

#### Install Dependencies

```bash
npm install express multer cloudinary streamifier dotenv cors
```

> ⚠️ `package.json` mein `"type": "module"` add karna mat bhoolo, kyunki yahan `import/export` syntax use kiya hai:
```json
{
  "type": "module"
}
```

---

## ⚠️ Common Errors & Fixes {#errors}

| Error | Cause | Fix |
|-------|-------|-----|
| `MulterError: File too large` | File `limits.fileSize` se bada hai | Limit badhao ya chhoti file bhejo |
| `MulterError: Unexpected field` | Form field name route ke field name se match nahi | `upload.array('image')` aur `<input name="image">` same rakho |
| `Only image files allowed` | `fileFilter` ne reject kiya | Sahi image file select karo |
| `Cloudinary: Must supply api_key` | `.env` load nahi hua | `dotenv/config` sabse pehle import karo, `.env` file check karo |
| `req.files is undefined` | Route mein `upload.array()` missing | Middleware route mein add karo |
| `req.file.buffer is undefined` | `diskStorage` use ho raha hai | `memoryStorage` use karo |
| `upload_stream hung — no response` | `.pipe()` missing | `streamifier.createReadStream(buffer).pipe(uploadStream)` zaroor karo |
| File backend tak nahi pahunchi | `enctype="multipart/form-data"` missing | Form tag mein enctype add karo |
| `fetch` se upload fail | Manually `Content-Type` header set kiya | Header mat lagao — browser khud set karega |
| `SyntaxError: Cannot use import` | `package.json` mein `"type": "module"` missing | Add karo |

---

## ✅ Quick Reference Cheatsheet {#cheatsheet}

```html
<!-- ─── HTML Form ─────────────────────────────────────────── -->
<form enctype="multipart/form-data" method="POST">
  <input type="file" name="image" multiple>
</form>
```

```js
// ─── Vanilla JS Upload ──────────────────────────────────────
const formData = new FormData(form);   // ya formData.append('image', file)
fetch(url, { method: 'POST', body: formData });
// ⚠️ Content-Type header MAT lagao

// ─── Multer Setup ───────────────────────────────────────────
const upload = multer({ storage: multer.memoryStorage() });

upload.single('fieldName')           // ek file  → req.file
upload.array('fieldName', 10)        // multiple → req.files
upload.fields([{ name: 'img' }])     // named    → req.files['img']

req.file.buffer        // raw binary data (memoryStorage only)
req.files               // array — upload.array ke baad

// ─── Buffer → Stream → Cloudinary ──────────────────────────
function uploadToCloudinary(buffer) {
  return new Promise((resolve, reject) => {
    const stream = cloudinary.uploader.upload_stream(
      { folder: 'posts' },
      (err, result) => err ? reject(err) : resolve(result)
    );
    streamifier.createReadStream(buffer).pipe(stream);
  });
}

// ─── Multiple Files — Parallel Upload ──────────────────────
const results = await Promise.allSettled(
  files.map(f => uploadToCloudinary(f.buffer))
);
const successful = results.filter(r => r.status === 'fulfilled').map(r => r.value);

// ─── Cloudinary Result ───────────────────────────────────────
result.secure_url   // DB mein save karo
result.public_id    // delete ke liye zaroori
```

---

> 📝 **Summary:** Frontend se `enctype="multipart/form-data"` form (ya `FormData` + `fetch`) se file bhejo → Multer `memoryStorage` se RAM mein Buffer banao → `streamifier` se Readable Stream banao → `.pipe()` se Cloudinary ko do → `Promise.allSettled` se safely multiple files handle karo. No auth, no extra utilities — sirf core flow.
