# CollectTracker SQLite Node.js Example

This document explains how the CollectTracker application uses SQLite with Node.js, including database configuration, initialization, schema, and query patterns.

## Table of Contents

- [Database Initialization](#database-initialization)
- [Database Schema](#database-schema)

## Database Configuration

[[Link to db.config.js in repository](https://github.com/Bighairymtnman/CollectTracker/blob/main/server/db.config.js)]

The database configuration file (`server/db.config.js`) handles setting up the SQLite connection and initializing the database structure. It also provides promise-based wrappers around the SQLite methods for easier async/await usage.

```javascript
const sqlite3 = require("sqlite3").verbose();
const path = require("path");
const fs = require("fs");
const { app } = require("electron");

const getAppPath = () => {
  if (process.env.NODE_ENV === "development") {
    return __dirname;
  }
  try {
    const { app } = require("electron");
    return app.getPath("userData");
  } catch {
    return __dirname;
  }
};

// Ensure database directory exists
const dbDirectory = path.join(getAppPath(), "database");
if (!fs.existsSync(dbDirectory)) {
  fs.mkdirSync(dbDirectory, { recursive: true });
}

// Create database connection
const db = new sqlite3.Database(
  path.join(dbDirectory, "collecttracker.db"),
  (err) => {
    if (err) {
      console.error("Error opening database:", err);
    } else {
      console.log("Connected to SQLite database");
      initializeDatabase();
    }
  }
);

// Initialize database tables
function initializeDatabase() {
  db.serialize(() => {
    // Create tables if they don't exist
    db.run(`CREATE TABLE IF NOT EXISTS collections (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            type TEXT,
            description TEXT,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )`);

    db.run(`CREATE TABLE IF NOT EXISTS items (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            collection_id INTEGER,
            title TEXT NOT NULL,
            type TEXT,
            description TEXT,
            data TEXT,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (collection_id) REFERENCES collections(id)
        )`);

    db.run(`CREATE TABLE IF NOT EXISTS categories (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            collection_id INTEGER,
            name TEXT NOT NULL,
            FOREIGN KEY (collection_id) REFERENCES collections(id)
        )`);

    db.run(`CREATE TABLE IF NOT EXISTS item_categories (
            item_id INTEGER,
            category_id INTEGER,
            PRIMARY KEY (item_id, category_id),
            FOREIGN KEY (item_id) REFERENCES items(id),
            FOREIGN KEY (category_id) REFERENCES categories(id)
        )`);

    db.run(`CREATE TABLE IF NOT EXISTS images (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            item_id INTEGER,
            image_data BLOB,
            is_cover BOOLEAN DEFAULT 0,
            FOREIGN KEY (item_id) REFERENCES items(id)
        )`);

    db.run(`CREATE TABLE IF NOT EXISTS item_photos (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            item_id INTEGER,
            image_data BLOB,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (item_id) REFERENCES items(id)
        )`);
  });
}

// Wrap SQLite methods in promises for async/await support
const dbAsync = {
  all: (sql, params = []) => {
    return new Promise((resolve, reject) => {
      db.all(sql, params, (err, rows) => {
        if (err) reject(err);
        else resolve(rows);
      });
    });
  },
  run: (sql, params = []) => {
    return new Promise((resolve, reject) => {
      db.run(sql, params, function (err) {
        if (err) reject(err);
        else resolve({ id: this.lastID, changes: this.changes });
      });
    });
  },
  get: (sql, params = []) => {
    return new Promise((resolve, reject) => {
      db.get(sql, params, (err, row) => {
        if (err) reject(err);
        else resolve(row);
      });
    });
  },
};

module.exports = dbAsync;
```

### Key Features:

1. **Dynamic Database Path**: The application determines the appropriate database path based on whether it's running in development mode or as an Electron app.

2. **Directory Creation**: Ensures the database directory exists before attempting to create the database file.

3. **Database Connection**: Creates a connection to the SQLite database with verbose mode enabled for detailed error messages.

4. **Table Initialization**: Automatically creates all necessary tables if they don't exist when the database connection is established.

5. **Promise Wrappers**: Wraps the callback-based SQLite methods (`all`, `run`, `get`) in promises to support modern async/await syntax.

## Database Initialization

[[Link to initDb.js in repository](https://github.com/Bighairymtnman/CollectTracker/blob/main/server/utils/initDb.js)]

The database initialization file (`server/utils/initDb.js`) creates all the necessary tables for the application if they don't already exist. This file provides a more structured approach to database initialization compared to the inline initialization in the db.config.js file.

```javascript
const db = require("../db.config");

async function initializeDatabase() {
  try {
    // Collections table
    await db.run(`CREATE TABLE IF NOT EXISTS collections (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            type TEXT NOT NULL,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            description TEXT
        )`);

    // Categories table
    await db.run(`CREATE TABLE IF NOT EXISTS categories (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            collection_id INTEGER NOT NULL,
            name TEXT NOT NULL,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (collection_id) REFERENCES collections (id)
        )`);

    // Items table
    await db.run(`CREATE TABLE IF NOT EXISTS items (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            collection_id INTEGER,
            title TEXT NOT NULL,
            type TEXT NOT NULL,
            data TEXT,
            description TEXT,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (collection_id) REFERENCES collections (id)
        )`);

    // Images table
    await db.run(`CREATE TABLE IF NOT EXISTS images (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            item_id INTEGER NOT NULL,
            image_data BLOB NOT NULL,
            is_cover INTEGER DEFAULT 0,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (item_id) REFERENCES items (id) ON DELETE CASCADE
        )`);

    // Item_categories table
    await db.run(`CREATE TABLE IF NOT EXISTS item_categories (
            item_id INTEGER NOT NULL,
            category_id INTEGER NOT NULL,
            PRIMARY KEY (item_id, category_id),
            FOREIGN KEY (item_id) REFERENCES items (id),
            FOREIGN KEY (category_id) REFERENCES categories (id)
        )`);

    // Item_photos table
    await db.run(`CREATE TABLE IF NOT EXISTS item_photos (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            item_id INTEGER NOT NULL,
            image_data BLOB NOT NULL,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (item_id) REFERENCES items (id) ON DELETE CASCADE
        )`);

    // Item_properties table
    await db.run(`CREATE TABLE IF NOT EXISTS item_properties (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            item_id INTEGER,
            property_name TEXT NOT NULL,
            property_value TEXT,
            FOREIGN KEY (item_id) REFERENCES items (id)
        )`);

    console.log("Database initialized successfully");
  } catch (error) {
    console.error("Error initializing database:", error);
  }
}

initializeDatabase();
```

### Key Features:

1. **Async/Await Pattern**: Uses the promise-based wrapper methods from db.config.js to create tables using async/await for cleaner code.

2. **Error Handling**: Wraps the initialization in a try/catch block to properly handle and log any errors during table creation.

3. **Table Structure**:

   - **Collections**: Stores information about different collections (e.g., book collections, stamp collections)
   - **Categories**: Allows for categorization within collections
   - **Items**: Stores individual items within collections
   - **Images**: Stores images associated with items, with a flag for cover images
   - **Item_categories**: Junction table for many-to-many relationship between items and categories
   - **Item_photos**: Stores additional photos for items
   - **Item_properties**: Stores custom properties for items in a key-value format

4. **Relationships**:

   - Collections have many items and categories
   - Items belong to a collection and can have many categories
   - Items can have multiple images and properties
   - Categories belong to a collection and can be associated with many items

5. **Cascading Deletes**: The `ON DELETE CASCADE` clause on some foreign keys ensures that when a parent record is deleted, related child records are automatically deleted as well.
