## Create project for back-end

## 📁 Step 1: Project Setup
### 1. Create project folder
```
mkdir project-name // create new folder
cd project-name // join to folder
npm init -y
```
### 2. Install dependencies
Run exactly what you showed:
```
1, npm install express cors dotenv express-async-handler express-validator multer
2, npm install --save-dev nodemon body-parser pg
```
## 🗂️ Step 2: Project Structure
Create a clean folder structure:
```
project-name/
│
├── config/
│   └── db.js
│
├── controllers/
│   └── workController.js
│
├── routes/
│   └── workRoutes.js
│
├── models/
│   └── workModel.js
│
├── services/
│   └── workService.js
│
├── middleware/
│   └── uploadMiddleware.js
│
├── .env
├── index.js
└── package.json
```
## 🔐 Step 3: Environment Variables and PostgreSQL Connection
```.env```
```javascript
PORT=5000

POSTGRES_USER=vanchay
POSTGRES_PASSWORD=123
POSTGRES_HOST=postgres // it's should be postgres or localhost
POSTGRES_DB=portfolio
POSTGRES_PORT=5432
```
```config/db.js```
```javascript
const { Pool } = require("pg");
require("dotenv").config();

const pool = new Pool({
  user: process.env.POSTGRES_USER,
  password: String(process.env.POSTGRES_PASSWORD),
  host: process.env.POSTGRES_HOST,
  database: process.env.POSTGRES_DB,
  port: Number(process.env.POSTGRES_PORT),
});
module.exports = pool;
```
🐳 create db in docker for example_project:
```docker```
```javascript
step 1: docker pull postgres:15
step 2: docker run -d \
          --name postgres \
          -e POSTGRES_USER=vanchay \
          -e POSTGRES_PASSWORD=123 \
          -e POSTGRES_DB=portfolio \
          -p 5432:5432 \
          -v pgdata:/var/lib/postgresql/data \
          postgres:15
```
## 🧱 Step 4: Create Table DB
```config/db.js```
```javascript
const pool = require("../config/db.js");

const scriptDB = async () => {
  try {
    await pool.query(`
      CREATE TABLE IF NOT EXISTS work (
        id SERIAL PRIMARY KEY,
        name VARCHAR(255),
        position VARCHAR(150),
        image TEXT,
        github TEXT,
        demo TEXT,
        framework VARCHAR(255),
        description TEXT,
        created_at TIMESTAMP DEFAULT NOW()
      );

      CREATE TABLE IF NOT EXISTS image_work (
        id SERIAL PRIMARY KEY,
        originalname VARCHAR(255) NOT NULL,
        path TEXT NOT NULL,
        filename VARCHAR(255) NOT NULL,
        size INT NOT NULL,
        encoding VARCHAR(255) NOT NULL,
        by_work INT NOT NULL,
        created_at TIMESTAMP DEFAULT NOW(),
        CONSTRAINT fk_image_work
          FOREIGN KEY (by_work) REFERENCES work(id)
          ON DELETE CASCADE
      );

      CREATE TABLE IF NOT EXISTS technology (
        id SERIAL PRIMARY KEY,
        name VARCHAR(255),
        by_work INT NOT NULL,
        created_at TIMESTAMP DEFAULT NOW(),
        CONSTRAINT fk_technology_work
          FOREIGN KEY (by_work) REFERENCES work(id)
          ON DELETE CASCADE
      );

      CREATE TABLE IF NOT EXISTS technology_tool (
        id SERIAL PRIMARY KEY,
        name VARCHAR(255),
        by_technology INT NOT NULL,
        created_at TIMESTAMP DEFAULT NOW(),
        CONSTRAINT fk_tool_technology
          FOREIGN KEY (by_technology) REFERENCES technology(id)
          ON DELETE CASCADE
      );
    `);

    console.log("✅ All tables created successfully");
  } catch (err) {
    console.error("❌ Error creating tables:", err);
  }
};

module.exports = scriptDB;

```
## 🚀 Step 5: Basic Express Server
```index.js```
```javascript
const express = require("express");
const pool = require("./src/config/db");
const cors = require("cors");
const body_parser = require("body-parser")
const scriptDB = require("./src/config/scriptDB");
const { errHandle, logger } = require("./src/middleware");
const workRouter = require("./src/routes/workRoutes");
const app = express();
require("dotenv").config();

// middleware
app.use(logger); 
pool
  .connect()
  .then(() => console.log("✅ Index.js => Connected to PostgreSQL"))
  .catch((err) => console.error("❌ DB connection error:", err));
scriptDB() // create Table of DB
app.use(cors()); // allow frontend requests
app.use(body_parser.json()) // allow requests body as a json
app.use("/uploads", express.static("uploads")); // This lets the browser access images like: http://localhost:5000/uploads/image-17100022222.png

// route
app.use('/api', workRouter)

app.use(errHandle) // Check error with with middleware

app.listen(process.env.PORT, () => {
  console.log(`✅ Example app listening on port ${process.env.PORT} `);
  console.log(`✅ API === http://localhost:${process.env.PORT}/api`);
});
```
## 🔄 Step 6: Middleware
```middleware/index.js```
```javascript
const { validationResult } = require("express-validator");
const multer = require("multer");
const path = require("path")

