# SQLite with Node.js Basics

This guide covers the fundamental concepts of using SQLite with Node.js, from installation to basic database operations.

## Table of Contents

- [Introduction to SQLite](#introduction-to-sqlite)
- [Setting Up SQLite with Node.js](#setting-up-sqlite-with-nodejs)
- [Creating and Connecting to a Database](#creating-and-connecting-to-a-database)
- [Creating Tables](#creating-tables)
- [Basic CRUD Operations](#basic-crud-operations)
- [Working with Query Results](#working-with-query-results)
- [Error Handling](#error-handling)
- [Basic Transactions](#basic-transactions)
- [Simple Migrations](#simple-migrations)
- [Best Practices](#best-practices)

## Introduction to SQLite

### What is SQLite?

SQLite is a self-contained, serverless, zero-configuration, transactional SQL database engine. Unlike most other SQL databases, SQLite doesn't have a separate server process. SQLite reads and writes directly to ordinary disk files.

### Key Features of SQLite

- **Serverless**: No separate server process required
- **Zero Configuration**: No setup or administration needed
- **Self-Contained**: A single file contains the entire database
- **Cross-Platform**: Works on virtually any platform
- **Reliable**: ACID-compliant, even after system crashes
- **Lightweight**: Small footprint, minimal dependencies
- **Public Domain**: Free for any use

### When to Use SQLite

SQLite is ideal for:

- Embedded applications
- Local/client storage
- Small to medium websites
- Prototyping and development
- Testing environments
- Educational purposes
- Mobile applications

### When Not to Use SQLite

SQLite may not be suitable for:

- High-volume websites with many concurrent writes
- Very large datasets (multi-terabyte)
- High-concurrency applications
- Applications requiring fine-grained access control

## Setting Up SQLite with Node.js

### Installing the sqlite3 Module

The most popular Node.js module for SQLite is `sqlite3`. Install it using npm:

```javascript
npm install sqlite3
```

For TypeScript users, also install the type definitions:

```javascript
npm install @types/sqlite3
```

### Alternative: better-sqlite3

An alternative module with better performance for synchronous operations is `better-sqlite3`:

```javascript
npm install better-sqlite3
```

### Basic Project Setup

A typical project structure for a Node.js application with SQLite might look like:

```
my-sqlite-app/
├── node_modules/
├── src/
│   ├── db/
│   │   ├── database.js    # Database connection
│   │   ├── migrations.js  # Database migrations
│   │   └── models/        # Data models
│   ├── routes/            # API routes
│   └── index.js           # Application entry point
├── package.json
└── database.sqlite        # SQLite database file
```

## Creating and Connecting to a Database

### Importing the Module

```javascript
// Using sqlite3
const sqlite3 = require("sqlite3").verbose();

// Using better-sqlite3
const Database = require("better-sqlite3");
```

### Creating a Database Connection

```javascript
// sqlite3
const db = new sqlite3.Database("./database.sqlite", (err) => {
  if (err) {
    console.error("Error opening database", err.message);
  } else {
    console.log("Connected to the SQLite database.");
  }
});
```

```javascript
// better-sqlite3
try {
  const db = new Database("./database.sqlite");
  console.log("Connected to the SQLite database.");
} catch (err) {
  console.error("Error opening database", err.message);
}
```

### Database Connection Modes

```javascript
// Read-only mode
const readOnlyDB = new sqlite3.Database(
  "./database.sqlite",
  sqlite3.OPEN_READONLY
);

// Read-write mode (default)
const readWriteDB = new sqlite3.Database(
  "./database.sqlite",
  sqlite3.OPEN_READWRITE
);

// Create if not exists and open read-write
const createDB = new sqlite3.Database(
  "./database.sqlite",
  sqlite3.OPEN_READWRITE | sqlite3.OPEN_CREATE
);
```

### In-Memory Database

For temporary databases that don't need to persist to disk:

```javascript
// sqlite3
const memDB = new sqlite3.Database(":memory:");

// better-sqlite3
const memDB = new Database(":memory:");
```

### Closing the Connection

Always close the database connection when you're done:

```javascript
// sqlite3
db.close((err) => {
  if (err) {
    console.error("Error closing database", err.message);
  } else {
    console.log("Database connection closed.");
  }
});
```

```javascript
// better-sqlite3
db.close();
console.log("Database connection closed.");
```

## Creating Tables

### Basic Table Creation

```javascript
// sqlite3
db.run(
  `
  CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT UNIQUE,
    age INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
  )
`,
  (err) => {
    if (err) {
      console.error("Error creating table", err.message);
    } else {
      console.log("Users table created or already exists.");
    }
  }
);
```

```javascript
// better-sqlite3
try {
  db.exec(`
    CREATE TABLE IF NOT EXISTS users (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT NOT NULL,
      email TEXT UNIQUE,
      age INTEGER,
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
  `);
  console.log("Users table created or already exists.");
} catch (err) {
  console.error("Error creating table", err.message);
}
```

### Common Column Types

SQLite supports these basic data types:

- `INTEGER`: Whole numbers
- `REAL`: Floating-point numbers
- `TEXT`: Text strings
- `BLOB`: Binary data
- `NULL`: Null values

### Column Constraints

Common constraints include:

- `PRIMARY KEY`: Uniquely identifies each row
- `NOT NULL`: Column cannot contain NULL values
- `UNIQUE`: All values in the column must be different
- `DEFAULT`: Specifies a default value
- `CHECK`: Ensures values meet a condition
- `AUTOINCREMENT`: Automatically increments the value

### Creating Tables with Foreign Keys

```javascript
db.run(`
  CREATE TABLE IF NOT EXISTS posts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER,
    title TEXT NOT NULL,
    content TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users (id)
  )
`);
```

To enable foreign key constraints:

```javascript
// sqlite3
db.run("PRAGMA foreign_keys = ON");

// better-sqlite3
db.pragma("foreign_keys = ON");
```

## Basic CRUD Operations

### Create (INSERT)

```javascript
// Basic insert
db.run(
  "INSERT INTO users (name, email, age) VALUES (?, ?, ?)",
  ["John Doe", "john@example.com", 30],
  function (err) {
    if (err) {
      return console.error("Error inserting data", err.message);
    }
    console.log(`A row has been inserted with rowid ${this.lastID}`);
  }
);

// Insert with named parameters
db.run(
  "INSERT INTO users (name, email, age) VALUES ($name, $email, $age)",
  {
    $name: "Jane Doe",
    $email: "jane@example.com",
    $age: 25,
  },
  function (err) {
    if (err) {
      return console.error("Error inserting data", err.message);
    }
    console.log(`A row has been inserted with rowid ${this.lastID}`);
  }
);
```

### Read (SELECT)

#### Selecting a Single Row

```javascript
// Get a single user by ID
db.get("SELECT * FROM users WHERE id = ?", [1], (err, row) => {
  if (err) {
    return console.error("Error querying data", err.message);
  }

  if (row) {
    console.log("User found:", row);
  } else {
    console.log("No user found with that ID");
  }
});
```

#### Selecting Multiple Rows

```javascript
// Get all users
db.all("SELECT * FROM users", [], (err, rows) => {
  if (err) {
    return console.error("Error querying data", err.message);
  }

  console.log(`Found ${rows.length} users:`);
  rows.forEach((row) => {
    console.log(`${row.id}: ${row.name} (${row.email})`);
  });
});

// Get users with filtering
db.all("SELECT * FROM users WHERE age > ?", [25], (err, rows) => {
  if (err) {
    return console.error("Error querying data", err.message);
  }

  console.log(`Found ${rows.length} users over 25:`);
  rows.forEach((row) => {
    console.log(`${row.id}: ${row.name}, Age: ${row.age}`);
  });
});
```

#### Using ORDER BY, LIMIT, and OFFSET

```javascript
// Get users sorted by age, limited to 5 results
db.all("SELECT * FROM users ORDER BY age DESC LIMIT 5", [], (err, rows) => {
  if (err) {
    return console.error("Error querying data", err.message);
  }

  console.log("Top 5 oldest users:");
  rows.forEach((row) => {
    console.log(`${row.name}: ${row.age}`);
  });
});

// Pagination example
const page = 1;
const pageSize = 10;
const offset = (page - 1) * pageSize;

db.all(
  "SELECT * FROM users LIMIT ? OFFSET ?",
  [pageSize, offset],
  (err, rows) => {
    if (err) {
      return console.error("Error querying data", err.message);
    }

    console.log(`Users (Page ${page}):`);
    rows.forEach((row) => {
      console.log(`${row.id}: ${row.name}`);
    });
  }
);
```

### Update (UPDATE)

```javascript
// Update a single user
db.run(
  "UPDATE users SET name = ?, age = ? WHERE id = ?",
  ["John Smith", 31, 1],
  function (err) {
    if (err) {
      return console.error("Error updating data", err.message);
    }

    console.log(`User updated: ${this.changes} row(s) modified`);
  }
);

// Update multiple users
db.run(
  "UPDATE users SET status = ? WHERE age > ?",
  ["senior", 50],
  function (err) {
    if (err) {
      return console.error("Error updating data", err.message);
    }

    console.log(`Users updated: ${this.changes} row(s) modified`);
  }
);
```

### Delete (DELETE)

```javascript
// Delete a single user
db.run("DELETE FROM users WHERE id = ?", [1], function (err) {
  if (err) {
    return console.error("Error deleting data", err.message);
  }

  console.log(`User deleted: ${this.changes} row(s) removed`);
});

// Delete multiple users
db.run("DELETE FROM users WHERE age < ?", [18], function (err) {
  if (err) {
    return console.error("Error deleting data", err.message);
  }

  console.log(`Users deleted: ${this.changes} row(s) removed`);
});
```

## Working with Query Results

### Handling Single Row Results

```javascript
db.get("SELECT * FROM users WHERE id = ?", [1], (err, row) => {
  if (err) {
    return console.error(err.message);
  }

  // Check if a row was found
  if (row) {
    // Access properties directly
    console.log(`Name: ${row.name}, Email: ${row.email}`);

    // Convert row to JSON
    const userJson = JSON.stringify(row);
    console.log("User as JSON:", userJson);

    // Destructure properties
    const { id, name, email } = row;
    console.log(`ID: ${id}, Name: ${name}, Email: ${email}`);
  } else {
    console.log("No user found");
  }
});
```

### Processing Multiple Rows

```javascript
db.all("SELECT * FROM users", [], (err, rows) => {
  if (err) {
    return console.error(err.message);
  }

  // Check if any rows were found
  if (rows.length === 0) {
    return console.log("No users found");
  }

  // Iterate with forEach
  rows.forEach((row) => {
    console.log(`${row.id}: ${row.name}`);
  });

  // Map to a new array
  const names = rows.map((row) => row.name);
  console.log("User names:", names);

  // Filter rows
  const olderUsers = rows.filter((row) => row.age > 30);
  console.log("Users over 30:", olderUsers.length);

  // Find a specific row
  const jane = rows.find((row) => row.email === "jane@example.com");
  if (jane) {
    console.log("Found Jane:", jane);
  }

  // Reduce to calculate average age
  const totalAge = rows.reduce((sum, row) => sum + row.age, 0);
  const averageAge = totalAge / rows.length;
  console.log(`Average age: ${averageAge.toFixed(1)}`);
});
```

### Processing Rows One at a Time

```javascript
db.each(
  "SELECT * FROM users",
  [],
  (err, row) => {
    if (err) {
      return console.error(err.message);
    }

    // Process each row as it comes in
    console.log(`Processing user: ${row.name}`);

    // Do something with the row
    processUser(row);
  },
  (err, count) => {
    // This callback runs after all rows are processed
    if (err) {
      return console.error(err.message);
    }

    console.log(`Processed ${count} users`);
  }
);

function processUser(user) {
  // Example processing function
  console.log(`User ${user.id} processed`);
}
```

## Error Handling

### Basic Error Handling

```javascript
db.run("INSERT INTO users (name) VALUES (?)", ["John"], function (err) {
  if (err) {
    console.error("Error code:", err.code);
    console.error("Error message:", err.message);

    // Handle specific error types
    if (err.code === "SQLITE_CONSTRAINT") {
      console.error("Constraint violation (e.g., unique constraint)");
    } else if (err.code === "SQLITE_ERROR") {
      console.error("SQL syntax error");
    }

    return;
  }

  console.log("Insert successful");
});
```

### Using Try-Catch with Promises

```javascript
// Promisify the run method
function runAsync(sql, params = []) {
  return new Promise((resolve, reject) => {
    db.run(sql, params, function (err) {
      if (err) {
        reject(err);
      } else {
        resolve({ lastID: this.lastID, changes: this.changes });
      }
    });
  });
}

// Using the promisified function with async/await
async function createUser(name, email) {
  try {
    const result = await runAsync(
      "INSERT INTO users (name, email) VALUES (?, ?)",
      [name, email]
    );

    console.log(`User created with ID: ${result.lastID}`);
    return result.lastID;
  } catch (err) {
    console.error("Failed to create user:", err.message);

    // Re-throw or handle specific errors
    if (err.code === "SQLITE_CONSTRAINT") {
      throw new Error("A user with that email already exists");
    }

    throw err;
  }
}

// Usage
createUser("Alice", "alice@example.com")
  .then((userId) => {
    console.log(`Success! User ID: ${userId}`);
  })
  .catch((err) => {
    console.error("Error:", err.message);
  });
```

### Common SQLite Error Codes

- `SQLITE_CONSTRAINT`: Constraint violation (e.g., UNIQUE, NOT NULL)
- `SQLITE_ERROR`: SQL syntax error
- `SQLITE_BUSY`: Database is locked
- `SQLITE_NOTFOUND`: Table or database not found
- `SQLITE_FULL`: Database or disk is full
- `SQLITE_CANTOPEN`: Unable to open database file
- `SQLITE_PERM`: Access permission denied
- `SQLITE_READONLY`: Database is read-only

## Basic Transactions

Transactions allow you to group multiple operations together so they either all succeed or all fail.

### Why Use Transactions?

- **Data Integrity**: Ensures related changes are applied together
- **Consistency**: Database remains in a consistent state
- **Rollback**: Ability to undo changes if something goes wrong
- **Performance**: Can improve performance for multiple operations

### Basic Transaction Example

```javascript
// sqlite3
db.serialize(() => {
  db.run("BEGIN TRANSACTION");

  db.run(
    "INSERT INTO users (name, email) VALUES (?, ?)",
    ["User 1", "user1@example.com"],
    function (err) {
      if (err) {
        console.error("Error in first insert:", err.message);
        return db.run("ROLLBACK");
      }

      db.run(
        "INSERT INTO users (name, email) VALUES (?, ?)",
        ["User 2", "user2@example.com"],
        function (err) {
          if (err) {
            console.error("Error in second insert:", err.message);
            return db.run("ROLLBACK");
          }

          db.run("COMMIT", (err) => {
            if (err) {
              console.error("Error committing transaction:", err.message);
              return db.run("ROLLBACK");
            }
            console.log("Transaction completed successfully");
          });
        }
      );
    }
  );
});
```

### Transactions with Async/Await

```javascript
// Promisified transaction helper
async function runTransaction(operations) {
  return new Promise((resolve, reject) => {
    db.serialize(() => {
      db.run("BEGIN TRANSACTION");

      let results = [];
      let currentOp = 0;

      function runNext() {
        if (currentOp >= operations.length) {
          // All operations completed successfully
          db.run("COMMIT", (err) => {
            if (err) {
              db.run("ROLLBACK");
              return reject(err);
            }
            resolve(results);
          });
          return;
        }

        const { sql, params } = operations[currentOp];

        db.run(sql, params, function (err) {
          if (err) {
            db.run("ROLLBACK");
            return reject(err);
          }

          results.push({
            lastID: this.lastID,
            changes: this.changes,
          });

          currentOp++;
          runNext();
        });
      }

      runNext();
    });
  });
}

// Usage
async function createUserWithPosts(userData, posts) {
  try {
    // Prepare operations for the transaction
    const operations = [
      {
        sql: "INSERT INTO users (name, email) VALUES (?, ?)",
        params: [userData.name, userData.email],
      },
    ];

    // Add post operations
    posts.forEach((post) => {
      operations.push({
        sql: "INSERT INTO posts (user_id, title, content) VALUES (last_insert_rowid(), ?, ?)",
        params: [post.title, post.content],
      });
    });

    // Run the transaction
    const results = await runTransaction(operations);

    console.log(
      `Created user with ID ${results[0].lastID} and ${posts.length} posts`
    );
    return results[0].lastID;
  } catch (err) {
    console.error("Transaction failed:", err.message);
    throw err;
  }
}
```

## Simple Migrations

Migrations help you manage database schema changes over time.

### Basic Migration System

```javascript
// Define your migrations
const migrations = [
  {
    name: "001-initial-schema",
    up: `
      CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        email TEXT UNIQUE,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      );
      
      CREATE TABLE IF NOT EXISTS posts (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        title TEXT NOT NULL,
        content TEXT,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        FOREIGN KEY (user_id) REFERENCES users (id)
      );
    `,
  },
  {
    name: "002-add-user-status",
    up: `
      ALTER TABLE users ADD COLUMN status TEXT DEFAULT 'active';
    `,
  },
];

// Function to run migrations
function runMigrations() {
  // Create migrations table if it doesn't exist
  db.run(
    `
    CREATE TABLE IF NOT EXISTS migrations (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT UNIQUE,
      applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
  `,
    (err) => {
      if (err) {
        return console.error("Error creating migrations table:", err.message);
      }

      // Get applied migrations
      db.all("SELECT name FROM migrations", [], (err, appliedMigrations) => {
        if (err) {
          return console.error(
            "Error getting applied migrations:",
            err.message
          );
        }

        // Convert to a set for easy lookup
        const appliedMigrationNames = new Set(
          appliedMigrations.map((m) => m.name)
        );

        // Filter out already applied migrations
        const pendingMigrations = migrations.filter(
          (m) => !appliedMigrationNames.has(m.name)
        );

        if (pendingMigrations.length === 0) {
          return console.log("No pending migrations");
        }

        console.log(`Running ${pendingMigrations.length} migrations...`);

        // Run each migration in sequence
        db.serialize(() => {
          pendingMigrations.forEach((migration) => {
            console.log(`Applying migration: ${migration.name}`);

            // Begin transaction for this migration
            db.run("BEGIN TRANSACTION");

            // Run the migration
            db.run(migration.up, (err) => {
              if (err) {
                console.error(
                  `Error applying migration ${migration.name}:`,
                  err.message
                );
                return db.run("ROLLBACK");
              }

              // Record the migration
              db.run(
                "INSERT INTO migrations (name) VALUES (?)",
                [migration.name],
                (err) => {
                  if (err) {
                    console.error(
                      `Error recording migration ${migration.name}:`,
                      err.message
                    );
                    return db.run("ROLLBACK");
                  }

                  // Commit the transaction
                  db.run("COMMIT", (err) => {
                    if (err) {
                      console.error(
                        `Error committing migration ${migration.name}:`,
                        err.message
                      );
                      return db.run("ROLLBACK");
                    }

                    console.log(`Migration applied: ${migration.name}`);
                  });
                }
              );
            });
          });
        });
      });
    }
  );
}

// Run migrations
runMigrations();
```

### Async Migration System

```javascript
// Promisified migration system
async function runMigrationsAsync() {
  try {
    // Create migrations table if it doesn't exist
    await runAsync(`
      CREATE TABLE IF NOT EXISTS migrations (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT UNIQUE,
        applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      )
    `);

    // Get applied migrations
    const appliedMigrations = await allAsync("SELECT name FROM migrations");
    const appliedMigrationNames = new Set(appliedMigrations.map((m) => m.name));

    // Filter out already applied migrations
    const pendingMigrations = migrations.filter(
      (m) => !appliedMigrationNames.has(m.name)
    );

    if (pendingMigrations.length === 0) {
      console.log("No pending migrations");
      return;
    }

    console.log(`Running ${pendingMigrations.length} migrations...`);

    // Run each migration in sequence
    for (const migration of pendingMigrations) {
      console.log(`Applying migration: ${migration.name}`);

      try {
        // Begin transaction
        await runAsync("BEGIN TRANSACTION");

        // Run the migration
        await runAsync(migration.up);

        // Record the migration
        await runAsync("INSERT INTO migrations (name) VALUES (?)", [
          migration.name,
        ]);

        // Commit the transaction
        await runAsync("COMMIT");

        console.log(`Migration applied: ${migration.name}`);
      } catch (err) {
        // Rollback on error
        await runAsync("ROLLBACK");
        console.error(
          `Error applying migration ${migration.name}:`,
          err.message
        );
        throw err;
      }
    }

    console.log("All migrations applied successfully");
  } catch (err) {
    console.error("Migration process failed:", err.message);
    throw err;
  }
}

// Helper functions
function runAsync(sql, params = []) {
  return new Promise((resolve, reject) => {
    db.run(sql, params, function (err) {
      if (err) reject(err);
      else resolve({ lastID: this.lastID, changes: this.changes });
    });
  });
}

function allAsync(sql, params = []) {
  return new Promise((resolve, reject) => {
    db.all(sql, params, (err, rows) => {
      if (err) reject(err);
      else resolve(rows);
    });
  });
}
```

## Best Practices

### Database Connection Management

```javascript
// Create a database module (database.js)
const sqlite3 = require("sqlite3").verbose();
const path = require("path");

// Database file path
const dbPath = path.resolve(__dirname, "../database.sqlite");

// Create a singleton connection
let db = null;

function getDatabase() {
  if (db === null) {
    db = new sqlite3.Database(dbPath, (err) => {
      if (err) {
        console.error("Could not connect to database", err);
      } else {
        console.log("Connected to database");
      }
    });

    // Enable foreign keys
    db.run("PRAGMA foreign_keys = ON");

    // Configure for better performance
    db.run("PRAGMA journal_mode = WAL");
  }

  return db;
}

function closeDatabase() {
  if (db) {
    db.close((err) => {
      if (err) {
        console.error("Error closing database", err);
      } else {
        console.log("Database connection closed");
        db = null;
      }
    });
  }
}

// Handle application shutdown
process.on("SIGINT", () => {
  closeDatabase();
  process.exit(0);
});

module.exports = {
  getDatabase,
  closeDatabase,
};
```

### Query Parameters

Always use parameterized queries to prevent SQL injection:

```javascript
// UNSAFE - vulnerable to SQL injection
const name = "O'Reilly"; // This could break your query or be exploited
db.run(`INSERT INTO users (name) VALUES ('${name}')`); // BAD!

// SAFE - using parameters
db.run("INSERT INTO users (name) VALUES (?)", [name]); // GOOD!

// SAFE - using named parameters
db.run("INSERT INTO users (name) VALUES ($name)", { $name: name }); // GOOD!
```

### Error Handling Strategy

```javascript
// Consistent error handling
function handleDatabaseError(operation, err) {
  console.error(`Database error during ${operation}:`, err.message);

  // Log additional information for debugging
  console.error("Error code:", err.code);
  console.error("Stack trace:", err.stack);

  // You might want to notify monitoring systems here

  // Return a user-friendly error
  return {
    error: true,
    message: `An error occurred during ${operation}. Please try again later.`,
    code: err.code,
  };
}

// Usage
db.get("SELECT * FROM users WHERE id = ?", [userId], (err, row) => {
  if (err) {
    const error = handleDatabaseError("user lookup", err);
    return res.status(500).json(error);
  }

  // Continue with normal operation
  res.json(row);
});
```

### Performance Optimization

```javascript
// Use prepared statements for repeated queries
const stmt = db.prepare("INSERT INTO logs (message, level) VALUES (?, ?)");

for (let i = 0; i < 100; i++) {
  stmt.run(`Log message ${i}`, "info");
}

stmt.finalize();

// Use transactions for bulk operations
db.serialize(() => {
  db.run("BEGIN TRANSACTION");

  for (let i = 0; i < 1000; i++) {
    db.run("INSERT INTO numbers (value) VALUES (?)", [i]);
  }

  db.run("COMMIT");
});

// Create indexes for frequently queried columns
db.run("CREATE INDEX IF NOT EXISTS idx_users_email ON users(email)");
db.run("CREATE INDEX IF NOT EXISTS idx_posts_user_id ON posts(user_id)");
```

### Data Validation

`````javascript
function validateUser(user) {
  const errors = [];

  // Check required fields
  if (!user.name) {
    errors.push('Name is required');
  }

  if (!user.email) {
    errors.push('Email is required');
  } else if (!user.email.includes('@')) {
    errors.push('Email is invalid');
  }

  // Check field lengths
  if (user.name && user.name.length > 100) {
    errors.push('Name must be less than 100 characters');
  }

  if (user.email && user.email.length > 255) {
    errors.push('Email must be less than 255 characters');
  }

  // Return validation result
  return {
    valid: errors.length === 0,
    errors
  };
}


````javascript
// Usage
function createUser(userData) {
  // Validate input
  const validation = validateUser(userData);

  if (!validation.valid) {
    console.error('Validation errors:', validation.errors);
    return { success: false, errors: validation.errors };
  }

  // Sanitize input
  const sanitizedData = {
    name: userData.name.trim(),
    email: userData.email.trim().toLowerCase()
  };

  // Insert into database
  db.run(
    'INSERT INTO users (name, email) VALUES (?, ?)',
    [sanitizedData.name, sanitizedData.email],
    function(err) {
      if (err) {
        if (err.code === 'SQLITE_CONSTRAINT') {
          return { success: false, errors: ['Email already exists'] };
        }
        return { success: false, errors: ['Database error'] };
      }

      return { success: true, userId: this.lastID };
    }
  );
}
```

### Organizing Database Code

A well-organized approach separates database operations into logical modules:

````javascript
// db/models/user.js
const db = require('../database').getDatabase();

const User = {
  // Create a new user
  create: function(userData) {
    return new Promise((resolve, reject) => {
      db.run(
        'INSERT INTO users (name, email) VALUES (?, ?)',
        [userData.name, userData.email],
        function(err) {
          if (err) return reject(err);
          resolve({ id: this.lastID, ...userData });
        }
      );
    });
  },

  // Find user by ID
  findById: function(id) {
    return new Promise((resolve, reject) => {
      db.get('SELECT * FROM users WHERE id = ?', [id], (err, row) => {
        if (err) return reject(err);
        resolve(row);
      });
    });
  },

  // Find user by email
  findByEmail: function(email) {
    return new Promise((resolve, reject) => {
      db.get('SELECT * FROM users WHERE email = ?', [email], (err, row) => {
        if (err) return reject(err);
        resolve(row);
      });
    });
  },

  // Update user
  update: function(id, userData) {
    return new Promise((resolve, reject) => {
      db.run(
        'UPDATE users SET name = ?, email = ? WHERE id = ?',
        [userData.name, userData.email, id],
        function(err) {
          if (err) return reject(err);
          resolve({ changes: this.changes });
        }
      );
    });
  },

  // Delete user
  delete: function(id) {
    return new Promise((resolve, reject) => {
      db.run('DELETE FROM users WHERE id = ?', [id], function(err) {
        if (err) return reject(err);
        resolve({ changes: this.changes });
      });
    });
  },

  // List all users
  findAll: function() {
    return new Promise((resolve, reject) => {
      db.all('SELECT * FROM users', [], (err, rows) => {
        if (err) return reject(err);
        resolve(rows);
      });
    });
  }
};

module.exports = User;
```

### Using the Model in Your Application

````javascript
// routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../db/models/user');

// Get all users
router.get('/', async (req, res) => {
  try {
    const users = await User.findAll();
    res.json(users);
  } catch (err) {
    console.error('Error fetching users:', err);
    res.status(500).json({ error: 'Failed to fetch users' });
  }
});

// Get user by ID
router.get('/:id', async (req, res) => {
  try {
    const user = await User.findById(req.params.id);

    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.json(user);
  } catch (err) {
    console.error('Error fetching user:', err);
    res.status(500).json({ error: 'Failed to fetch user' });
  }
});

// Create new user
router.post('/', async (req, res) => {
  try {
    // Validate request body
    if (!req.body.name || !req.body.email) {
      return res.status(400).json({ error: 'Name and email are required' });
    }

    // Check if email already exists
    const existingUser = await User.findByEmail(req.body.email);
    if (existingUser) {
      return res.status(409).json({ error: 'Email already in use' });
    }

    // Create user
    const user = await User.create({
      name: req.body.name,
      email: req.body.email
    });

    res.status(201).json(user);
  } catch (err) {
    console.error('Error creating user:', err);
    res.status(500).json({ error: 'Failed to create user' });
  }
});

// Update user
router.put('/:id', async (req, res) => {
  try {
    // Check if user exists
    const user = await User.findById(req.params.id);
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    // Update user
    const result = await User.update(req.params.id, {
      name: req.body.name || user.name,
      email: req.body.email || user.email
    });

    if (result.changes === 0) {
      return res.status(404).json({ error: 'User not found or no changes made' });
    }

    // Get updated user
    const updatedUser = await User.findById(req.params.id);
    res.json(updatedUser);
  } catch (err) {
    console.error('Error updating user:', err);
    res.status(500).json({ error: 'Failed to update user' });
  }
});

// Delete user
router.delete('/:id', async (req, res) => {
  try {
    const result = await User.delete(req.params.id);

    if (result.changes === 0) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.status(204).send();
  } catch (err) {
    console.error('Error deleting user:', err);
    res.status(500).json({ error: 'Failed to delete user' });
  }
});

module.exports = router;
```

### Database Backup

````javascript
const fs = require('fs');
const path = require('path');

// Function to backup the database
function backupDatabase() {
  const db = require('./database').getDatabase();
  const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
  const backupDir = path.join(__dirname, 'backups');
  const backupPath = path.join(backupDir, `backup-${timestamp}.sqlite`);

  // Create backup directory if it doesn't exist
  if (!fs.existsSync(backupDir)) {
    fs.mkdirSync(backupDir, { recursive: true });
  }

  return new Promise((resolve, reject) => {
    // Create a backup using the VACUUM INTO command (SQLite 3.27.0+)
    db.run(`VACUUM INTO '${backupPath}'`, (err) => {
      if (err) {
        console.error('Backup failed:', err);
        reject(err);
      } else {
        console.log(`Database backed up to: ${backupPath}`);
        resolve(backupPath);
      }
    });
  });
}

// Schedule regular backups
function scheduleBackups() {
  // Backup immediately on startup
  backupDatabase()
    .then(() => console.log('Initial backup completed'))
    .catch(err => console.error('Initial backup failed:', err));

  // Schedule daily backups (24 hours)
  const TWENTY_FOUR_HOURS = 24 * 60 * 60 * 1000;
  setInterval(() => {
    backupDatabase()
      .then(() => console.log('Scheduled backup completed'))
      .catch(err => console.error('Scheduled backup failed:', err));
  }, TWENTY_FOUR_HOURS);
}

module.exports = {
  backupDatabase,
  scheduleBackups
};
```

### Testing Database Operations

````javascript
// test/user.test.js
const assert = require('assert');
const sqlite3 = require('sqlite3').verbose();
const User = require('../db/models/user');

describe('User Model', function() {
  let db;

  // Set up test database before tests
  before(function(done) {
    // Create an in-memory database for testing
    db = new sqlite3.Database(':memory:');

    // Create tables
    db.serialize(() => {
      db.run(`
        CREATE TABLE users (
          id INTEGER PRIMARY KEY AUTOINCREMENT,
          name TEXT NOT NULL,
          email TEXT UNIQUE,
          created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
      `, (err) => {
        if (err) return done(err);
        done();
      });
    });
  });

  // Clean up after tests
  after(function(done) {
    db.close(done);
  });

  // Clear data between tests
  beforeEach(function(done) {
    db.run('DELETE FROM users', done);
  });

  describe('#create()', function() {
    it('should create a new user', async function() {
      const userData = { name: 'Test User', email: 'test@example.com' };
      const user = await User.create(userData);

      assert.strictEqual(user.name, userData.name);
      assert.strictEqual(user.email, userData.email);
      assert.ok(user.id > 0);
    });

    it('should reject duplicate emails', async function() {
      const userData = { name: 'Test User', email: 'test@example.com' };

      await User.create(userData);

      try {
        await User.create(userData);
        assert.fail('Should have thrown an error for duplicate email');
      } catch (err) {
        assert.ok(err.code === 'SQLITE_CONSTRAINT');
      }
    });
  });

  describe('#findById()', function() {
    it('should find a user by ID', async function() {
      const userData = { name: 'Test User', email: 'test@example.com' };
      const createdUser = await User.create(userData);

      const user = await User.findById(createdUser.id);

      assert.strictEqual(user.name, userData.name);
      assert.strictEqual(user.email, userData.email);
    });

    it('should return undefined for non-existent ID', async function() {
      const user = await User.findById(999);
      assert.strictEqual(user, undefined);
    });
  });

  // Add more tests for other methods...
});
```

### Logging Database Operations

````javascript
// db/logger.js
const fs = require('fs');
const path = require('path');

// Create a logger
class DatabaseLogger {
  constructor(options = {}) {
    this.logToConsole = options.console !== false;
    this.logToFile = options.file !== false;
    this.logLevel = options.level || 'info';
    this.logPath = options.path || path.join(__dirname, '../logs');

    // Create log directory if it doesn't exist
    if (this.logToFile && !fs.existsSync(this.logPath)) {
      fs.mkdirSync(this.logPath, { recursive: true });
    }

    // Log levels
    this.levels = {
      error: 0,
      warn: 1,
      info: 2,
      debug: 3
    };
  }

  _shouldLog(level) {
    return this.levels[level] <= this.levels[this.logLevel];
  }

  _formatMessage(level, message, data) {
    const timestamp = new Date().toISOString();
    let logMessage = `[${timestamp}] [${level.toUpperCase()}] ${message}`;

    if (data) {
      if (typeof data === 'object') {
        logMessage += ` ${JSON.stringify(data)}`;
      } else {
        logMessage += ` ${data}`;
      }
    }

    return logMessage;
  }

  _writeToFile(message, level) {
    if (!this.logToFile) return;

    const date = new Date().toISOString().split('T')[0];
    const logFile = path.join(this.logPath, `${date}.log`);

    fs.appendFile(logFile, message + '\n', (err) => {
      if (err) {
        console.error('Error writing to log file:', err);
      }
    });
  }

  log(level, message, data) {
    if (!this._shouldLog(level)) return;

    const formattedMessage = this._formatMessage(level, message, data);

    if (this.logToConsole) {
      if (level === 'error') {
        console.error(formattedMessage);
      } else if (level === 'warn') {
        console.warn(formattedMessage);
      } else {
        console.log(formattedMessage);
      }
    }

    this._writeToFile(formattedMessage, level);
  }

  error(message, data) {
    this.log('error', message, data);
  }

  warn(message, data) {
    this.log('warn', message, data);
  }

  info(message, data) {
    this.log('info', message, data);
  }

  debug(message, data) {
    this.log('debug', message, data);
  }

  // Log database queries
  query(sql, params) {
    this.debug('Executing query', { sql, params });
  }

  // Log query results
  result(sql, result) {
    this.debug('Query result', {
      sql: sql.substring(0, 100) + (sql.length > 100 ? '...' : ''),
      rows: Array.isArray(result) ? result.length : (result ? 1 : 0)
    });
  }
}

module.exports = new DatabaseLogger({
  console: true,
  file: true,
  level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
  path: path.join(__dirname, '../logs')
});
```

### Using the Logger

````javascript
// db/models/user.js with logging
const db = require('../database').getDatabase();
const logger = require('../logger');

const User = {
  // Create a new user
  create: function(userData) {
    return new Promise((resolve, reject) => {
      const sql = 'INSERT INTO users (name, email) VALUES (?, ?)';
      const params = [userData.name, userData.email];

      logger.query(sql, params);

      db.run(sql, params, function(err) {
        if (err) {
          logger.error('Failed to create user', { error: err.message, userData });
          return reject(err);
        }

        const result = { id: this.lastID, ...userData };
        logger.info('User created', { id: this.lastID });
        resolve(result);
      });
    });
  },

  // Find user by ID
  findById: function(id) {
    return new Promise((resolve, reject) => {
      const sql = 'SELECT * FROM users WHERE id = ?';

      logger.query(sql, [id]);

      db.get(sql, [id], (err, row) => {
        if (err) {
          logger.error('Error finding user by ID', { error: err.message, id });
          return reject(err);
        }

        logger.result(sql, row);
        resolve(row);
      });
    });
  },

  // Find all users
  findAll: function() {
    return new Promise((resolve, reject) => {
      const sql = 'SELECT * FROM users';

      logger.query(sql);

      db.all(sql, [], (err, rows) => {
        if (err) {
          logger.error('Error finding all users', { error: err.message });
          return reject(err);
        }

        logger.result(sql, rows);
        resolve(rows);
      });
    });
  }

  // Other methods...
};

module.exports = User;
```

### Monitoring Database Performance

````javascript
// db/performance.js
const logger = require('./logger');

// Create a performance monitor
class DatabasePerformanceMonitor {
  constructor() {
    this.queries = [];
    this.slowQueryThreshold = 100; // ms
  }

  // Start timing a query
  startQuery(sql, params) {
    const query = {
      sql,
      params,
      startTime: Date.now(),
      endTime: null,
      duration: null
    };

    this.queries.push(query);
    return query;
  }

  // End timing a query
  endQuery(query) {
    query.endTime = Date.now();
    query.duration = query.endTime - query.startTime;

    // Log slow queries
    if (query.duration > this.slowQueryThreshold) {
      logger.warn('Slow query detected', {
        sql: query.sql,
        duration: query.duration + 'ms',
        threshold: this.slowQueryThreshold + 'ms'
      });
    }

    return query.duration;
  }

  // Get performance statistics
  getStats() {
    if (this.queries.length === 0) {
      return {
        totalQueries: 0,
        totalTime: 0,
        averageTime: 0,
        slowQueries: 0,
        fastestQuery: null,
        slowestQuery: null
      };
    }

    const completedQueries = this.queries.filter(q => q.endTime !== null);
    const totalTime = completedQueries.reduce((sum, q) => sum + q.duration, 0);
    const slowQueries = completedQueries.filter(q => q.duration > this.slowQueryThreshold);

    // Find fastest and slowest queries
    let fastestQuery = completedQueries[0];
    let slowestQuery = completedQueries[0];

    for (const query of completedQueries) {
      if (query.duration < fastestQuery.duration) {
        fastestQuery = query;
      }

      if (query.duration > slowestQuery.duration) {
        slowestQuery = query;
      }
    }

    return {
      totalQueries: completedQueries.length,
      totalTime,
      averageTime: totalTime / completedQueries.length,
      slowQueries: slowQueries.length,
      fastestQuery: {
        sql: fastestQuery.sql,
        duration: fastestQuery.duration
      },
      slowestQuery: {
        sql: slowestQuery.sql,
        duration: slowestQuery.duration
      }
    };
  }

  // Reset statistics
  reset() {
    this.queries = [];
  }
}

module.exports = new DatabasePerformanceMonitor();
```

### Using the Performance Monitor

````javascript
// db/database.js with performance monitoring
const sqlite3 = require('sqlite3').verbose();
const path = require('path');
const logger = require('./logger');
const perfMonitor = require('./performance');

// Database file path
const dbPath = path.resolve(__dirname, '../database.sqlite');

// Create a singleton connection
let db = null;

// Wrap a database method with performance monitoring
function monitorPerformance(method, methodName) {
  return function(...args) {
    const callback = args[args.length - 1];

    // Only wrap if the last argument is a callback
    if (typeof callback !== 'function') {
      return method.apply(this, args);
    }

    // Get SQL from arguments
    const sql = args[0];
    const params = args.length > 2 ? args[1] : [];

    // Start monitoring
    const query = perfMonitor.startQuery(sql, params);

    // Replace the callback with our monitored version
    args[args.length - 1] = function(...cbArgs) {
      perfMonitor.endQuery(query);
      return callback.apply(this, cbArgs);
    };

    return method.apply(this, args);
  };
}

function getDatabase() {
  if (db === null) {
    db = new sqlite3.Database(dbPath, (err) => {
      if (err) {
        logger.error('Could not connect to database', { error: err.message });
      } else {
        logger.info('Connected to database', { path: dbPath });
      }
    });

    // Enable foreign keys
    db.run('PRAGMA foreign_keys = ON');

    // Configure for better performance
    db.run('PRAGMA journal_mode = WAL');

    // Wrap methods with performance monitoring
    db.run = monitorPerformance(db.run, 'run');
    db.get = monitorPerformance(db.get, 'get');
    db.all = monitorPerformance(db.all, 'all');
    db.each = monitorPerformance(db.each, 'each');
  }

  return db;
}

function closeDatabase() {
  if (db) {
    db.close((err) => {
      if (err) {
        logger.error('Error closing database', { error: err.message });
      } else {
        logger.info('Database connection closed');

        // Log performance stats
        const stats = perfMonitor.getStats();
        logger.info('Database performance summary', stats);

        db = null;
      }
    });
  }
}

// Handle application shutdown
process.on('SIGINT', () => {
  closeDatabase();
  process.exit(0);
});

module.exports = {
  getDatabase,
  closeDatabase
};
```

### Final Tips

1. **Keep SQLite Updated**: Always use the latest version of SQLite and its Node.js modules for security and performance improvements.

2. **Use WAL Mode**: Write-Ahead Logging (WAL) mode can significantly improve performance:
   ```javascript
   db.run('PRAGMA journal_mode = WAL');
   ```

3. **Index Wisely**: Create indexes for columns you frequently search or join on, but don't over-index:
   ```javascript
   db.run('CREATE INDEX IF NOT EXISTS idx_users_email ON users(email)');
   ```

4. **Regular Maintenance**: Periodically run VACUUM to optimize your database:
   ```javascript
   db.run('VACUUM');
   ```

5. **Analyze Query Performance**: Use the EXPLAIN QUERY PLAN command to understand how SQLite executes your queries:
   ```javascript
   db.all('EXPLAIN QUERY PLAN SELECT * FROM users WHERE email = ?', ['user@example.com'],
     (err, plan) => {
       console.log('Query plan:', plan);
     }
   );
   ```

6. **Use Transactions**: Wrap multiple related operations in transactions for better performance and data integrity.

7. **Handle Concurrency**: Be aware of SQLite's concurrency limitations and use appropriate locking strategies.

8. **Backup Regularly**: Implement a regular backup strategy to prevent data loss.

9. **Validate Input**: Always validate and sanitize user input before using it in database operations.

10. **Parameterize Queries**: Never concatenate user input directly into SQL strings to prevent SQL injection.
`````
