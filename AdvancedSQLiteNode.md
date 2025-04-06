# SQLite with Node.js Advanced Techniques

This guide covers advanced techniques for using SQLite with Node.js, focusing on performance optimization, complex queries, and production-ready patterns.

## Table of Contents

- [Advanced Query Techniques](#advanced-query-techniques)
- [Performance Optimization](#performance-optimization)
- [Complex Transactions](#complex-transactions)
- [Advanced Migrations](#advanced-migrations)
- [Concurrency Management](#concurrency-management)
- [Memory Management](#memory-management)
- [Production Deployment](#production-deployment)
- [Security Hardening](#security-hardening)
- [Integration Patterns](#integration-patterns)

## Advanced Query Techniques

### Complex Joins

```javascript
// Multi-table join with aliasing
db.all(
  `
  SELECT u.id, u.name, p.title, c.content
  FROM users u
  LEFT JOIN posts p ON u.id = p.user_id
  LEFT JOIN comments c ON p.id = c.post_id
  WHERE u.id = ?
  ORDER BY p.created_at DESC
`,
  [userId],
  (err, rows) => {
    // Process results
  }
);
```

### Subqueries

```javascript
// Subquery in SELECT
db.all(
  `
  SELECT 
    id, 
    name, 
    (SELECT COUNT(*) FROM posts WHERE user_id = users.id) AS post_count
  FROM users
  WHERE (SELECT COUNT(*) FROM posts WHERE user_id = users.id) > 0
`,
  [],
  (err, rows) => {
    // Process results
  }
);
```

### Window Functions

```javascript
// Ranking users by post count
db.all(
  `
  SELECT 
    u.id, 
    u.name, 
    COUNT(p.id) AS post_count,
    RANK() OVER (ORDER BY COUNT(p.id) DESC) AS rank
  FROM users u
  LEFT JOIN posts p ON u.id = p.user_id
  GROUP BY u.id
  ORDER BY post_count DESC
`,
  [],
  (err, rows) => {
    // Process results
  }
);
```

### Common Table Expressions (CTEs)

```javascript
// Recursive CTE for hierarchical data
db.all(
  `
  WITH RECURSIVE comment_tree AS (
    -- Base case: top-level comments
    SELECT id, content, parent_id, 0 AS depth
    FROM comments
    WHERE parent_id IS NULL AND post_id = ?
    
    UNION ALL
    
    -- Recursive case: replies
    SELECT c.id, c.content, c.parent_id, ct.depth + 1
    FROM comments c
    JOIN comment_tree ct ON c.parent_id = ct.id
  )
  SELECT * FROM comment_tree ORDER BY depth, id
`,
  [postId],
  (err, rows) => {
    // Process results
  }
);
```

### JSON Functions

```javascript
// Store and query JSON data
db.run(`
  CREATE TABLE IF NOT EXISTS settings (
    id INTEGER PRIMARY KEY,
    user_id INTEGER,
    preferences TEXT, -- JSON stored as text
    FOREIGN KEY (user_id) REFERENCES users(id)
  )
`);

// Insert JSON data
const preferences = JSON.stringify({
  theme: "dark",
  notifications: true,
  language: "en",
});

db.run("INSERT INTO settings (user_id, preferences) VALUES (?, ?)", [
  userId,
  preferences,
]);

// Query JSON properties
db.all(
  `
  SELECT 
    user_id, 
    json_extract(preferences, '$.theme') AS theme,
    json_extract(preferences, '$.language') AS language
  FROM settings
  WHERE json_extract(preferences, '$.notifications') = 1
`,
  [],
  (err, rows) => {
    // Process results
  }
);
```

## Performance Optimization

### Indexing Strategies

```javascript
// Create indexes for frequently queried columns
db.serialize(() => {
  // Simple index
  db.run("CREATE INDEX IF NOT EXISTS idx_users_email ON users(email)");

  // Compound index
  db.run(
    "CREATE INDEX IF NOT EXISTS idx_posts_user_created ON posts(user_id, created_at)"
  );

  // Partial index
  db.run(
    'CREATE INDEX IF NOT EXISTS idx_active_users ON users(email) WHERE status = "active"'
  );
});

// Analyze indexes
db.run("ANALYZE");
```

### Query Optimization

```javascript
// Use EXPLAIN QUERY PLAN to understand query execution
db.all(
  `
  EXPLAIN QUERY PLAN
  SELECT * FROM users 
  WHERE email LIKE ? AND status = ?
`,
  ["%example.com", "active"],
  (err, plan) => {
    console.log("Query plan:", plan);
  }
);

// Optimize slow queries
const optimizedQuery = `
  SELECT u.id, u.name
  FROM users u
  INDEXED BY idx_users_email
  WHERE u.email LIKE ?
`;
```

### Statement Caching

```javascript
// Prepare statements once and reuse
const insertStmt = db.prepare(
  "INSERT INTO logs (message, level) VALUES (?, ?)"
);

for (let i = 0; i < 1000; i++) {
  insertStmt.run(`Log message ${i}`, "info");
}

insertStmt.finalize();
```

### WAL Mode Configuration

```javascript
// Enable Write-Ahead Logging for better concurrency
db.run("PRAGMA journal_mode = WAL");

// Configure WAL checkpoint settings
db.run("PRAGMA wal_autocheckpoint = 1000"); // Checkpoint after 1000 pages

// Manual checkpoint
db.run("PRAGMA wal_checkpoint(FULL)");
```

### Memory Optimizations

```javascript
// Configure cache size (in pages)
db.run("PRAGMA cache_size = 10000");

// Configure memory mapping
db.run("PRAGMA mmap_size = 30000000000"); // 30GB

// Optimize temp storage
db.run("PRAGMA temp_store = MEMORY");
```

## Complex Transactions

### Nested Transactions with Savepoints

```javascript
async function complexOperation() {
  try {
    await runAsync("BEGIN TRANSACTION");

    // First part of transaction
    await runAsync("INSERT INTO users (name) VALUES (?)", ["User 1"]);

    // Create savepoint
    await runAsync("SAVEPOINT user_posts");

    try {
      // Operations that might fail
      await runAsync(
        "INSERT INTO posts (user_id, title) VALUES (last_insert_rowid(), ?)",
        ["Post 1"]
      );
      await runAsync(
        "INSERT INTO posts (user_id, title) VALUES (last_insert_rowid(), ?)",
        ["Post 2"]
      );
    } catch (err) {
      // Rollback to savepoint only
      await runAsync("ROLLBACK TO SAVEPOINT user_posts");
      console.log("Posts failed, but user was created");
    }

    // Release savepoint
    await runAsync("RELEASE SAVEPOINT user_posts");

    // Commit the transaction
    await runAsync("COMMIT");
  } catch (err) {
    // Rollback everything
    await runAsync("ROLLBACK");
    throw err;
  }
}
```

### Transaction Isolation Levels

```javascript
// Set isolation level
db.run("PRAGMA read_uncommitted = 1"); // Enable READ UNCOMMITTED

// Default is SERIALIZABLE in SQLite
db.run("PRAGMA read_uncommitted = 0"); // Back to SERIALIZABLE
```

## Advanced Migrations

### Version-Based Migration System

```javascript
class MigrationManager {
  constructor(db) {
    this.db = db;
  }

  async initialize() {
    await this.runAsync(`
      CREATE TABLE IF NOT EXISTS migrations (
        id INTEGER PRIMARY KEY,
        version INTEGER UNIQUE,
        name TEXT,
        applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      )
    `);
  }

  async getCurrentVersion() {
    const row = await this.getAsync(
      "SELECT MAX(version) as version FROM migrations"
    );
    return row?.version || 0;
  }

  async migrate(targetVersion = Infinity) {
    await this.initialize();
    const currentVersion = await this.getCurrentVersion();

    // Get pending migrations
    const pendingMigrations = this.migrations
      .filter((m) => m.version > currentVersion && m.version <= targetVersion)
      .sort((a, b) => a.version - b.version);

    if (pendingMigrations.length === 0) {
      return { applied: 0, currentVersion };
    }

    // Apply each migration in a transaction
    let appliedCount = 0;

    for (const migration of pendingMigrations) {
      try {
        await this.runAsync("BEGIN TRANSACTION");

        // Run the up migration
        await this.runAsync(migration.up);

        // Record the migration
        await this.runAsync(
          "INSERT INTO migrations (version, name) VALUES (?, ?)",
          [migration.version, migration.name]
        );

        await this.runAsync("COMMIT");
        appliedCount++;
      } catch (err) {
        await this.runAsync("ROLLBACK");
        throw new Error(
          `Migration ${migration.version} failed: ${err.message}`
        );
      }
    }

    const newVersion = await this.getCurrentVersion();
    return { applied: appliedCount, currentVersion: newVersion };
  }

  // Helper methods
  runAsync(sql, params = []) {
    return new Promise((resolve, reject) => {
      this.db.run(sql, params, function (err) {
        if (err) reject(err);
        else resolve({ lastID: this.lastID, changes: this.changes });
      });
    });
  }

  getAsync(sql, params = []) {
    return new Promise((resolve, reject) => {
      this.db.get(sql, params, (err, row) => {
        if (err) reject(err);
        else resolve(row);
      });
    });
  }
}
```

### Migration with Data Transformations

```javascript
const migrations = [
  {
    version: 1,
    name: "initial-schema",
    up: `
      CREATE TABLE users (
        id INTEGER PRIMARY KEY,
        name TEXT,
        email TEXT UNIQUE
      );
    `,
  },
  {
    version: 2,
    name: "add-user-status",
    up: `
      ALTER TABLE users ADD COLUMN status TEXT DEFAULT 'active';
    `,
  },
  {
    version: 3,
    name: "split-name-fields",
    up: `
      -- Add new columns
      ALTER TABLE users ADD COLUMN first_name TEXT;
      ALTER TABLE users ADD COLUMN last_name TEXT;
      
      -- Update data
      UPDATE users SET 
        first_name = substr(name, 1, instr(name || ' ', ' ') - 1),
        last_name = substr(name, instr(name || ' ', ' ') + 1)
      WHERE name IS NOT NULL;
    `,
  },
];
```

## Concurrency Management

### Connection Pooling

```javascript
class ConnectionPool {
  constructor(dbPath, options = {}) {
    this.dbPath = dbPath;
    this.maxConnections = options.maxConnections || 5;
    this.connections = [];
    this.waiting = [];
  }

  getConnection() {
    return new Promise((resolve, reject) => {
      // Check for available connection
      const availableConnection = this.connections.find((c) => !c.inUse);

      if (availableConnection) {
        availableConnection.inUse = true;
        return resolve(availableConnection.db);
      }

      // Create new connection if pool not full
      if (this.connections.length < this.maxConnections) {
        try {
          const db = new sqlite3.Database(this.dbPath);

          // Configure connection
          db.run("PRAGMA foreign_keys = ON");
          db.run("PRAGMA journal_mode = WAL");

          const connection = { db, inUse: true };
          this.connections.push(connection);

          return resolve(db);
        } catch (err) {
          return reject(err);
        }
      }

      // Add to waiting queue
      this.waiting.push({ resolve, reject });
    });
  }

  releaseConnection(db) {
    const index = this.connections.findIndex((c) => c.db === db);

    if (index !== -1) {
      // Mark as available
      this.connections[index].inUse = false;

      // Check waiting queue
      if (this.waiting.length > 0) {
        const { resolve } = this.waiting.shift();
        this.connections[index].inUse = true;
        resolve(db);
      }
    }
  }

  closeAll() {
    this.connections.forEach((connection) => {
      connection.db.close();
    });

    this.connections = [];

    // Reject any waiting requests
    this.waiting.forEach(({ reject }) => {
      reject(new Error("Connection pool closed"));
    });

    this.waiting = [];
  }
}
```

### Busy Timeout and Retry Logic

```javascript
function configureConnection(db) {
  // Set busy timeout (milliseconds)
  db.run("PRAGMA busy_timeout = 5000");
}

// Retry logic for SQLITE_BUSY errors
async function runWithRetry(db, sql, params = [], maxRetries = 3) {
  let retries = 0;

  while (true) {
    try {
      return await new Promise((resolve, reject) => {
        db.run(sql, params, function (err) {
          if (err) reject(err);
          else resolve({ lastID: this.lastID, changes: this.changes });
        });
      });
    } catch (err) {
      if (err.code === "SQLITE_BUSY" && retries < maxRetries) {
        retries++;

        // Exponential backoff
        const delay = 100 * Math.pow(2, retries);
        await new Promise((resolve) => setTimeout(resolve, delay));

        continue;
      }

      throw err;
    }
  }
}
```

## Memory Management

### Handling Large Result Sets

```javascript
// Process large result sets in chunks
function processLargeTable(tableName, batchSize = 1000) {
  return new Promise((resolve, reject) => {
    let offset = 0;
    let processedCount = 0;
    let hasMore = true;

    async function processBatch() {
      try {
        const rows = await allAsync(
          `SELECT * FROM ${tableName} LIMIT ? OFFSET ?`,
          [batchSize, offset]
        );

        if (rows.length === 0) {
          hasMore = false;
          return;
        }

        // Process this batch
        for (const row of rows) {
          await processRow(row);
          processedCount++;
        }

        // Move to next batch
        offset += batchSize;

        // Continue if there might be more
        if (rows.length === batchSize) {
          // Use setImmediate to prevent stack overflow
          setImmediate(processBatch);
        } else {
          hasMore = false;
        }
      } catch (err) {
        reject(err);
      }
    }

    // Start processing
    processBatch()
      .then(() => {
        resolve(processedCount);
      })
      .catch(reject);
  });
}

async function processRow(row) {
  // Process individual row
  console.log(`Processing row ${row.id}`);
}
```

### Streaming Results

```javascript
function streamQueryResults(sql, params = []) {
  return new Promise((resolve, reject) => {
    const stream = new Readable({
      objectMode: true,
      read() {}, // This will be pushed to by the each() method
    });

    let count = 0;

    db.each(
      sql,
      params,
      // Row callback
      (err, row) => {
        if (err) {
          stream.emit("error", err);
          return;
        }

        count++;
        stream.push(row);
      },
      // Completion callback
      (err) => {
        if (err) {
          stream.emit("error", err);
          reject(err);
          return;
        }

        // End the stream
        stream.push(null);
        resolve({ stream, count });
      }
    );

    return stream;
  });
}

// Usage example
async function processUserStream() {
  try {
    const { stream, count } = await streamQueryResults("SELECT * FROM users");

    console.log(`Processing ${count} users as a stream`);

    // Process the stream
    stream.on("data", (user) => {
      console.log(`Processing user: ${user.name}`);
      // Do something with each user
    });

    stream.on("error", (err) => {
      console.error("Stream error:", err);
    });

    // Wait for stream to finish
    await new Promise((resolve, reject) => {
      stream.on("end", resolve);
      stream.on("error", reject);
    });

    console.log("Stream processing complete");
  } catch (err) {
    console.error("Error:", err);
  }
}
```

### Memory-Efficient Blob Handling

```javascript
// Store a file as a BLOB
async function storeFile(filePath, description) {
  return new Promise((resolve, reject) => {
    const fileStream = fs.createReadStream(filePath);

    // Create a statement
    const stmt = db.prepare(
      "INSERT INTO files (name, data, description) VALUES (?, ?, ?)"
    );

    fileStream.on("error", (err) => {
      stmt.finalize();
      reject(err);
    });

    // Get file name from path
    const fileName = path.basename(filePath);

    // Read file in chunks and bind to statement
    const chunks = [];

    fileStream.on("data", (chunk) => {
      chunks.push(chunk);
    });

    fileStream.on("end", () => {
      const buffer = Buffer.concat(chunks);

      stmt.run(fileName, buffer, description, function (err) {
        stmt.finalize();

        if (err) {
          reject(err);
        } else {
          resolve({ id: this.lastID, size: buffer.length });
        }
      });
    });
  });
}

// Retrieve a BLOB and stream it
function retrieveFile(fileId, writeStream) {
  return new Promise((resolve, reject) => {
    db.get(
      "SELECT name, data FROM files WHERE id = ?",
      [fileId],
      (err, row) => {
        if (err) {
          return reject(err);
        }

        if (!row) {
          return reject(new Error("File not found"));
        }

        // Write BLOB data to stream
        writeStream.write(row.data);
        writeStream.end();

        resolve({
          name: row.name,
          size: row.data.length,
        });
      }
    );
  });
}

// Usage example - save to file
async function downloadFile(fileId, outputPath) {
  const writeStream = fs.createWriteStream(outputPath);

  try {
    const fileInfo = await retrieveFile(fileId, writeStream);
    console.log(
      `File ${fileInfo.name} (${fileInfo.size} bytes) saved to ${outputPath}`
    );
    return fileInfo;
  } catch (err) {
    // Clean up the file if there was an error
    fs.unlinkSync(outputPath);
    throw err;
  }
}
```

## Production Deployment

### Database Encryption

```javascript
// Using SQLCipher for encryption
const sqlite3 = require("sqlite3-cipher");

// Open encrypted database
const db = new sqlite3.Database("encrypted.db");

// Set encryption key
db.run("PRAGMA key = 'your-secret-key'");

// Change the encryption key
db.run("PRAGMA rekey = 'new-secret-key'");

// Create tables and use as normal
db.run(`CREATE TABLE IF NOT EXISTS sensitive_data (
  id INTEGER PRIMARY KEY,
  user_id INTEGER,
  data TEXT
)`);
```

### Automated Backup Strategy

```javascript
class BackupManager {
  constructor(db, options = {}) {
    this.db = db;
    this.backupDir = options.backupDir || path.join(process.cwd(), "backups");
    this.maxBackups = options.maxBackups || 10;
    this.backupInterval = options.backupInterval || 24 * 60 * 60 * 1000; // 24 hours
    this.compressionLevel = options.compressionLevel || 9; // 0-9, higher = more compression

    // Create backup directory if it doesn't exist
    if (!fs.existsSync(this.backupDir)) {
      fs.mkdirSync(this.backupDir, { recursive: true });
    }
  }

  async backup() {
    const timestamp = new Date().toISOString().replace(/[:.]/g, "-");
    const backupPath = path.join(this.backupDir, `backup-${timestamp}.sqlite`);

    try {
      // Create backup
      await this.runAsync(`VACUUM INTO '${backupPath}'`);

      // Compress the backup
      await this.compressBackup(backupPath);

      // Clean up old backups
      await this.cleanupOldBackups();

      return {
        success: true,
        path: `${backupPath}.gz`,
        timestamp,
      };
    } catch (err) {
      console.error("Backup failed:", err);

      // Clean up failed backup file
      if (fs.existsSync(backupPath)) {
        fs.unlinkSync(backupPath);
      }

      throw err;
    }
  }

  async compressBackup(backupPath) {
    return new Promise((resolve, reject) => {
      const readStream = fs.createReadStream(backupPath);
      const writeStream = fs.createWriteStream(`${backupPath}.gz`);
      const gzip = zlib.createGzip({ level: this.compressionLevel });

      readStream.pipe(gzip).pipe(writeStream);

      writeStream.on("finish", () => {
        // Remove uncompressed file
        fs.unlinkSync(backupPath);
        resolve();
      });

      writeStream.on("error", reject);
      readStream.on("error", reject);
      gzip.on("error", reject);
    });
  }

  async cleanupOldBackups() {
    // Get all backup files
    const files = fs
      .readdirSync(this.backupDir)
      .filter((file) => file.endsWith(".gz"))
      .map((file) => ({
        name: file,
        path: path.join(this.backupDir, file),
        time: fs.statSync(path.join(this.backupDir, file)).mtime.getTime(),
      }))
      .sort((a, b) => b.time - a.time); // Sort newest first

    // Remove old backups
    if (files.length > this.maxBackups) {
      const filesToRemove = files.slice(this.maxBackups);

      for (const file of filesToRemove) {
        fs.unlinkSync(file.path);
        console.log(`Removed old backup: ${file.name}`);
      }
    }
  }

  startScheduledBackups() {
    // Perform initial backup
    this.backup()
      .then((result) => console.log(`Initial backup created: ${result.path}`))
      .catch((err) => console.error("Initial backup failed:", err));

    // Schedule regular backups
    this.intervalId = setInterval(() => {
      this.backup()
        .then((result) =>
          console.log(`Scheduled backup created: ${result.path}`)
        )
        .catch((err) => console.error("Scheduled backup failed:", err));
    }, this.backupInterval);

    return this;
  }

  stopScheduledBackups() {
    if (this.intervalId) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
  }

  // Helper method
  runAsync(sql, params = []) {
    return new Promise((resolve, reject) => {
      this.db.run(sql, params, function (err) {
        if (err) reject(err);
        else resolve({ lastID: this.lastID, changes: this.changes });
      });
    });
  }
}
```

### Health Monitoring

```javascript
class DatabaseMonitor {
  constructor(db) {
    this.db = db;
    this.metrics = {
      queries: 0,
      errors: 0,
      slowQueries: 0,
      lastError: null,
      startTime: Date.now(),
      status: "healthy",
    };

    this.slowQueryThreshold = 500; // ms
  }

  async checkHealth() {
    try {
      // Check if database is responsive
      const start = Date.now();
      await this.runAsync("SELECT 1");
      const responseTime = Date.now() - start;

      // Check database size
      const { size } = await this.getDatabaseSize();

      // Check WAL file size
      const walSize = await this.getWALSize();

      // Check for corruption
      const integrityResult = await this.checkIntegrity();

      return {
        status: integrityResult.valid ? "healthy" : "corrupted",
        responseTime,
        uptime: Date.now() - this.metrics.startTime,
        size,
        walSize,
        metrics: this.metrics,
        integrity: integrityResult,
      };
    } catch (err) {
      this.metrics.errors++;
      this.metrics.lastError = {
        message: err.message,
        time: new Date().toISOString(),
      };
      this.metrics.status = "error";

      return {
        status: "error",
        error: err.message,
        uptime: Date.now() - this.metrics.startTime,
        metrics: this.metrics,
      };
    }
  }

  async getDatabaseSize() {
    // Get database file path
    const dbInfo = await this.getAsync("PRAGMA database_list");
    const mainDbPath = dbInfo.find((db) => db.name === "main")?.file;

    if (!mainDbPath) {
      return { size: 0 };
    }

    // Get file size
    const stats = fs.statSync(mainDbPath);
    return { size: stats.size };
  }

  async getWALSize() {
    // Get database file path
    const dbInfo = await this.getAsync("PRAGMA database_list");
    const mainDbPath = dbInfo.find((db) => db.name === "main")?.file;

    if (!mainDbPath) {
      return 0;
    }

    // Check if WAL file exists
    const walPath = `${mainDbPath}-wal`;

    if (!fs.existsSync(walPath)) {
      return 0;
    }

    // Get WAL file size
    const stats = fs.statSync(walPath);
    return stats.size;
  }

  async checkIntegrity() {
    const result = await this.getAsync("PRAGMA integrity_check");
    return {
      valid: result[0].integrity_check === "ok",
      details: result,
    };
  }

  // Helper methods
  runAsync(sql, params = []) {
    this.metrics.queries++;
    const start = Date.now();

    return new Promise((resolve, reject) => {
      this.db.run(
        sql,
        params,
        function (err) {
          const duration = Date.now() - start;

          if (duration > this.slowQueryThreshold) {
            this.metrics.slowQueries++;
          }

          if (err) {
            this.metrics.errors++;
            this.metrics.lastError = {
              message: err.message,
              query: sql,
              time: new Date().toISOString(),
            };
            reject(err);
          } else {
            resolve({ lastID: this.lastID, changes: this.changes });
          }
        }.bind(this)
      );
    });
  }

  getAsync(sql, params = []) {
    this.metrics.queries++;
    const start = Date.now();

    return new Promise((resolve, reject) => {
      this.db.all(sql, params, (err, rows) => {
        const duration = Date.now() - start;

        if (duration > this.slowQueryThreshold) {
          this.metrics.slowQueries++;
        }

        if (err) {
          this.metrics.errors++;
          this.metrics.lastError = {
            message: err.message,
            query: sql,
            time: new Date().toISOString(),
          };
          reject(err);
        } else {
          resolve(rows);
        }
      });
    });
  }
}
```

## Security Hardening

### Input Validation and Sanitization

```javascript
// Input validation library
const Joi = require("joi");

// Define validation schemas
const schemas = {
  user: Joi.object({
    name: Joi.string().max(100).required(),
    email: Joi.string().email().max(255).required(),
    age: Joi.number().integer().min(0).max(120).optional(),
    status: Joi.string()
      .valid("active", "inactive", "pending")
      .default("pending"),
  }),

  post: Joi.object({
    title: Joi.string().min(3).max(200).required(),
    content: Joi.string().max(10000).required(),
    user_id: Joi.number().integer().positive().required(),
    tags: Joi.array().items(Joi.string().max(50)).max(10).optional(),
  }),
};

// Validation middleware
function validate(schema) {
  return (req, res, next) => {
    const { error, value } = schema.validate(req.body, {
      abortEarly: false,
      stripUnknown: true,
    });

    if (error) {
      const errors = error.details.map((detail) => ({
        field: detail.path.join("."),
        message: detail.message,
      }));

      return res.status(400).json({
        error: "Validation failed",
        details: errors,
      });
    }

    // Replace request body with validated and sanitized data
    req.body = value;
    next();
  };
}

// Usage in routes
app.post("/users", validate(schemas.user), async (req, res) => {
  try {
    const user = await User.create(req.body);
    res.status(201).json(user);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});
```

### SQL Injection Prevention

```javascript
// NEVER do this - vulnerable to SQL injection
function unsafeSearch(name) {
  return new Promise((resolve, reject) => {
    // DANGEROUS: Direct string interpolation
    db.all(`SELECT * FROM users WHERE name LIKE '%${name}%'`, (err, rows) => {
      if (err) reject(err);
      else resolve(rows);
    });
  });
}

// ALWAYS use parameterized queries
function safeSearch(name) {
  return new Promise((resolve, reject) => {
    // SAFE: Using parameters
    db.all(
      "SELECT * FROM users WHERE name LIKE ?",
      [`%${name}%`],
      (err, rows) => {
        if (err) reject(err);
        else resolve(rows);
      }
    );
  });
}

// For dynamic queries, build them safely
function safeDynamicQuery(filters) {
  let sql = "SELECT * FROM users WHERE 1=1";
  const params = [];

  if (filters.name) {
    sql += " AND name LIKE ?";
    params.push(`%${filters.name}%`);
  }

  if (filters.status) {
    sql += " AND status = ?";
    params.push(filters.status);
  }

  if (filters.minAge) {
    sql += " AND age >= ?";
    params.push(filters.minAge);
  }

  return new Promise((resolve, reject) => {
    db.all(sql, params, (err, rows) => {
      if (err) reject(err);
      else resolve(rows);
    });
  });
}

// For table/column names (which can't be parameterized)
function safeOrderBy(table, column, direction) {
  // Whitelist of allowed tables
  const allowedTables = ["users", "posts", "comments"];

  // Whitelist of allowed columns per table
  const allowedColumns = {
    users: ["id", "name", "email", "created_at"],
    posts: ["id", "title", "created_at", "user_id"],
    comments: ["id", "content", "created_at", "user_id", "post_id"],
  };

  // Validate table
  if (!allowedTables.includes(table)) {
    throw new Error("Invalid table name");
  }

  // Validate column
  if (!allowedColumns[table].includes(column)) {
    throw new Error("Invalid column name");
  }

  // Validate direction
  const dir = direction.toUpperCase() === "DESC" ? "DESC" : "ASC";

  // Build and execute query
  const sql = `SELECT * FROM ${table} ORDER BY ${column} ${dir}`;

  return new Promise((resolve, reject) => {
    db.all(sql, [], (err, rows) => {
      if (err) reject(err);
      else resolve(rows);
    });
  });
}
```

### Access Control

```javascript
// Database-level permissions
function setupDatabasePermissions() {
  // Create roles table
  db.run(`
    CREATE TABLE IF NOT EXISTS roles (
      id INTEGER PRIMARY KEY,
      name TEXT UNIQUE
    )
  `);

  // Create user_roles table
  db.run(`
    CREATE TABLE IF NOT EXISTS user_roles (
      user_id INTEGER,
      role_id INTEGER,
      PRIMARY KEY (user_id, role_id),
      FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
      FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE
    )
  `);

  // Create permissions table
  db.run(`
    CREATE TABLE IF NOT EXISTS permissions (
      id INTEGER PRIMARY KEY,
      name TEXT UNIQUE,
      description TEXT
    )
  `);

  // Create role_permissions table
  db.run(`
    CREATE TABLE IF NOT EXISTS role_permissions (
      role_id INTEGER,
      permission_id INTEGER,
      PRIMARY KEY (role_id, permission_id),
      FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE,
      FOREIGN KEY (permission_id) REFERENCES permissions(id) ON DELETE CASCADE
    )
  `);
}

// Check if user has permission
async function hasPermission(userId, permissionName) {
  const sql = `
    SELECT 1 FROM user_roles ur
    JOIN role_permissions rp ON ur.role_id = rp.role_id
    JOIN permissions p ON rp.permission_id = p.id
    WHERE ur.user_id = ? AND p.name = ?
    LIMIT 1
  `;

  const row = await getAsync(sql, [userId, permissionName]);
  return !!row;
}

// Middleware to check permissions
function requirePermission(permissionName) {
  return async (req, res, next) => {
    try {
      const userId = req.user.id; // Assuming authentication middleware sets req.user

      if (await hasPermission(userId, permissionName)) {
        return next();
      }

      res.status(403).json({ error: "Permission denied" });
    } catch (err) {
      res.status(500).json({ error: "Error checking permissions" });
    }
  };
}

// Usage in routes
app.delete(
  "/users/:id",
  requirePermission("delete:users"),
  async (req, res) => {
    try {
      await User.delete(req.params.id);
      res.status(204).send();
    } catch (err) {
      res.status(500).json({ error: err.message });
    }
  }
);
```

### Data Encryption

```javascript
const crypto = require("crypto");

// Encryption helper
class Encryptor {
  constructor(encryptionKey) {
    // Key should be 32 bytes (256 bits)
    this.key = Buffer.from(encryptionKey, "hex");
    this.algorithm = "aes-256-gcm";
  }

  encrypt(text) {
    // Generate random initialization vector
    const iv = crypto.randomBytes(16);

    // Create cipher
    const cipher = crypto.createCipheriv(this.algorithm, this.key, iv);

    // Encrypt the data
    let encrypted = cipher.update(text, "utf8", "hex");
    encrypted += cipher.final("hex");

    // Get authentication tag
    const authTag = cipher.getAuthTag().toString("hex");

    // Return IV, encrypted data, and auth tag
    return {
      iv: iv.toString("hex"),
      encrypted,
      authTag,
    };
  }

  decrypt(data) {
    // Convert hex strings back to buffers
    const iv = Buffer.from(data.iv, "hex");
    const authTag = Buffer.from(data.authTag, "hex");

    // Create decipher
    const decipher = crypto.createDecipheriv(this.algorithm, this.key, iv);
    decipher.setAuthTag(authTag);

    // Decrypt the data
    let decrypted = decipher.update(data.encrypted, "hex", "utf8");
    decrypted += decipher.final("utf8");

    return decrypted;
  }
}

// Usage with database
async function storeEncryptedData(userId, sensitiveData) {
  // Create encryptor with application encryption key
  const encryptor = new Encryptor(process.env.ENCRYPTION_KEY);

  // Encrypt the data
  const encrypted = encryptor.encrypt(JSON.stringify(sensitiveData));

  // Store in database
  return runAsync(
    "INSERT INTO encrypted_data (user_id, data, iv, auth_tag) VALUES (?, ?, ?, ?)",
    [userId, encrypted.encrypted, encrypted.iv, encrypted.authTag]
  );
}

async function retrieveEncryptedData(dataId) {
  // Get encrypted data from database
  const row = await getAsync(
    "SELECT data, iv, auth_tag FROM encrypted_data WHERE id = ?",
    [dataId]
  );

  if (!row) {
    throw new Error("Data not found");
  }

  // Create encryptor with application encryption key
  const encryptor = new Encryptor(process.env.ENCRYPTION_KEY);

  // Decrypt the data
  const decrypted = encryptor.decrypt({
    encrypted: row.data,
    iv: row.iv,
    authTag: row.auth_tag,
  });

  // Parse JSON data
  return JSON.parse(decrypted);
}
```

## Integration Patterns

### Repository Pattern

```javascript
// Base repository class
class Repository {
  constructor(db, tableName) {
    this.db = db;
    this.tableName = tableName;
  }

  async findAll(options = {}) {
    let sql = `SELECT * FROM ${this.tableName}`;
    const params = [];

    // Add WHERE clause if filters provided
    if (options.where) {
      const whereClauses = [];

      for (const [key, value] of Object.entries(options.where)) {
        whereClauses.push(`${key} = ?`);
        params.push(value);
      }

      if (whereClauses.length > 0) {
        sql += ` WHERE ${whereClauses.join(" AND ")}`;
      }
    }

    // Add ORDER BY if specified
    if (options.orderBy) {
      const { column, direction = "ASC" } = options.orderBy;
      sql += ` ORDER BY ${column} ${direction}`;
    }

    // Add LIMIT and OFFSET if specified
    if (options.limit) {
      sql += ` LIMIT ?`;
      params.push(options.limit);

      if (options.offset) {
        sql += ` OFFSET ?`;
        params.push(options.offset);
      }
    }

    return this.allAsync(sql, params);
  }

  async findById(id) {
    return this.getAsync(`SELECT * FROM ${this.tableName} WHERE id = ?`, [id]);
  }

  async findOne(where) {
    const whereClauses = [];
    const params = [];

    for (const [key, value] of Object.entries(where)) {
      whereClauses.push(`${key} = ?`);
      params.push(value);
    }

    const sql = `SELECT * FROM ${this.tableName} WHERE ${whereClauses.join(
      " AND "
    )} LIMIT 1`;

    return this.getAsync(sql, params);
  }

  async create(data) {
    const columns = Object.keys(data).join(", ");
    const placeholders = Object.keys(data)
      .map(() => "?")
      .join(", ");
    const values = Object.values(data);

    const sql = `INSERT INTO ${this.tableName} (${columns}) VALUES (${placeholders})`;

    const result = await this.runAsync(sql, values);

    return {
      id: result.lastID,
      ...data,
    };
  }

  async update(id, data) {
    const setClause = Object.keys(data)
      .map((key) => `${key} = ?`)
      .join(", ");
    const values = [...Object.values(data), id];

    const sql = `UPDATE ${this.tableName} SET ${setClause} WHERE id = ?`;

    const result = await this.runAsync(sql, values);

    return {
      changes: result.changes,
      id,
    };
  }

  async delete(id) {
    const sql = `DELETE FROM ${this.tableName} WHERE id = ?`;

    const result = await this.runAsync(sql, [id]);

    return {
      changes: result.changes,
    };
  }

  // Helper methods
  runAsync(sql, params = []) {
    return new Promise((resolve, reject) => {
      this.db.run(sql, params, function (err) {
        if (err) reject(err);
        else resolve({ lastID: this.lastID, changes: this.changes });
      });
    });
  }

  getAsync(sql, params = []) {
    return new Promise((resolve, reject) => {
      this.db.get(sql, params, (err, row) => {
        if (err) reject(err);
        else resolve(row);
      });
    });
  }

  allAsync(sql, params = []) {
    return new Promise((resolve, reject) => {
      this.db.all(sql, params, (err, rows) => {
        if (err) reject(err);
        else resolve(rows);
      });
    });
  }
}

// User repository
class UserRepository extends Repository {
  constructor(db) {
    super(db, "users");
  }

  // Add user-specific methods
  async findByEmail(email) {
    return this.getAsync("SELECT * FROM users WHERE email = ?", [email]);
  }

  async findWithPosts(userId) {
    const user = await this.findById(userId);

    if (!user) {
      return null;
    }

    const posts = await this.allAsync("SELECT * FROM posts WHERE user_id = ?", [
      userId,
    ]);

    return {
      ...user,
      posts,
    };
  }
}
```

### Service Layer

```javascript
// User service
class UserService {
  constructor(userRepository, postRepository) {
    this.userRepository = userRepository;
    this.postRepository = postRepository;
  }

  async createUser(userData) {
    // Check if email already exists
    const existingUser = await this.userRepository.findByEmail(userData.email);

    if (existingUser) {
      throw new Error("Email already in use");
    }

    // Hash password if provided
    if (userData.password) {
      userData.password = await this.hashPassword(userData.password);
    }

    // Create user
    return this.userRepository.create(userData);
  }

  async getUserProfile(userId) {
    // Get user with posts
    const user = await this.userRepository.findWithPosts(userId);

    if (!user) {
      throw new Error("User not found");
    }

    // Remove sensitive information
    delete user.password;

    return user;
  }

  async updateUser(userId, userData) {
    // Check if user exists
    const user = await this.userRepository.findById(userId);

    if (!user) {
      throw new Error("User not found");
    }

    // If updating email, check if it's already in use
    if (userData.email && userData.email !== user.email) {
      const existingUser = await this.userRepository.findByEmail(
        userData.email
      );

      if (existingUser) {
        throw new Error("Email already in use");
      }
    }

    // Update user
    return this.userRepository.update(userId, userData);
  }

  async deleteUser(userId) {
    // Check if user exists
    const user = await this.userRepository.findById(userId);

    if (!user) {
      throw new Error("User not found");
    }

    // Delete user (posts will be deleted by foreign key constraint)
    return this.userRepository.delete(userId);
  }

  // Helper methods
  async hashPassword(password) {
    // In a real app, use bcrypt or similar
    return crypto.createHash("sha256").update(password).digest("hex");
  }
}
```

### Event-Driven Architecture

```javascript
// Event emitter for database changes
const EventEmitter = require("events");

class DatabaseEvents extends EventEmitter {
  constructor() {
    super();
    // Set higher limit for listeners
    this.setMaxListeners(50);
  }
}

const dbEvents = new DatabaseEvents();

// Enhanced repository with events
class EventedRepository extends Repository {
  constructor(db, tableName) {
    super(db, tableName);
    this.events = dbEvents;
  }

  async create(data) {
    const result = await super.create(data);

    // Emit create event
    this.events.emit(`${this.tableName}:created`, result);

    return result;
  }

  async update(id, data) {
    // Get the original record
    const original = await this.findById(id);

    if (!original) {
      throw new Error(`${this.tableName} with id ${id} not found`);
    }

    const result = await super.update(id, data);

    // Emit update event with before/after
    this.events.emit(`${this.tableName}:updated`, {
      id,
      changes: result.changes,
      before: original,
      after: { ...original, ...data },
    });

    return result;
  }

  async delete(id) {
    // Get the original record
    const original = await this.findById(id);

    if (!original) {
      throw new Error(`${this.tableName} with id ${id} not found`);
    }

    const result = await super.delete(id);

    // Emit delete event
    this.events.emit(`${this.tableName}:deleted`, {
      id,
      changes: result.changes,
      record: original,
    });

    return result;
  }
}

// Example event listeners
function setupEventListeners() {
  // Listen for user creation
  dbEvents.on("users:created", (user) => {
    console.log(`User created: ${user.id} - ${user.name}`);

    // Send welcome email
    sendWelcomeEmail(user.email, user.name).catch((err) => {
      console.error("Failed to send welcome email:", err);
    });
  });

  // Listen for post creation
  dbEvents.on("posts:created", (post) => {
    console.log(`Post created: ${post.id} - ${post.title}`);

    // Update user's post count
    updateUserPostCount(post.user_id).catch((err) => {
      console.error("Failed to update post count:", err);
    });

    // Index post for search
    indexPostForSearch(post).catch((err) => {
      console.error("Failed to index post:", err);
    });
  });

  // Listen for user deletion
  dbEvents.on("users:deleted", ({ record }) => {
    console.log(`User deleted: ${record.id} - ${record.name}`);

    // Clean up user data
    cleanupUserData(record.id).catch((err) => {
      console.error("Failed to clean up user data:", err);
    });
  });
}

// Example event handlers
async function sendWelcomeEmail(email, name) {
  console.log(`Sending welcome email to ${name} at ${email}`);
  // Email sending logic
}

async function updateUserPostCount(userId) {
  const count = await db.get(
    "SELECT COUNT(*) as count FROM posts WHERE user_id = ?",
    [userId]
  );

  await db.run("UPDATE users SET post_count = ? WHERE id = ?", [
    count.count,
    userId,
  ]);
}

async function indexPostForSearch(post) {
  console.log(`Indexing post: ${post.id} - ${post.title}`);
  // Search indexing logic
}

async function cleanupUserData(userId) {
  console.log(`Cleaning up data for user: ${userId}`);
  // Data cleanup logic
}
```

### Caching Layer

```javascript
// Simple in-memory cache
class Cache {
  constructor(options = {}) {
    this.cache = new Map();
    this.ttl = options.ttl || 60000; // Default TTL: 60 seconds
    this.maxSize = options.maxSize || 1000; // Maximum items in cache
  }

  set(key, value, ttl = this.ttl) {
    // Enforce max size by removing oldest entries
    if (this.cache.size >= this.maxSize) {
      const oldestKey = this.cache.keys().next().value;
      this.cache.delete(oldestKey);
    }

    const expiry = Date.now() + ttl;

    this.cache.set(key, {
      value,
      expiry,
    });

    return value;
  }

  get(key) {
    const item = this.cache.get(key);

    // Return undefined if item doesn't exist
    if (!item) {
      return undefined;
    }

    // Check if item has expired
    if (item.expiry < Date.now()) {
      this.cache.delete(key);
      return undefined;
    }

    return item.value;
  }

  delete(key) {
    return this.cache.delete(key);
  }

  clear() {
    this.cache.clear();
  }

  // Clear expired items (can be called periodically)
  clearExpired() {
    const now = Date.now();

    for (const [key, item] of this.cache.entries()) {
      if (item.expiry < now) {
        this.cache.delete(key);
      }
    }
  }
}

// Repository with caching
class CachedRepository extends Repository {
  constructor(db, tableName, cache) {
    super(db, tableName);
    this.cache = cache;
  }

  // Generate cache key
  cacheKey(method, params) {
    return `${this.tableName}:${method}:${JSON.stringify(params)}`;
  }

  async findById(id) {
    const cacheKey = this.cacheKey("findById", id);

    // Check cache first
    const cached = this.cache.get(cacheKey);
    if (cached) {
      return cached;
    }

    // Get from database
    const result = await super.findById(id);

    // Cache the result (only if found)
    if (result) {
      this.cache.set(cacheKey, result);
    }

    return result;
  }

  async findAll(options = {}) {
    const cacheKey = this.cacheKey("findAll", options);

    // Check cache first
    const cached = this.cache.get(cacheKey);
    if (cached) {
      return cached;
    }

    // Get from database
    const results = await super.findAll(options);

    // Cache the results
    this.cache.set(cacheKey, results);

    return results;
  }

  async create(data) {
    const result = await super.create(data);

    // Invalidate relevant caches
    this.invalidateCache();

    return result;
  }

  async update(id, data) {
    const result = await super.update(id, data);

    // Invalidate specific and collection caches
    this.cache.delete(this.cacheKey("findById", id));
    this.invalidateCache();

    return result;
  }

  async delete(id) {
    const result = await super.delete(id);

    // Invalidate specific and collection caches
    this.cache.delete(this.cacheKey("findById", id));
    this.invalidateCache();

    return result;
  }

  // Invalidate all cache entries for this repository
  invalidateCache() {
    // This is a simple approach - in a real app, you might want more granular invalidation
    for (const key of this.cache.cache.keys()) {
      if (key.startsWith(`${this.tableName}:`)) {
        this.cache.delete(key);
      }
    }
  }
}

// Usage example
const cache = new Cache({ ttl: 300000 }); // 5 minutes TTL
const userRepository = new CachedRepository(db, "users", cache);
```

### Full-Text Search

```javascript
// Setup FTS tables
function setupFullTextSearch() {
  // Create virtual FTS table for posts
  db.run(`
    CREATE VIRTUAL TABLE IF NOT EXISTS posts_fts USING fts5(
      title, 
      content,
      tags,
      content='posts',
      content_rowid='id'
    )
  `);

  // Create triggers to keep FTS table in sync
  db.run(`
    CREATE TRIGGER IF NOT EXISTS posts_ai AFTER INSERT ON posts BEGIN
      INSERT INTO posts_fts(rowid, title, content, tags)
      VALUES (new.id, new.title, new.content, new.tags);
    END
  `);

  db.run(`
    CREATE TRIGGER IF NOT EXISTS posts_ad AFTER DELETE ON posts BEGIN
      INSERT INTO posts_fts(posts_fts, rowid, title, content, tags)
      VALUES('delete', old.id, old.title, old.content, old.tags);
    END
  `);

  db.run(`
    CREATE TRIGGER IF NOT EXISTS posts_au AFTER UPDATE ON posts BEGIN
      INSERT INTO posts_fts(posts_fts, rowid, title, content, tags)
      VALUES('delete', old.id, old.title, old.content, old.tags);
      INSERT INTO posts_fts(rowid, title, content, tags)
      VALUES (new.id, new.title, new.content, new.tags);
    END
  `);
}

// Search repository
class SearchRepository {
  constructor(db) {
    this.db = db;
  }

  async searchPosts(query, options = {}) {
    const limit = options.limit || 20;
    const offset = options.offset || 0;

    // Build the SQL query
    const sql = `
      SELECT 
        p.id,
        p.title,
        p.content,
        p.user_id,
        p.created_at,
        p.tags,
        u.name as author_name,
        highlight(posts_fts, 0, '<mark>', '</mark>') as title_highlight,
        highlight(posts_fts, 1, '<mark>', '</mark>') as content_highlight,
        rank
      FROM posts_fts
      JOIN posts p ON posts_fts.rowid = p.id
      JOIN users u ON p.user_id = u.id
      WHERE posts_fts MATCH ?
      ORDER BY rank
      LIMIT ? OFFSET ?
    `;

    // Execute the search
    return this.allAsync(sql, [query, limit, offset]);
  }

  // Helper method
  allAsync(sql, params = []) {
    return new Promise((resolve, reject) => {
      this.db.all(sql, params, (err, rows) => {
        if (err) reject(err);
        else resolve(rows);
      });
    });
  }
}

// Usage example
async function searchContent(query) {
  const searchRepo = new SearchRepository(db);

  try {
    const results = await searchRepo.searchPosts(query);

    console.log(`Found ${results.length} results for "${query}"`);

    results.forEach((result) => {
      console.log(`${result.title} (by ${result.author_name})`);
      console.log(`Snippet: ${result.content_highlight.substring(0, 100)}...`);
      console.log("---");
    });

    return results;
  } catch (err) {
    console.error("Search error:", err);
    throw err;
  }
}
```

## Conclusion

This guide covered advanced SQLite techniques for Node.js applications, focusing on performance optimization, complex queries, and production-ready patterns. By implementing these techniques, you can build robust, efficient, and secure applications with SQLite as your database.

Remember that SQLite is a powerful database that can handle significant workloads when properly optimized. For most applications, these advanced techniques will provide excellent performance and reliability.

For extremely high-traffic applications or those with complex concurrency requirements, you might eventually need to consider migrating to a client-server database like PostgreSQL. However, SQLite can take you much further than many developers realize, especially when following these best practices.
