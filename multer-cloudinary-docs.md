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
4. [Part 3 — Integration](#part-3-integration)
   - Why memoryStorage + upload_stream?
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

## 🔗 Part 3 — Integration {#part-3-integration}

### Why `memoryStorage` + `upload_stream`?

| Approach | Pros | Cons |
|----------|------|------|
| `diskStorage` → Cloudinary | Simple to understand | Creates temp files on server, extra cleanup needed |
| `memoryStorage` → `upload_stream` | No temp files, faster, production-ready | Slightly more complex setup |

For production, always prefer `memoryStorage` + `upload_stream`.

---

### Full Working Example

#### Project Structure

```
project/
├── server.js
├── routes/
│   └── upload.js
├── middleware/
│   └── multer.js
├── config/
│   └── cloudinary.js
├── .env
└── package.json
```

---

#### `config/cloudinary.js`

```js
const cloudinary = require('cloudinary').v2;

cloudinary.config({
  cloud_name: process.env.CLOUDINARY_CLOUD_NAME,
  api_key: process.env.CLOUDINARY_API_KEY,
  api_secret: process.env.CLOUDINARY_API_SECRET
});

module.exports = cloudinary;
```

---

#### `middleware/multer.js`

```js
const multer = require('multer');

const storage = multer.memoryStorage(); // hold file in RAM

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
  limits: { fileSize: 5 * 1024 * 1024 } // 5 MB
});

module.exports = upload;
```

---

#### `routes/upload.js`

```js
const express = require('express');
const router = express.Router();
const multer = require('multer');
const streamifier = require('streamifier');
const cloudinary = require('../config/cloudinary');
const upload = require('../middleware/multer');

// Helper: wrap upload_stream in a Promise
function uploadToCloudinary(buffer) {
  return new Promise((resolve, reject) => {
    const stream = cloudinary.uploader.upload_stream(
      {
        folder: 'my-app/uploads',
        transformation: [{ quality: 'auto' }, { format: 'webp' }]
      },
      (error, result) => {
        if (error) reject(error);
        else resolve(result);
      }
    );
    streamifier.createReadStream(buffer).pipe(stream);
  });
}

// POST /api/upload
router.post('/', upload.single('image'), async (req, res) => {

  // Step 1: Check if Multer found a file
  if (!req.file) {
    return res.status(400).json({ error: 'No file uploaded' });
  }

  try {
    // Step 2: Upload buffer to Cloudinary
    const result = await uploadToCloudinary(req.file.buffer);

    // Step 3: Return the URL
    res.status(200).json({
      message: 'Upload successful',
      url: result.secure_url,
      public_id: result.public_id
    });

  } catch (error) {
    res.status(500).json({ error: 'Cloudinary upload failed', details: error.message });
  }
});

// DELETE /api/upload/:publicId
router.delete('/:publicId', async (req, res) => {
  try {
    const result = await cloudinary.uploader.destroy(req.params.publicId);
    res.json({ message: 'File deleted', result });
  } catch (error) {
    res.status(500).json({ error: 'Delete failed', details: error.message });
  }
});

module.exports = router;
```

---

#### `server.js`

```js
require('dotenv').config();
const express = require('express');
const app = express();

app.use(express.json());
app.use('/api/upload', require('./routes/upload'));

app.listen(3000, () => console.log('Server running on port 3000'));
```

---

#### `.env`

```
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

1. Method: `POST`
2. URL: `http://localhost:3000/api/upload`
3. Body: `form-data`
4. Key: `image` → Type: `File` → Value: choose an image file
5. Send → You'll get back a `secure_url`

---

### Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `MulterError: File too large` | File exceeds `limits.fileSize` | Increase limit or tell user to upload smaller file |
| `MulterError: Unexpected field` | Field name in form doesn't match route (`upload.single('image')` but form uses `photo`) | Make sure field names match |
| `Only image files allowed` | fileFilter rejected non-image | Frontend should validate too |
| `Cloudinary: Must supply api_key` | Config not loaded | Check `.env` and `dotenv.config()` call |
| `Cannot read property 'buffer' of undefined` | `req.file` is undefined — multer didn't run | Ensure middleware is added to the route |
| `ENOENT: no such file or directory` | diskStorage destination folder doesn't exist | Create the `uploads/` folder manually or with `fs.mkdirSync` |

---

## ✅ Quick Reference Cheatsheet

```js
// ─── Multer ────────────────────────────────────────────────
const upload = multer({ storage: multer.memoryStorage() });

upload.single('fieldName')          // one file → req.file
upload.array('fieldName', 5)        // multiple files → req.files
upload.fields([{ name: 'img' }])    // named fields → req.files['img']

req.file.buffer                     // Buffer (memoryStorage)
req.file.mimetype                   // e.g. 'image/jpeg'
req.file.originalname               // original filename
req.file.size                       // size in bytes

// ─── Cloudinary ────────────────────────────────────────────
cloudinary.uploader.upload(path, options)           // from disk/URL
cloudinary.uploader.upload_stream(options, callback) // from Buffer
cloudinary.uploader.destroy(public_id)              // delete file

result.secure_url    // public HTTPS URL
result.public_id     // unique ID (save this to DB!)
result.format        // file format
result.width         // image width
result.height        // image height
```

---

> 📝 **Summary:** Use **Multer** to receive files in your Express app, and **Cloudinary** to store them in the cloud. The production pattern is `memoryStorage` (Multer) → `upload_stream` (Cloudinary) — no temp files, clean and fast.
