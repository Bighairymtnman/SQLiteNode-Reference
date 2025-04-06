# SQLite with Node.js Quick Reference

This document provides a quick reference for working with SQLite in Node.js applications. It covers core modules, connection management, query execution, and common patterns.

## Table of Contents

- [Core Modules](#core-modules)
- [Connection Management](#connection-management)
- [Query Execution Methods](#query-execution-methods)
- [Prepared Statements](#prepared-statements)
- [Transactions](#transactions)
- [Error Handling](#error-handling)
- [Data Types and Conversions](#data-types-and-conversions)
- [Configuration Options](#configuration-options)
- [Common Patterns](#common-patterns)
- [Performance Tips](#performance-tips)
- [SQLite-Specific Features](#sqlite-specific-features)
- [Working with Dates](#working-with-dates)
- [Backup and Recovery](#backup-and-recovery)
- [Security Considerations](#security-considerations)

## Core Modules

### sqlite3 Module

```javascript
// Installation
npm install sqlite3
```

```javascript
// Importing the module
const sqlite3 = require("sqlite3").verbose();
```

### better-sqlite3 Module (Alternative)

```javascript
// Installation
npm install better-sqlite3
```

```javascript
// Importing the module
const Database = require("better-sqlite3");
```

## Connection Management

### Creating/Opening a Database

```javascript
// sqlite3
const sqlite3 = require("sqlite3").verbose();

// In-memory database
const memoryDB = new sqlite3.Database(":memory:");

// File database
const fileDB = new sqlite3.Database("./database.sqlite");

// With options
const db = new sqlite3.Database(
  "./database.sqlite",
  sqlite3.OPEN_READWRITE | sqlite3.OPEN_CREATE,
  (err) => {
    if (err) {
      console.error("Database opening error: ", err);
    } else {
      console.log("Database opened successfully");
    }
  }
);
```

```javascript
// better-sqlite3
const Database = require("better-sqlite3");

// In-memory database
const memoryDB = new Database(":memory:");

// File database
const fileDB = new Database("./database.sqlite");

// With options
const db = new Database("./database.sqlite", {
  readonly: false,
  fileMustExist: false,
  timeout: 5000,
});
```

### Closing a Database

```javascript
// sqlite3
db.close((err) => {
  if (err) {
    console.error("Error closing database: ", err);
  } else {
    console.log("Database closed successfully");
  }
});
```

```javascript
// better-sqlite3
db.close();
```

## Query Execution Methods

### Running Queries (sqlite3)

```javascript
// Simple execution (no results)
db.run("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT, email TEXT)");

// With parameters
db.run(
  "INSERT INTO users (name, email) VALUES (?, ?)",
  ["John Doe", "john@example.com"],
  function (err) {
    if (err) {
      return console.error(err.message);
    }
    console.log(`Row inserted with ID: ${this.lastID}`);
  }
);

// With named parameters
db.run(
  "INSERT INTO users (name, email) VALUES ($name, $email)",
  {
    $name: "Jane Doe",
    $email: "jane@example.com",
  },
  function (err) {
    if (err) {
      return console.error(err.message);
    }
    console.log(`Row inserted with ID: ${this.lastID}`);
  }
);
```

### Getting a Single Row (sqlite3)

```javascript
// Get a single row
db.get("SELECT * FROM users WHERE id = ?", [1], (err, row) => {
  if (err) {
    return console.error(err.message);
  }
  console.log(row ? row.name : "No user found");
});
```

### Getting Multiple Rows (sqlite3)

```javascript
// Get all rows
db.all("SELECT * FROM users", [], (err, rows) => {
  if (err) {
    return console.error(err.message);
  }
  rows.forEach((row) => {
    console.log(row.name);
  });
});
```

### Iterating Through Rows (sqlite3)

```javascript
// Process rows one at a time
db.each(
  "SELECT * FROM users",
  [],
  (err, row) => {
    if (err) {
      return console.error(err.message);
    }
    console.log(`${row.id}: ${row.name}`);
  },
  (err, count) => {
    if (err) {
      return console.error(err.message);
    }
    console.log(`Total rows: ${count}`);
  }
);
```

### Query Execution (better-sqlite3)

```javascript
// Simple execution (no results)
db.exec("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT, email TEXT)");

// Insert with parameters
const insert = db.prepare("INSERT INTO users (name, email) VALUES (?, ?)");
const result = insert.run("John Doe", "john@example.com");
console.log(`Row inserted with ID: ${result.lastInsertRowid}`);

// Get a single row
const getUser = db.prepare("SELECT * FROM users WHERE id = ?");
const user = getUser.get(1);
console.log(user ? user.name : "No user found");

// Get all rows
const getAllUsers = db.prepare("SELECT * FROM users");
const users = getAllUsers.all();
users.forEach((user) => {
  console.log(user.name);
});

// Iterate through rows
const allUsers = db.prepare("SELECT * FROM users");
const iterator = allUsers.iterate();
for (const user of iterator) {
  console.log(`${user.id}: ${user.name}`);
}
```

## Prepared Statements

### sqlite3

```javascript
// Create a prepared statement
const stmt = db.prepare("INSERT INTO users (name, email) VALUES (?, ?)");

// Execute multiple times
stmt.run("User 1", "user1@example.com");
stmt.run("User 2", "user2@example.com");
stmt.run("User 3", "user3@example.com");

// Finalize when done
stmt.finalize();
```

### better-sqlite3

```javascript
// Create a prepared statement
const stmt = db.prepare("INSERT INTO users (name, email) VALUES (?, ?)");

// Execute multiple times
stmt.run("User 1", "user1@example.com");
stmt.run("User 2", "user2@example.com");
stmt.run("User 3", "user3@example.com");

// No need to finalize - automatically handled
```

## Transactions

### sqlite3

```javascript
// Begin transaction
db.serialize(() => {
  db.run("BEGIN TRANSACTION");

  try {
    db.run("INSERT INTO users (name, email) VALUES (?, ?)", [
      "User 1",
      "user1@example.com",
    ]);
    db.run("INSERT INTO users (name, email) VALUES (?, ?)", [
      "User 2",
      "user2@example.com",
    ]);
    db.run("COMMIT");
  } catch (err) {
    db.run("ROLLBACK");
    console.error("Transaction failed: ", err);
  }
});
```

### better-sqlite3

```javascript
// Simple transaction
const insertUser = db.prepare("INSERT INTO users (name, email) VALUES (?, ?)");

const transaction = db.transaction((users) => {
  for (const user of users) {
    insertUser.run(user.name, user.email);
  }
});

// Execute transaction
transaction([
  { name: "User 1", email: "user1@example.com" },
  { name: "User 2", email: "user2@example.com" },
]);
```

## Error Handling

### sqlite3

```javascript
// Using callbacks
db.run("INSERT INTO nonexistent_table VALUES (?)", [1], function (err) {
  if (err) {
    console.error("Error executing query: ", err.message);
    return;
  }
  console.log("Query executed successfully");
});

// Using try/catch with async/await
async function executeQuery() {
  return new Promise((resolve, reject) => {
    db.run("INSERT INTO users (name) VALUES (?)", ["John"], function (err) {
      if (err) reject(err);
      else resolve(this.lastID);
    });
  });
}

async function main() {
  try {
    const id = await executeQuery();
    console.log(`Inserted row with ID: ${id}`);
  } catch (err) {
    console.error("Failed to execute query: ", err.message);
  }
}
```

### better-sqlite3

```javascript
// Using try/catch
try {
  const stmt = db.prepare("INSERT INTO nonexistent_table VALUES (?)");
  stmt.run(1);
} catch (err) {
  console.error("Error executing query: ", err.message);
}
```

## Data Types and Conversions

SQLite supports the following data types:

- NULL
- INTEGER
- REAL
- TEXT
- BLOB

```javascript
// Storing different data types
const stmt = db.prepare(`
  INSERT INTO data_types (
    null_value, 
    integer_value, 
    real_value, 
    text_value, 
    blob_value
  ) VALUES (?, ?, ?, ?, ?)
`);

stmt.run(
  null, // NULL
  42, // INTEGER
  3.14, // REAL
  "Hello, world!", // TEXT
  Buffer.from("binary") // BLOB
);
```

## Configuration Options

### sqlite3

```javascript
// Enable verbose mode
const sqlite3 = require("sqlite3").verbose();

// Configure database
const db = new sqlite3.Database("./database.sqlite", {
  // Open the database for reading and writing
  mode: sqlite3.OPEN_READWRITE | sqlite3.OPEN_CREATE,
});

// Set a busy timeout
db.configure("busyTimeout", 3000);
```

### better-sqlite3

```javascript
// Configure database
const db = new Database("./database.sqlite", {
  readonly: false, // Open in read-write mode
  fileMustExist: false, // Create if doesn't exist
  timeout: 5000, // Busy timeout (ms)
  verbose: console.log, // Log all queries
});

// Set pragmas
db.pragma("journal_mode = WAL");
db.pragma("foreign_keys = ON");
```

## Common Patterns

### Promisifying sqlite3

```javascript
// Promisify the sqlite3 API
function runAsync(db, sql, params = []) {
  return new Promise((resolve, reject) => {
    db.run(sql, params, function (err) {
      if (err) reject(err);
      else resolve({ lastID: this.lastID, changes: this.changes });
    });
  });
}

function getAsync(db, sql, params = []) {
  return new Promise((resolve, reject) => {
    db.get(sql, params, (err, row) => {
      if (err) reject(err);
      else resolve(row);
    });
  });
}

function allAsync(db, sql, params = []) {
  return new Promise((resolve, reject) => {
    db.all(sql, params, (err, rows) => {
      if (err) reject(err);
      else resolve(rows);
    });
  });
}

// Usage
async function main() {
  try {
    await runAsync(
      db,
      "CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, name TEXT)"
    );
    const result = await runAsync(db, "INSERT INTO users (name) VALUES (?)", [
      "John",
    ]);
    console.log(`Inserted ID: ${result.lastID}`);

    const user = await getAsync(db, "SELECT * FROM users WHERE id = ?", [
      result.lastID,
    ]);
    console.log("User:", user);

    const allUsers = await allAsync(db, "SELECT * FROM users");
    console.log("All users:", allUsers);
  } catch (err) {
    console.error("Database error:", err);
  }
}
```

### Database Wrapper Class

```javascript
// A simple wrapper class for sqlite3
class Database {
  constructor(dbPath) {
    const sqlite3 = require("sqlite3").verbose();
    this.db = new sqlite3.Database(dbPath);
  }

  run(sql, params = []) {
    return new Promise((resolve, reject) => {
      this.db.run(sql, params, function (err) {
        if (err) reject(err);
        else resolve({ lastID: this.lastID, changes: this.changes });
      });
    });
  }

  get(sql, params = []) {
    return new Promise((resolve, reject) => {
      this.db.get(sql, params, (err, row) => {
        if (err) reject(err);
        else resolve(row);
      });
    });
  }

  all(sql, params = []) {
    return new Promise((resolve, reject) => {
      this.db.all(sql, params, (err, rows) => {
        if (err) reject(err);
        else resolve(rows);
      });
    });
  }

  close() {
    return new Promise((resolve, reject) => {
      this.db.close((err) => {
        if (err) reject(err);
        else resolve();
      });
    });
  }
}

// Usage
async function main() {
  const db = new Database("./database.sqlite");

  try {
    await db.run(
      "CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, name TEXT)"
    );
    const result = await db.run("INSERT INTO users (name) VALUES (?)", [
      "John",
    ]);
    console.log(`Inserted ID: ${result.lastID}`);

    const users = await db.all("SELECT * FROM users");
    console.log("Users:", users);
  } catch (err) {
    console.error("Error:", err);
  } finally {
    await db.close();
  }
}
```

### Migration Pattern

```javascript
// Simple migration system
async function runMigrations(db) {
  // Create migrations table if it doesn't exist
  await db.run(`
    CREATE TABLE IF NOT EXISTS migrations (
      id INTEGER PRIMARY KEY,
      name TEXT UNIQUE,
      applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
  `);

  // Define migrations
  const migrations = [
    {
      name: "001-initial-schema",
      sql: `
        CREATE TABLE IF NOT EXISTS users (
          id INTEGER PRIMARY KEY,
          name TEXT NOT NULL,
          email TEXT UNIQUE,
          created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        );
        
        CREATE TABLE IF NOT EXISTS tasks (
          id INTEGER PRIMARY KEY,
          user_id INTEGER,
          title TEXT NOT NULL,
          completed BOOLEAN DEFAULT 0,
          FOREIGN KEY (user_id) REFERENCES users (id)
        );
      `,
    },
    {
      name: "002-add-user-status",
      sql: `
        ALTER TABLE users ADD COLUMN status TEXT DEFAULT 'active';
      `,
    },
  ];

  // Run each migration if not already applied
  for (const migration of migrations) {
    const existing = await db.get("SELECT id FROM migrations WHERE name = ?", [
      migration.name,
    ]);

    if (!existing) {
      console.log(`Applying migration: ${migration.name}`);

      // Start transaction
      await db.run("BEGIN TRANSACTION");

      try {
        // Run the migration SQL
        await db.run(migration.sql);

        // Record the migration
        await db.run("INSERT INTO migrations (name) VALUES (?)", [
          migration.name,
        ]);

        // Commit the transaction
        await db.run("COMMIT");
        console.log(`Migration applied: ${migration.name}`);
      } catch (err) {
        // Rollback on error
        await db.run("ROLLBACK");
        console.error(`Migration failed: ${migration.name}`, err);
        throw err;
      }
    } else {
      console.log(`Skipping migration (already applied): ${migration.name}`);
    }
  }
}
```

## Performance Tips

### Optimizing Queries

```javascript
// Use prepared statements for repeated queries
const stmt = db.prepare("SELECT * FROM users WHERE name = ?");

// Use indexes for frequently queried columns
db.exec("CREATE INDEX idx_users_name ON users(name)");

// Use transactions for multiple operations
db.exec("BEGIN TRANSACTION");
// ... multiple operations
db.exec("COMMIT");

// Use EXPLAIN to analyze query performance
db.all(
  "EXPLAIN QUERY PLAN SELECT * FROM users WHERE name = ?",
  ["John"],
  (err, plan) => {
    console.log("Query plan:", plan);
  }
);
```

### Bulk Operations

```javascript
// Better performance for bulk inserts (sqlite3)
db.serialize(() => {
  db.run("BEGIN TRANSACTION");

  const stmt = db.prepare("INSERT INTO users (name, email) VALUES (?, ?)");

  for (let i = 0; i < 1000; i++) {
    stmt.run(`User ${i}`, `user${i}@example.com`);
  }

  stmt.finalize();
  db.run("COMMIT");
});

// Bulk operations with better-sqlite3
const insert = db.prepare("INSERT INTO users (name, email) VALUES (?, ?)");
const insertMany = db.transaction((users) => {
  for (const user of users) {
    insert.run(user.name, user.email);
  }
});

// Create 1000 users
const users = Array.from({ length: 1000 }, (_, i) => ({
  name: `User ${i}`,
  email: `user${i}@example.com`,
}));

// Insert all in one transaction
insertMany(users);
```

### Memory Usage

```javascript
// Control memory usage with each() instead of all() for large result sets
db.each("SELECT * FROM large_table", (err, row) => {
  // Process one row at a time
  processRow(row);
});

// Use streaming with better-sqlite3
const stmt = db.prepare("SELECT * FROM large_table");
const iterator = stmt.iterate();

for (const row of iterator) {
  // Process one row at a time
  processRow(row);
}
```

## SQLite-Specific Features

### PRAGMA Statements

```javascript
// Check foreign key constraints
db.get("PRAGMA foreign_keys", (err, result) => {
  console.log("Foreign keys enabled:", result.foreign_keys === 1);
});

// Enable foreign key constraints
db.run("PRAGMA foreign_keys = ON");

// Set journal mode
db.run("PRAGMA journal_mode = WAL");

// Get database information
db.all("PRAGMA database_list", (err, databases) => {
  console.log("Attached databases:", databases);
});

// Get table information
db.all("PRAGMA table_info(users)", (err, columns) => {
  console.log("User table columns:", columns);
});

// Get index information
db.all("PRAGMA index_list(users)", (err, indexes) => {
  console.log("User table indexes:", indexes);
});
```

### JSON Functions

```javascript
// Create a table with JSON data
db.run(`
  CREATE TABLE IF NOT EXISTS settings (
    id INTEGER PRIMARY KEY,
    user_id INTEGER,
    preferences TEXT,  -- Stored as JSON
    FOREIGN KEY (user_id) REFERENCES users (id)
  )
`);

// Insert JSON data
const preferences = {
  theme: "dark",
  notifications: true,
  language: "en",
};

db.run("INSERT INTO settings (user_id, preferences) VALUES (?, ?)", [
  1,
  JSON.stringify(preferences),
]);

// Query with JSON functions (SQLite 3.38.0+)
db.all(
  `
  SELECT 
    user_id,
    json_extract(preferences, '$.theme') AS theme,
    json_extract(preferences, '$.notifications') AS notifications
  FROM settings
  WHERE json_extract(preferences, '$.theme') = 'dark'
`,
  (err, rows) => {
    console.log("Users with dark theme:", rows);
  }
);

// Alternative for older SQLite versions
db.all(
  `
  SELECT user_id, preferences
  FROM settings
`,
  (err, rows) => {
    if (err) return console.error(err);

    // Filter in JavaScript
    const darkThemeUsers = rows.filter((row) => {
      const prefs = JSON.parse(row.preferences);
      return prefs.theme === "dark";
    });

    console.log("Users with dark theme:", darkThemeUsers);
  }
);
```

### Full-Text Search

```javascript
// Enable FTS5 extension
db.run(`
  CREATE VIRTUAL TABLE IF NOT EXISTS posts_fts USING fts5(
    title,
    content,
    tokenize='porter'
  )
`);

// Insert data
db.run(`
  INSERT INTO posts_fts (title, content) VALUES
  ('SQLite Tutorial', 'Learn how to use SQLite with Node.js'),
  ('Node.js Basics', 'Introduction to Node.js development'),
  ('Database Design', 'Best practices for database schema design')
`);

// Search
db.all(
  `
  SELECT * FROM posts_fts
  WHERE posts_fts MATCH ?
  ORDER BY rank
`,
  ["sqlite OR database"],
  (err, results) => {
    if (err) return console.error(err);
    console.log("Search results:", results);
  }
);
```

## Working with Dates

```javascript
// SQLite doesn't have a dedicated date type, but offers date functions

// Store dates as ISO strings
const now = new Date().toISOString();
db.run("INSERT INTO events (name, event_date) VALUES (?, ?)", ["Meeting", now]);

// Store as Unix timestamp (seconds since epoch)
const timestamp = Math.floor(Date.now() / 1000);
db.run("INSERT INTO events (name, event_timestamp) VALUES (?, ?)", [
  "Conference",
  timestamp,
]);

// Query with date functions
db.all(
  `
  SELECT 
    name,
    event_date,
    datetime(event_date) AS formatted_date,
    strftime('%Y-%m-%d', event_date) AS just_date,
    strftime('%H:%M', event_date) AS just_time
  FROM events
  WHERE date(event_date) = date('now')
`,
  (err, events) => {
    console.log("Today's events:", events);
  }
);

// Date calculations
db.all(
  `
  SELECT 
    name,
    event_date,
    datetime(event_date, '+1 day') AS next_day,
    datetime(event_date, '+1 month') AS next_month,
    datetime(event_date, '+1 year') AS next_year
  FROM events
`,
  (err, events) => {
    console.log("Events with calculated dates:", events);
  }
);
```

## Backup and Recovery

```javascript
// Backup database (sqlite3)
const fs = require("fs");
const path = require("path");

function backupDatabase(sourceDB, backupPath) {
  return new Promise((resolve, reject) => {
    sourceDB.serialize(() => {
      // Create backup directory if it doesn't exist
      const dir = path.dirname(backupPath);
      if (!fs.existsSync(dir)) {
        fs.mkdirSync(dir, { recursive: true });
      }

      // Backup using VACUUM INTO (SQLite 3.27.0+)
      sourceDB.run(`VACUUM INTO '${backupPath}'`, (err) => {
        if (err) {
          reject(err);
        } else {
          resolve(backupPath);
        }
      });
    });
  });
}

// Usage
backupDatabase(db, "./backups/database-backup.sqlite")
  .then((path) => console.log(`Database backed up to ${path}`))
  .catch((err) => console.error("Backup failed:", err));

// Alternative backup method for older SQLite versions
function backupDatabaseLegacy(sourceDB, backupPath) {
  return new Promise((resolve, reject) => {
    const backupDB = new sqlite3.Database(backupPath);

    sourceDB.serialize(() => {
      // Begin transaction
      backupDB.run("BEGIN TRANSACTION");

      // Get all tables
      sourceDB.all(
        "SELECT name FROM sqlite_master WHERE type='table'",
        (err, tables) => {
          if (err) {
            backupDB.run("ROLLBACK");
            backupDB.close();
            return reject(err);
          }

          // Process each table
          let processed = 0;
          tables.forEach((table) => {
            // Skip sqlite_ internal tables
            if (table.name.startsWith("sqlite_")) {
              processed++;
              if (processed === tables.length) {
                backupDB.run("COMMIT");
                backupDB.close();
                resolve(backupPath);
              }
              return;
            }

            // Get table schema
            sourceDB.all(
              `SELECT sql FROM sqlite_master WHERE name='${table.name}'`,
              (err, schema) => {
                if (err) {
                  backupDB.run("ROLLBACK");
                  backupDB.close();
                  return reject(err);
                }

                // Create table in backup
                backupDB.run(schema[0].sql, (err) => {
                  if (err) {
                    backupDB.run("ROLLBACK");
                    backupDB.close();
                    return reject(err);
                  }

                  // Copy data
                  sourceDB.all(`SELECT * FROM ${table.name}`, (err, rows) => {
                    if (err) {
                      backupDB.run("ROLLBACK");
                      backupDB.close();
                      return reject(err);
                    }

                    if (rows.length > 0) {
                      // Generate INSERT statement
                      const columns = Object.keys(rows[0]).join(", ");
                      const placeholders = Object.keys(rows[0])
                        .map(() => "?")
                        .join(", ");
                      const stmt = backupDB.prepare(
                        `INSERT INTO ${table.name} (${columns}) VALUES (${placeholders})`
                      );

                      rows.forEach((row) => {
                        stmt.run(Object.values(row));
                      });

                      stmt.finalize();
                    }

                    processed++;
                    if (processed === tables.length) {
                      backupDB.run("COMMIT");
                      backupDB.close();
                      resolve(backupPath);
                    }
                  });
                });
              }
            );
          });
        }
      );
    });
  });
}
```

## Security Considerations

```javascript
// Prevent SQL injection by using parameterized queries
// UNSAFE:
const username = "user' OR 1=1--";
db.all(`SELECT * FROM users WHERE username = '${username}'`); // SQL Injection vulnerability!

// SAFE:
db.all("SELECT * FROM users WHERE username = ?", [username]); // Safe parameterized query

// Limit database permissions
const readOnlyDB = new sqlite3.Database(
  "./database.sqlite",
  sqlite3.OPEN_READONLY
);

// Encrypt database with SQLCipher (requires SQLCipher extension)
const sqlcipher = require("sqlcipher");
const encryptedDB = new sqlcipher.Database("./encrypted.db");
encryptedDB.run('PRAGMA key = "your_secret_key"');

// Validate input before storing
function validateUser(user) {
  if (!user.name || typeof user.name !== "string" || user.name.length > 100) {
    throw new Error("Invalid user name");
  }
  if (user.email && (!user.email.includes("@") || user.email.length > 255)) {
    throw new Error("Invalid email");
  }
  return user;
}

try {
  const user = validateUser({ name: "John", email: "john@example.com" });
  db.run("INSERT INTO users (name, email) VALUES (?, ?)", [
    user.name,
    user.email,
  ]);
} catch (err) {
  console.error("Validation error:", err.message);
}
```
