# 📁 Multer + Cloudinary — Complete Documentation

> A beginner-friendly, from-scratch guide to handling file uploads in Node.js using **Multer** and **Cloudinary** — written in documentation style.

---

## 📌 Table of Contents

1. [Overview & Core Flow](#overview)
2. [Part 1 — Multer](#part-1-multer)
   - Installation
   - What is Multer?
   - Storage Engines
   - File Filter
   - Limits
   - Upload Methods
   - Error Handling
3. [Part 2 — Cloudinary](#part-2-cloudinary)
   - What is Cloudinary?
   - Account Setup
   - Configuration
   - Upload a File
   - Upload from Buffer (upload_stream)
   - Delete a File
   - Transformations
   - Folder Management
4. [Part 3 — Core Concepts (Deep Dive)](#part-3-concepts)
   - Buffer — Kya hota hai?
   - Readable Stream — Buffer se Stream kaise banate hain?
   - Streamifier — Kyun use karte hain?
   - .pipe() — Stream ko connect karna
   - upload_stream vs upload()
   - Promise — Async code wrap karna
   - Promise.all vs Promise.allSettled
   - .map() — Parallel uploads
5. [Part 4 — Integration](#part-4-integration)
   - Why memoryStorage + upload_stream?
   - Multer + Route Connection
   - upload.array() — Multiple Files
   - Middleware Order Matters
   - Full Working Example
   - Common Errors & Fixes

---

## 🗺️ Overview & Core Flow {#overview}

**Multer** is a Node.js middleware for handling `multipart/form-data`, which is the encoding type used when a form submits files.

**Cloudinary** is a cloud-based media management service — it stores your images/videos, gives you a public URL, and lets you transform them on the fly.

### How They Work Together

```
User uploads a file (via HTML form or API client)
              ↓
      Express Route receives request
              ↓
      Multer parses the file from the request
      (holds it in memory or saves to disk)
              ↓
      Cloudinary receives the file
      → stores it on the cloud
      → returns a public URL
              ↓
      You save that URL in your database
```

Think of **Multer** as the delivery guy who picks up the file, and **Cloudinary** as the warehouse where the file is stored permanently.

---

## 📦 Part 1 — Multer {#part-1-multer}

### Installation

```bash
npm install multer
```

### What is Multer?

Multer adds a `body` object and a `file` or `files` object to the `request` object. The `body` object contains the text fields of the form, and the `file`/`files` object contains the uploaded files.

> ⚠️ **Important:** Multer only processes `multipart/form-data` requests. It will not work with JSON or URL-encoded bodies.

---

### Storage Engines

Multer gives you two ways to handle where a file goes when it arrives:

#### 1. `diskStorage` — Save file to your server's disk

Use this when you want to save files locally on your machine (not recommended for production).

```js
const multer = require('multer');
const path = require('path');

const storage = multer.diskStorage({

  // destination — folder where file will be saved
  destination: function (req, file, cb) {
    cb(null, 'uploads/'); // 'uploads' folder must exist
  },

  // filename — what to name the saved file
  filename: function (req, file, cb) {
    const uniqueName = Date.now() + '-' + file.originalname;
    cb(null, uniqueName);
  }

});

const upload = multer({ storage: storage });
```

**Key Terms:**

| Term | Meaning |
|------|---------|
| `destination` | Folder on disk where file is saved |
| `filename` | What the saved file will be named |
| `cb` | Callback — first arg is error (null = no error), second arg is the value |
| `file.originalname` | Original name of the uploaded file |

---

#### 2. `memoryStorage` — Hold file in RAM (as a Buffer)

Use this when you want to forward the file to a cloud service like Cloudinary, without saving it to disk first.

```js
const storage = multer.memoryStorage();
const upload = multer({ storage: storage });
```

When using `memoryStorage`, the file is available as `req.file.buffer` — a raw binary Buffer in memory.

> 💡 **Pro Tip:** For production apps with Cloudinary, always use `memoryStorage`. It's faster and cleaner — no temp files left on the server.

---

### File Filter

`fileFilter` lets you accept or reject files based on their type before they are processed.

```js
const upload = multer({
  storage: storage,

  fileFilter: function (req, file, cb) {

    // Accept only image files
    if (file.mimetype.startsWith('image/')) {
      cb(null, true);  // accept the file
    } else {
      cb(new Error('Only image files are allowed!'), false); // reject the file
    }

  }
});
```

**Key Terms:**

| Term | Meaning |
|------|---------|
| `file.mimetype` | The MIME type of the file e.g. `image/jpeg`, `video/mp4` |
| `cb(null, true)` | Accept the file |
| `cb(null, false)` | Silently reject the file |
| `cb(new Error(...), false)` | Reject and throw an error |

**Common MIME types:**

```
image/jpeg     → .jpg / .jpeg
image/png      → .png
image/gif      → .gif
image/webp     → .webp
application/pdf → .pdf
video/mp4      → .mp4
```

---

### Limits

`limits` lets you restrict the size of files or the number of files uploaded.

```js
const upload = multer({
  storage: storage,
  limits: {
    fileSize: 5 * 1024 * 1024,  // 5 MB in bytes
    files: 3                     // max 3 files at once
  }
});
```

**Key Terms:**

| Option | Type | Meaning |
|--------|------|---------|
| `fileSize` | Number (bytes) | Max file size. `1 * 1024 * 1024` = 1 MB |
| `files` | Number | Max number of files per request |
| `fields` | Number | Max number of non-file fields |
| `fieldSize` | Number | Max size of field values in bytes |

---

### Upload Methods

Once you have your `upload` middleware, you attach it to a route.

#### `upload.single(fieldname)`

Accepts **one file** from a field named `fieldname`.

```js
app.post('/upload', upload.single('avatar'), (req, res) => {
  console.log(req.file);   // the uploaded file
  console.log(req.body);   // other form fields
  res.send('File uploaded!');
});
```

`req.file` contains:

| Property | Meaning |
|----------|---------|
| `fieldname` | Field name from the form |
| `originalname` | Original filename |
| `encoding` | Encoding type |
| `mimetype` | MIME type |
| `size` | File size in bytes |
| `buffer` | (memoryStorage only) Raw file data |
| `path` | (diskStorage only) Path where file was saved |
| `filename` | (diskStorage only) Saved filename |

---

#### `upload.array(fieldname, maxCount)`

Accepts **multiple files** from the same field.

```js
app.post('/upload', upload.array('photos', 5), (req, res) => {
  console.log(req.files);  // array of uploaded files
  res.send('Files uploaded!');
});
```

---

#### `upload.fields(fields)`

Accepts **multiple files from multiple different fields**.

```js
app.post('/upload',
  upload.fields([
    { name: 'avatar', maxCount: 1 },
    { name: 'gallery', maxCount: 5 }
  ]),
  (req, res) => {
    console.log(req.files['avatar']);   // single file
    console.log(req.files['gallery']);  // array of files
    res.send('Done!');
  }
);
```

---

#### `upload.none()`

Accepts **only text fields**, no files. Useful for parsing multipart forms without files.

```js
app.post('/profile', upload.none(), (req, res) => {
  console.log(req.body);
});
```

---

### Error Handling

Multer throws two types of errors:

1. `MulterError` — errors from Multer itself (e.g., file too large)
2. Generic `Error` — errors thrown from your own code (e.g., wrong file type)

```js
app.post('/upload', (req, res) => {

  upload.single('file')(req, res, function (err) {

    if (err instanceof multer.MulterError) {
      // Multer-specific error
      if (err.code === 'LIMIT_FILE_SIZE') {
        return res.status(400).json({ error: 'File is too large. Max 5MB allowed.' });
      }
      return res.status(400).json({ error: err.message });

    } else if (err) {
      // Custom error from fileFilter
      return res.status(400).json({ error: err.message });
    }

    // No error — file uploaded successfully
    res.json({ message: 'Upload successful', file: req.file });
  });

});
```

**Common MulterError codes:**

| Code | Meaning |
|------|---------|
| `LIMIT_FILE_SIZE` | File exceeds the size limit |
| `LIMIT_FILE_COUNT` | Too many files uploaded |
| `LIMIT_UNEXPECTED_FILE` | Field name not expected |
| `LIMIT_FIELD_VALUE` | Field value too large |

---

## ☁️ Part 2 — Cloudinary {#part-2-cloudinary}

### What is Cloudinary?

Cloudinary is a **cloud media management platform**. You upload an image or video to Cloudinary, and it:

- Stores the file on its servers
- Gives you a **public URL** you can use anywhere
- Lets you **transform** the file (resize, crop, compress, change format) by modifying the URL
- Handles CDN delivery globally

---

### Account Setup

1. Go to [cloudinary.com](https://cloudinary.com) and create a free account
2. After login, go to your **Dashboard**
3. You'll see your credentials:
   - `Cloud Name`
   - `API Key`
   - `API Secret`

---

### Installation

```bash
npm install cloudinary
```

---

### Configuration

```js
const cloudinary = require('cloudinary').v2;

cloudinary.config({
  cloud_name: 'your_cloud_name',   // from dashboard
  api_key: 'your_api_key',         // from dashboard
  api_secret: 'your_api_secret'    // from dashboard
});
```

> 🔒 **Security:** Never hardcode credentials. Use environment variables:

```js
// .env file
CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret
```

```js
// In your code
require('dotenv').config();

cloudinary.config({
  cloud_name: process.env.CLOUDINARY_CLOUD_NAME,
  api_key: process.env.CLOUDINARY_API_KEY,
  api_secret: process.env.CLOUDINARY_API_SECRET
});
```

---

### Upload a File

#### From a local file path

```js
const result = await cloudinary.uploader.upload('path/to/image.jpg');
console.log(result.secure_url); // https://res.cloudinary.com/...
```

#### From a URL

```js
const result = await cloudinary.uploader.upload('https://example.com/image.jpg');
console.log(result.secure_url);
```

**The `result` object contains:**

| Property | Meaning |
|----------|---------|
| `public_id` | Unique ID of the file on Cloudinary |
| `secure_url` | HTTPS URL to access the file |
| `url` | HTTP URL (use `secure_url` instead) |
| `format` | File format (jpg, png, etc.) |
| `width` | Width of the image in pixels |
| `height` | Height of the image in pixels |
| `bytes` | File size in bytes |
| `created_at` | Timestamp of upload |

---

### Upload from Buffer — `upload_stream`

When you use Multer's `memoryStorage`, the file lives as a `Buffer` in `req.file.buffer`. Cloudinary's regular `upload()` doesn't accept a Buffer directly — you use `upload_stream` instead.

```js
const streamifier = require('streamifier'); // npm install streamifier

function uploadToCloudinary(buffer) {
  return new Promise((resolve, reject) => {
    
    const stream = cloudinary.uploader.upload_stream(
      { folder: 'my-app' },      // options
      (error, result) => {
        if (error) reject(error);
        else resolve(result);
      }
    );

    streamifier.createReadStream(buffer).pipe(stream);
  });
}

// Usage in a route
const result = await uploadToCloudinary(req.file.buffer);
console.log(result.secure_url);
```

**What's happening here:**
1. `upload_stream` creates a writable stream that Cloudinary listens to
2. `streamifier.createReadStream(buffer)` converts the Buffer into a readable stream
3. `.pipe(stream)` sends the data from the Buffer → into Cloudinary's stream

---

### Delete a File

To delete a file from Cloudinary, you need its `public_id`.

```js
const result = await cloudinary.uploader.destroy('my-app/image-public-id');
console.log(result); // { result: 'ok' }
```

> 💡 Always save the `public_id` in your database alongside the `secure_url`, so you can delete files later.

---

### Transformations

Cloudinary lets you transform images **on the fly** by passing options during upload or by modifying the URL.

#### During upload

```js
const result = await cloudinary.uploader.upload('image.jpg', {
  transformation: [
    { width: 500, height: 500, crop: 'fill' },  // resize to 500x500
    { quality: 'auto' },                          // auto compress
    { format: 'webp' }                            // convert to webp
  ]
});
```

#### Via URL

Cloudinary URLs follow this pattern:
```
https://res.cloudinary.com/{cloud_name}/image/upload/{transformations}/{public_id}
```

Examples:
```
# Resize to 300x300
.../image/upload/w_300,h_300/my-image

# Crop to face
.../image/upload/w_200,h_200,g_face,c_thumb/my-image

# Convert to WebP with quality 80
.../image/upload/q_80,f_webp/my-image
```

**Common transformation options:**

| Option | Meaning | Example |
|--------|---------|---------|
| `width` / `w` | Width in pixels | `width: 300` |
| `height` / `h` | Height in pixels | `height: 300` |
| `crop` / `c` | Crop mode | `crop: 'fill'` |
| `gravity` / `g` | Focal point | `gravity: 'face'` |
| `quality` / `q` | Compression (1-100 or 'auto') | `quality: 'auto'` |
| `format` / `f` | Output format | `format: 'webp'` |
| `radius` / `r` | Border radius | `radius: 'max'` (circle) |

**Crop modes:**

| Mode | Behaviour |
|------|-----------|
| `fill` | Fill the exact dimensions, crop edges |
| `fit` | Fit inside dimensions, no crop |
| `thumb` | Smart thumbnail |
| `scale` | Scale proportionally |
| `pad` | Add padding to fill dimensions |

---

### Folder Management

Organize your uploads into folders using the `folder` option:

```js
await cloudinary.uploader.upload('image.jpg', {
  folder: 'users/avatars'
});
// public_id will be: users/avatars/random-id
```

List all files in a folder:

```js
const result = await cloudinary.api.resources({
  type: 'upload',
  prefix: 'users/avatars'
});
console.log(result.resources);
```

---

## 🧠 Part 3 — Core Concepts (Deep Dive) {#part-3-concepts}

Yeh woh cheezein hain jo actually andar ho rahi hain jab tum Multer + Cloudinary use karte ho. Inhe samjhe bina code likha toh jaayega, lekin kuch toot a toh samajh nahi aayega kyun.

---

### Buffer — Kya hota hai?

Jab tum `memoryStorage` use karte ho, Multer file ko disk pe save nahi karta. File ka poora data **RAM mein** ek `Buffer` ke roop mein rakha jaata hai.

```
File (image.jpg)
      ↓
Browser → HTTP Request → Multer
      ↓
memoryStorage → file ko RAM mein rakho
      ↓
req.file.buffer → raw binary data (Buffer)
```

Buffer ek temporary memory container hai jahan raw binary data (0s aur 1s) store hota hai. Yeh JavaScript ka normal string ya object nahi hai — yeh direct machine-level data hai.

```js
console.log(req.file.buffer);
// <Buffer ff d8 ff e0 00 10 4a 46 49 46 ...>
// ↑ yeh image ka raw binary data hai — human readable nahi hota
```

**Key Terms:**

| Term | Meaning |
|------|---------|
| `Buffer` | Raw binary data ka container — Node.js built-in |
| `req.file.buffer` | Single file ka buffer (`upload.single` ke saath) |
| `req.files[i].buffer` | Multiple files mein se ek ka buffer (`upload.array` ke saath) |
| `file.size` | Buffer ka size bytes mein |
| `memoryStorage` | Files ko disk pe nahi, RAM mein Buffer ke roop mein rakho |

> ⚠️ **Important:** `diskStorage` use karo toh `buffer` property nahi milegi — sirf `path` milega. Buffer sirf `memoryStorage` ke saath aata hai.

---

### Readable Stream — Buffer se Stream kaise banate hain?

Cloudinary ka `upload_stream` ek **Writable Stream** expect karta hai — seedha Buffer accept nahi karta. Isliye Buffer ko pehle **Readable Stream** mein convert karna padta hai.

```
Buffer (RAM mein raw data)
      ↓
Readable Stream (data ka flow — chunk by chunk)
      ↓
Cloudinary upload_stream (Writable Stream)
      ↓
Cloudinary servers pe file save
```

**Stream kya hota hai?**

Stream ek data ka flow hai — poora data ek saath nahi bhejte, chunks mein bhejte hain. Jaise paani pipe se aata hai — ek saath poora tank nahi aata, continuously flow karta hai.

| Stream Type | Kaam | Example |
|-------------|------|---------|
| Readable Stream | Data padhna / produce karna | File read karna, Buffer se data nikalna |
| Writable Stream | Data likhna / consume karna | File write karna, Cloudinary upload |
| Duplex Stream | Dono — read aur write | Network socket |

> ⚠️ **Important:** Agar tum seedha `buffer` Cloudinary ko doge — kaam nahi karega. Pehle stream banana padega.

---

### Streamifier — Kyun use karte hain?

`streamifier` ek npm package hai jo Buffer ko cleanly ek Readable Stream mein convert karta hai.

```bash
npm install streamifier
```

```js
import streamifier from 'streamifier';

// Buffer → Readable Stream
const readableStream = streamifier.createReadStream(file.buffer);
```

**`Readable.from()` (Node built-in) vs `streamifier`:**

| | `Readable.from()` | `streamifier` |
|--|-------------------|---------------|
| Built-in | ✅ Haan | ❌ Install karna padega |
| Large files | ⚠️ Kabhi kabhi issue | ✅ Reliable |
| Production use | ⚠️ Caution | ✅ Recommended |
| Backpressure handling | ❌ Basic | ✅ Proper |

> 💡 **Backpressure** = Jab Writable Stream, Readable Stream se slower ho — streamifier isko properly handle karta hai taki data loss na ho. Isliye production mein streamifier prefer karte hain.

---

### `.pipe()` — Stream ko connect karna

`.pipe()` ek Readable Stream ko Writable Stream se connect karta hai — data automatically ek se doosre mein flow karta hai.

```js
readableStream.pipe(writableStream);
```

Real example:

```js
streamifier.createReadStream(file.buffer).pipe(uploadStream);
//          ↑ Readable Stream                  ↑ Writable Stream
//         (Buffer se data niklo)              (Cloudinary ko do)
```

Bina `.pipe()` ke data flow hi nahi hoga — `uploadStream` khali baithega aur response kabhi nahi aayega.

**Key Terms:**

| Term | Meaning |
|------|---------|
| `.pipe()` | Readable ka data Writable mein bhejo |
| `uploadStream` | Cloudinary ka Writable Stream — data receive karta hai |
| `streamifier.createReadStream(buffer)` | Buffer → Readable Stream |

---

### `upload_stream` vs `upload()`

| | `cloudinary.uploader.upload()` | `cloudinary.uploader.upload_stream()` |
|--|------|------|
| Input leta hai | File path ya URL string | Piped stream (Buffer se) |
| Buffer support | ❌ Nahi | ✅ Haan (via `.pipe()`) |
| Use case | `diskStorage` ke saath | `memoryStorage` ke saath |
| Returns | Promise directly | Callback based — Promise mein wrap karo |

```js
// upload() — diskStorage ke saath (file path dete hain)
const result = await cloudinary.uploader.upload('/tmp/image.jpg');

// upload_stream() — memoryStorage + Buffer ke saath
const uploadStream = cloudinary.uploader.upload_stream(
  { folder: 'posts' },         // options
  (error, result) => { ... }   // callback — upload complete hone pe chalega
);
streamifier.createReadStream(buffer).pipe(uploadStream);
```

**`upload_stream` options:**

| Option | Meaning | Example |
|--------|---------|---------|
| `folder` | Cloudinary mein kahan save karo | `'posts'` |
| `resource_type` | File ka type | `'image'`, `'video'`, `'raw'` |
| `public_id` | Custom file name set karo | `'user-123-avatar'` |
| `transformation` | Upload hote waqt transform karo | `[{ width: 500, crop: 'fill' }]` |
| `overwrite` | Same `public_id` pe overwrite karo | `true` / `false` |

---

### Promise — `upload_stream` ko wrap karna

`upload_stream` callback-based hai — directly `await` nahi kar sakte. Isliye isko manually `Promise` mein wrap karte hain.

**Promise kya hota hai?**

Promise ek placeholder hai future value ke liye — abhi value nahi hai, baad mein aayegi jab async kaam complete ho.

```
Promise States:

┌─────────────┐
│   Pending   │  ← abhi kaam chal raha hai
└──────┬──────┘
       │
   ┌───┴────┐
   ↓        ↓
┌──────────┐  ┌──────────┐
│Fulfilled │  │ Rejected │
│(resolve) │  │ (reject) │
└──────────┘  └──────────┘
```

```js
function uploadToCloudinary(buffer) {
  return new Promise((resolve, reject) => {

    const uploadStream = cloudinary.uploader.upload_stream(
      { folder: 'posts' },
      (error, result) => {
        if (error) reject(error);   // ❌ kuch gadbad — Promise fail
        else resolve(result);        // ✅ success — result return karo
      }
    );

    streamifier.createReadStream(buffer).pipe(uploadStream);
  });
}

// Ab await kar sakte hain
const result = await uploadToCloudinary(file.buffer);
console.log(result.secure_url);
```

**Key Terms:**

| Term | Meaning |
|------|---------|
| `new Promise((resolve, reject) => {})` | Promise banana |
| `resolve(value)` | Promise successfully complete karo — value bahar aayegi |
| `reject(error)` | Promise fail karo — error throw hoga |
| `await` | Promise ke complete hone ka wait karo |

---

### `Promise.all` vs `Promise.allSettled` — Multiple Files

Jab multiple files upload karni hain, sabko ek saath (parallel) upload karte hain.

#### `Promise.all` — Sab ya kuch nahi

```js
const results = await Promise.all([
  uploadToCloudinary(files[0].buffer),
  uploadToCloudinary(files[1].buffer),
  uploadToCloudinary(files[2].buffer),
]);
```

```
Behavior:

File 1 ✅ fulfilled
File 2 ✅ fulfilled  →  Sab succeed → results array milega
File 3 ✅ fulfilled

File 1 ✅ fulfilled (Cloudinary pe ja chuki)
File 2 ❌ rejected   →  IMMEDIATELY fail — File 1 ka result bhi nahi milega
File 3 ✅ fulfilled (Cloudinary pe ja chuki)
```

> ⚠️ **Problem:** File 1 aur 3 Cloudinary pe upload ho gayi, lekin `Promise.all` reject hua — DB mein kuch save nahi hua. Woh 2 files Cloudinary pe **orphan** ban ke padi rahengi — storage waste, manually delete karna padega.

---

#### `Promise.allSettled` — Sab ka result lo, chahe fail ho ya pass

```js
const results = await Promise.allSettled([
  uploadToCloudinary(files[0].buffer),
  uploadToCloudinary(files[1].buffer),
  uploadToCloudinary(files[2].buffer),
]);
```

```
Behavior:

File 1 ✅ → { status: 'fulfilled', value: result1  }
File 2 ❌ → { status: 'rejected',  reason: error   }  → SARE results milenge
File 3 ✅ → { status: 'fulfilled', value: result3  }
```

```js
const results = await Promise.allSettled(
  files.map(file => uploadToCloudinary(file.buffer))
);

// Successful uploads nikalo
const successful = results
  .filter(r => r.status === 'fulfilled')
  .map(r => r.value);      // r.value = cloudinary result object

// Failed uploads nikalo (optional — log karo ya return karo)
const failed = results
  .filter(r => r.status === 'rejected')
  .map(r => r.reason);     // r.reason = error object

if (successful.length === 0) {
  throw new Error('Saari images upload fail ho gayi');
}

// Sirf successful wali DB mein save karo
const posts = await Promise.all(
  successful.map(result =>
    POST.create({
      imageUrl: result.secure_url,
      publicId: result.public_id,
      // ...
    })
  )
);
```

**Comparison:**

| | `Promise.all` | `Promise.allSettled` |
|--|---------------|----------------------|
| Ek bhi fail ho toh | Sab fail — immediately reject | Baaki ka result phir bhi aata hai |
| Return karta hai | Array of values | Array of `{ status, value/reason }` |
| Kab use karo | Sab ka succeed karna zaroori ho | Partial success acceptable ho |
| File upload mein | ❌ Orphan files ka risk | ✅ Safe — partial save ho jaata hai |

---

### `.map()` — Parallel Uploads

`files.map()` har file pe ek operation run karta hai aur results ka naya array banata hai.

```js
// files = [file1, file2, file3]

const uploadPromises = files.map(file => uploadToCloudinary(file.buffer));
// uploadPromises = [Promise1, Promise2, Promise3]
//                  ↑ har file ke liye ek alag Promise bana

await Promise.allSettled(uploadPromises);
// Sab Promises ek saath run hoti hain — parallel, sequential nahi
```

```
Sequential (❌ slow):           Parallel (.map + Promise.allSettled) (✅ fast):

Upload File 1 → wait           Upload File 1 ─┐
      ↓                        Upload File 2 ─┼─→ sab ek saath → results
Upload File 2 → wait           Upload File 3 ─┘
      ↓
Upload File 3 → wait
```

> 💡 `.map()` sirf Promises ka array banata hai — actually run karne ke liye `Promise.all` ya `Promise.allSettled` mein wrap karo.

---

## 🔗 Part 4 — Integration {#part-4-integration}

### Why `memoryStorage` + `upload_stream`?

| Approach | Pros | Cons |
|----------|------|------|
| `diskStorage` → Cloudinary | Simple to understand | Creates temp files on server, extra cleanup needed |
| `memoryStorage` → `upload_stream` | No temp files, faster, production-ready | Slightly more complex setup |

For production, always prefer `memoryStorage` + `upload_stream`.

---

### Multer Middleware ko Route se Kaise Connect Karo

Doc mein `middleware/multer.js` banaya — lekin yeh nahi bataya ki route file mein kaise use karo.

```js
// routes/post.routes.js
import express from 'express';
import upload from '../middleware/multer.js';              // multer middleware import karo
import { uploadPost } from '../controllers/post.controller.js';

const router = express.Router();

router.post('/', upload.array('image', 10), uploadPost);
//               ↑
//       middleware as 2nd argument — pehle multer chalta hai, phir controller

export default router;
```

Aur `app.js` / `server.js` mein mount karo:

```js
import postRouter from './routes/post.routes.js';

app.use('/api/posts', postRouter);
// ab POST /api/posts → multer → controller → service → cloudinary
```

---

### `upload.array()` — Multiple Files ke liye

Jab form mein `multiple` attribute ho (`<input type="file" multiple />`), toh route mein `upload.single()` nahi — `upload.array()` lagega:

```js
// ❌ single — sirf ek file lega, multiple ignore karega
router.post('/', upload.single('image'), uploadPost);

// ✅ array — multiple files lega
router.post('/', upload.array('image', 10), uploadPost);
//                              ↑           ↑
//                         field name    max files allowed
```

`upload.array()` ke baad `req.files` milega (array) — `req.file` nahi:

```js
// Controller mein:
const files = req.files;   // ← array of file objects
// files[0].buffer, files[1].buffer, ...

// req.files ka ek element kuch aisa dikhta hai:
{
  fieldname: 'image',
  originalname: 'photo.jpg',
  encoding: '7bit',
  mimetype: 'image/jpeg',
  buffer: <Buffer ff d8 ff ...>,   // ← yahi Cloudinary ko dena hai
  size: 204800
}
```

---

### Middleware Order Matters — Bahut Zaroori

Middleware ka order route mein matter karta hai. Galat order = data missing.

```js
// ✅ Sahi order
router.post('/', authMiddleware, upload.array('image', 10), uploadPost);
//               ↑               ↑                           ↑
//           req.userId set    req.files set             dono available hain

// ❌ Galat — multer pehle chala toh auth ke baad req.userId set hi nahi hoga
router.post('/', upload.array('image', 10), authMiddleware, uploadPost);
```

```
Request aaya
      ↓
authMiddleware   → token verify karo, req.userId set karo
      ↓
upload.array()   → multipart parse karo, req.files set karo
      ↓
uploadPost()     → req.body, req.files, req.userId — sab available
      ↓
postService()    → cloudinary pe upload karo
      ↓
POST.create()    → DB mein save karo
      ↓
successResponse() → client ko URL wapas bhejo
```



### Full Working Example

#### Project Structure

```
project/
├── server.js
├── app.js
├── routes/
│   └── post.routes.js
├── controllers/
│   └── post.controller.js
├── services/
│   └── post.services.js
├── middleware/
│   └── multer.js
├── config/
│   └── cloudinary.config.js
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

const storage = multer.memoryStorage(); // file RAM mein rakho — no disk

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
  limits: { fileSize: 5 * 1024 * 1024 } // 5 MB max
});

export default upload;
```

---

#### `services/post.services.js`

```js
import cloudinary from '../config/cloudinary.config.js';
import { POST } from '../models/post.model.js';
import streamifier from 'streamifier';

// Helper — ek file ka Buffer Cloudinary pe upload karo
function uploadToCloudinary(buffer) {
  return new Promise((resolve, reject) => {
    const uploadStream = cloudinary.uploader.upload_stream(
      { resource_type: 'image', folder: 'posts' },
      (error, result) => {
        if (error) reject(new Error('Cloudinary upload failed: ' + error.message));
        else resolve(result);
      }
    );
    // Buffer → Readable Stream → pipe → Cloudinary Writable Stream
    streamifier.createReadStream(buffer).pipe(uploadStream);
  });
}

export const uploadPost = async function ({ files, title, description, owner }) {
  if (!files || files.length === 0) throw new Error('No image file provided');

  // Step 1 — sab files ko parallel upload karo
  // Promise.allSettled — ek fail ho toh baaki ka result phir bhi aata hai
  const uploadResults = await Promise.allSettled(
    files.map(file => uploadToCloudinary(file.buffer))
  );

  // Step 2 — successful aur failed alag karo
  const successful = uploadResults
    .filter(r => r.status === 'fulfilled')
    .map(r => r.value);       // r.value = cloudinary result object

  const failedCount = uploadResults
    .filter(r => r.status === 'rejected').length;

  if (successful.length === 0) {
    throw new Error('All image uploads failed');
  }

  // Step 3 — sirf successful uploads DB mein save karo
  const posts = await Promise.all(
    successful.map(result =>
      POST.create({
        title,
        description,
        imageUrl: result.secure_url,
        publicId: result.public_id,
        owner
      })
    )
  );

  return { posts, uploaded: successful.length, failed: failedCount };
};
```

---

#### `controllers/post.controller.js`

```js
import * as postService from '../services/post.services.js';
import { successResponse, errorResponse } from '../utils/apiResponse.js';

export const uploadPost = async function (req, res) {
  try {
    const { title, description } = req.body;  // text fields
    const files = req.files;                   // array of files (upload.array ke baad)

    const result = await postService.uploadPost({
      files,
      title,
      description,
      owner: req.userId   // authMiddleware ne set kiya hoga
    });

    successResponse(res, result, 'Post uploaded successfully');
  } catch (error) {
    errorResponse(res, error.message, 400);
  }
};
```

---

#### `routes/post.routes.js`

```js
import express from 'express';
import upload from '../middleware/multer.js';
import { uploadPost } from '../controllers/post.controller.js';
import authMiddleware from '../middleware/auth.js';

const router = express.Router();

// Order zaroori hai: auth → multer → controller
router.post('/', authMiddleware, upload.array('image', 10), uploadPost);

export default router;
```

---

#### `app.js`

```js
import express from 'express';
import postRouter from './routes/post.routes.js';

const app = express();

app.use(express.json());
app.use(express.urlencoded({ extended: true }));

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

#### `.env`

```
PORT=3000
CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret
```

---

#### Install all dependencies

```bash
npm install express multer cloudinary streamifier dotenv
```

---

### Testing with Postman / Thunder Client

```
POST  http://localhost:3000/api/posts
Headers → Authorization: Bearer <your_token>
Body    → form-data
          title       → Text → "My Post"
          description → Text → "Hello world"
          image       → File → [ek ya zyada images choose karo]
```

Response:
```json
{
  "success": true,
  "message": "Post uploaded successfully",
  "data": {
    "posts": [...],
    "uploaded": 3,
    "failed": 0
  }
}
```

---

### Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `MulterError: File too large` | File exceeds `limits.fileSize` | Limit badhao ya user ko chhoti file bhejne kaho |
| `MulterError: Unexpected field` | Form field name aur route ka name match nahi (`upload.array('image')` but form uses `photo`) | Field names match karo |
| `Only image files allowed` | `fileFilter` ne non-image reject kiya | Frontend pe bhi validate karo |
| `Cloudinary: Must supply api_key` | Config load nahi hua | `.env` check karo aur `dotenv/config` import sabse pehle karo |
| `req.files is undefined` | `upload.array()` nahi lagaya route mein | Middleware route mein add karo |
| `req.file.buffer is undefined` | `diskStorage` use ho raha hai | `memoryStorage` use karo |
| `Cannot read properties of undefined (reading 'buffer')` | `req.file` / `req.files` empty — multer nahi chala | Middleware order check karo, field name check karo |
| `upload_stream hung — no response` | `.pipe()` nahi kiya | `streamifier.createReadStream(buffer).pipe(uploadStream)` zaroor karo |

---

## ✅ Quick Reference Cheatsheet

```js
// ─── Multer Setup ───────────────────────────────────────────
import multer from 'multer';
const upload = multer({ storage: multer.memoryStorage() });

// ─── Upload Methods ─────────────────────────────────────────
upload.single('fieldName')           // ek file  → req.file
upload.array('fieldName', 10)        // multiple → req.files
upload.fields([{ name: 'img' }])     // named    → req.files['img']

// ─── req.file / req.files properties ───────────────────────
req.file.buffer        // raw binary data (memoryStorage only)
req.file.mimetype      // 'image/jpeg', 'image/png' etc.
req.file.originalname  // original filename from user
req.file.size          // size in bytes
req.files              // array — upload.array ke baad

// ─── Buffer → Stream → Cloudinary ──────────────────────────
import streamifier from 'streamifier';

function uploadToCloudinary(buffer) {
  return new Promise((resolve, reject) => {
    const stream = cloudinary.uploader.upload_stream(
      { folder: 'posts', resource_type: 'image' },
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
const failed     = results.filter(r => r.status === 'rejected').map(r => r.reason);

// ─── Cloudinary Methods ─────────────────────────────────────
cloudinary.uploader.upload(path, options)            // disk/URL se
cloudinary.uploader.upload_stream(options, callback) // Buffer se
cloudinary.uploader.destroy(public_id)               // file delete

// ─── Cloudinary Result Object ───────────────────────────────
result.secure_url   // HTTPS URL — yahi DB mein save karo
result.public_id    // unique ID — delete ke liye zaroori, DB mein save karo
result.format       // 'jpg', 'png', 'webp' etc.
result.width        // image width in pixels
result.height       // image height in pixels
result.bytes        // file size in bytes

// ─── Route Order ────────────────────────────────────────────
router.post('/', authMiddleware, upload.array('image', 10), controller);
//               ↑ pehle auth   ↑ phir multer              ↑ phir controller
```

---

> 📝 **Summary:** Use **Multer** to receive files in your Express app, and **Cloudinary** to store them in the cloud. Buffer RAM mein rakho (`memoryStorage`), streamifier se Readable Stream banao, `.pipe()` se Cloudinary ko do, aur `Promise.allSettled` se safely multiple files handle karo — no temp files, clean and fast.