const logger = (req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
};

const errHandle = (err, req, res, next) => {
    console.log(err.message)
    return res.status(500).json({
        Message: "SERVER IS DOWN",
        Error: err.message
    })
};

// validatior
const validate = (req, res, next) => {
  const result = validationResult(req)
  if (result.isEmpty()) {
    next();
  } else {
    console.log({ error: result.array() })
    return res.status(401).json({ error: result.array() });
  }
};

// upload
const storage = multer.diskStorage({
    destination: "./uploads/",
    filename: (req, file, cb) => {
        cb(null, file.fieldname +"-"+ Date.now() + path.extname(file.originalname))
    }
})
const upload = multer({
    storage: storage
})

module.exports = {
  logger,
  errHandle,
  validate,
  upload
};
```
## 🧩 Step 7: Models
```models/projectModel.js```
```javascript
const pool = require("../config/db");

const workModel = { 
    // save
    async save(client, data){
        const sql = await client.query(`
            INSERT INTO work(name, position, github, demo, framework, description)
            VALUES $1, $2, $3, $4, $5, $6 RETURNING * 
        `, [data.name, data.position, data.github, data.demo, data.framework, data.description])
        return sql.rows
    },
}
module.exports = workModel
```
## ⚙️ Step 8: Services
```service/workService.js```
```javascript
const pool = require("../config/db")
const workModel = require("../model/workModel")

const workService = {
    // save full of work
    async saveFull(data, files){
        const client = await pool.connect()
        try {
            await client.query("BEGIN")
            // 1 insert work
            const work = await workModel.save(client, data)

            const by_work = work.rows[0].id

            // 2 insert image
            for (const image of files || []) {
                await client.query(`
                    INSERT INTO image_work(originalname, path, filename, size, encoding, by_work)
                    VALUES $1, $2, $3,, $4, $5, $6
                `, [
                    image.originalname,
                    image.path,
                    image.filename,
                    image.size,
                    image.encoding,
                    by_work
                ])
            }
            await client.query("COMMIT")
            return work.rows
        } catch (err) {
            await client.query("ROLLBACK")
            throw err
        } finally {
            client.release()
        }
    }
}

module.exports = workService
```
## 🎮 Step 9: Controllers
```controllers/projectController.js```
```javascript
const expressAsyncHandler = require("express-async-handler");
const workService = require("../services/workService");

const addWork = expressAsyncHandler(async(req, res) => {
    const data = req.body
    const files = req.files
    const result = await workService.saveFull(data, files)
    return res.json({
        message: "Work created successfully",
        result
    })
})
module.exports = { addWork}
```

## 🌐 Step 10: Routes
```routes/projectRoutes.js```
```javascript
const express = require("express")
const { addWork } = require("../controllers/workCon")
const upload = require("../middleware/index")

const workRouter = express.Router()

workRouter.post(
    "/work",
    upload.array("image", 10),
    addWork
)
module.exports = workRouter
```
## 🔌 Step 11: Connect Routes to App
```index.js:```
```javascript
const profileRouter = require('./routes/projectRoutes');

app.use('/api', profileRouter)
```


