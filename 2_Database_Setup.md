# Database Integration - PostgreSQL & MongoDB

## Overview

This chapter covers integrating databases into your Node.js backend. You'll learn connection setup, schema definition, relationships, and controller usage for both **PostgreSQL** (relational) and **MongoDB** (NoSQL).

**What you'll learn**:
- Database connection and configuration
- Defining models, schemas, and constraints
- Establishing relationships between data
- Using databases in controllers
- When to choose which database

---

## 1. Understanding Database Types

### Relational Databases (PostgreSQL)

**Simple explanation**: Data is stored in tables with strict rules about what goes in each column. Tables can connect to each other through relationships.

**Technical explanation**: Relational databases use structured schemas with predefined data types and relationships. They follow ACID principles (Atomicity, Consistency, Isolation, Durability) and use SQL for querying. Data is normalized across tables to reduce redundancy.

**Key characteristics**:
- **Schema-enforced**: Structure defined before data insertion
- **Relationships**: Foreign keys connect tables
- **ACID transactions**: Guarantees data integrity
- **Joins**: Combine data from multiple tables
- **Normalization**: Reduce data duplication

### NoSQL Databases (MongoDB)

**Simple explanation**: Data is stored as flexible documents (like JSON objects). Each document can have different fields without strict rules.

**Technical explanation**: MongoDB is a document-oriented database that stores data in BSON (Binary JSON) format. It provides flexible schemas, horizontal scalability, and eventual consistency. Documents are grouped in collections and can embed related data.

**Key characteristics**:
- **Schema-flexible**: Structure can evolve without migrations
- **Document-based**: Nested data in single document
- **Horizontal scaling**: Easy to distribute across servers
- **Denormalization**: Embed related data for fast reads
- **Eventually consistent**: Prioritizes availability over immediate consistency

---

## 2. When to Use Which Database

### Decision Framework

#### Use PostgreSQL When:

1. **Data structure is well-defined and stable**
   - Banking systems, accounting software, ERP systems
   - Schema changes are infrequent and planned
   - Example: Financial transactions with fixed fields (amount, date, account_id)

2. **Complex relationships and joins are required**
   - Data is highly interconnected
   - Need to query across multiple entities efficiently
   - Example: E-commerce with products, orders, customers, inventory, suppliers

3. **ACID compliance is critical**
   - Data integrity cannot be compromised
   - Transactions must be atomic (all-or-nothing)
   - Example: Payment processing, booking systems, healthcare records

4. **Complex queries and aggregations**
   - Reporting and analytics on structured data
   - Need SQL's powerful querying capabilities
   - Example: Business intelligence dashboards, financial reports

5. **Data consistency is more important than availability**
   - Systems where outdated data causes problems
   - Single source of truth is essential
   - Example: Inventory management, ticketing systems

**Technical why**: PostgreSQL's relational model enforces referential integrity through foreign keys, ensuring data consistency. ACID transactions guarantee that multi-step operations either complete fully or not at all. The query optimizer can efficiently join normalized tables, and indexes on foreign keys make these operations fast.

#### Use MongoDB When:

1. **Schema evolves frequently**
   - Rapid prototyping and iteration
   - New fields added regularly
   - Example: Content management systems, social media platforms

2. **Data is hierarchical or document-oriented**
   - Natural fit for nested structures
   - Related data accessed together
   - Example: User profiles with preferences, settings, activity logs

3. **Horizontal scaling is required**
   - Need to handle massive data volumes
   - Traffic grows unpredictably
   - Example: IoT sensor data, logs, real-time analytics

4. **High write throughput**
   - Insert-heavy workloads
   - Can tolerate eventual consistency
   - Example: Event logging, metrics collection, activity streams

5. **Flexible data models**
   - Different document types in same collection
   - Schema varies by context
   - Example: Multi-tenant SaaS, polymorphic data

**Technical why**: MongoDB's document model allows embedding related data, reducing the need for joins and enabling single-document atomicity. Sharding distributes data across machines for horizontal scaling. The flexible schema means no migrations—just start inserting documents with new fields. Write operations are fast because they don't require immediate consistency checks across documents.

### Hybrid Approach

Many applications use **both**:
- PostgreSQL for critical transactional data (orders, payments)
- MongoDB for flexible, high-volume data (logs, user sessions, cache)

---

## 3. Connection Setup

### PostgreSQL Connection

#### Installing Dependencies

```bash
npm install pg
```

**What is `pg`?**: The official PostgreSQL client library for Node.js. It handles connection pooling, query execution, and result parsing.

#### Connection Configuration

**File**: `src/config/database.js`

```js
const { Pool } = require("pg");
const config = require("./env");

// Create connection pool
const pool = new Pool({
  host: config.db.host,
  port: config.db.port,
  database: config.db.name,
  user: config.db.user,
  password: config.db.password,
  max: 20,                    // Maximum connections in pool
  idleTimeoutMillis: 30000,   // Close idle connections after 30s
  connectionTimeoutMillis: 2000,  // Wait 2s for connection
});

// Test connection
pool.on("connect", () => {
  console.log("PostgreSQL connected");
});

pool.on("error", (err) => {
  console.error("PostgreSQL error:", err);
  process.exit(-1);
});

module.exports = pool;
```

**Why connection pooling?**: Creating new database connections is expensive (time + resources). A pool maintains multiple reusable connections, significantly improving performance under load.

**Pool configuration explained**:
- `max`: Limits concurrent connections to database
- `idleTimeoutMillis`: Closes unused connections to free resources
- `connectionTimeoutMillis`: Fails fast if database is unresponsive

#### Using the Connection

```js
const pool = require("../config/database");

// Query method
async function queryDatabase() {
  const result = await pool.query(
    "SELECT * FROM table_name WHERE column_name = $1",
    [value]
  );
  return result.rows;
}

// Transaction method
async function executeTransaction() {
  const client = await pool.connect();
  try {
    await client.query("BEGIN");
    await client.query("INSERT INTO ...", [values]);
    await client.query("UPDATE ...", [values]);
    await client.query("COMMIT");
  } catch (error) {
    await client.query("ROLLBACK");
    throw error;
  } finally {
    client.release();  // Return connection to pool
  }
}
```

**Parameterized queries** (`$1`, `$2`): Prevents SQL injection by treating user input as data, not code.

### MongoDB Connection

#### Installing Dependencies

```bash
npm install mongoose
```

**What is Mongoose?**: An Object Document Mapper (ODM) that provides schema validation, middleware hooks, and a cleaner API over the native MongoDB driver.

#### Connection Configuration

**File**: `src/config/database.js`

```js
const mongoose = require("mongoose");
const config = require("./env");

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(config.db.uri, {
      maxPoolSize: 10,           // Maximum connections
      serverSelectionTimeoutMS: 5000,  // Timeout for server selection
      socketTimeoutMS: 45000,    // Socket timeout
    });

    console.log(`MongoDB connected: ${conn.connection.host}`);

    // Connection events
    mongoose.connection.on("error", (err) => {
      console.error("MongoDB error:", err);
    });

    mongoose.connection.on("disconnected", () => {
      console.log("MongoDB disconnected");
    });

    // Graceful shutdown
    process.on("SIGINT", async () => {
      await mongoose.connection.close();
      process.exit(0);
    });

  } catch (error) {
    console.error("MongoDB connection error:", error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

**Connection string format**:
```
mongodb://username:password@host:port/database_name
mongodb+srv://username:password@cluster.mongodb.net/database_name  // MongoDB Atlas
```

#### Using the Connection

**File**: `src/server.js`

```js
require("dotenv").config();
const app = require("./app");
const connectDB = require("./config/database");
const config = require("./config/env");

// Connect to MongoDB before starting server
connectDB().then(() => {
  app.listen(config.port, () => {
    console.log(`Server running on port ${config.port}`);
  });
});
```

**Why connect before listening?**: Ensures database is ready before accepting requests. Prevents errors from routes trying to query a non-connected database.

---

## 4. Defining Schemas and Models

### PostgreSQL Table Creation

#### Basic Table Syntax

```sql
CREATE TABLE table_name (
  column_name1 DATA_TYPE CONSTRAINT,
  column_name2 DATA_TYPE CONSTRAINT,
  column_name3 DATA_TYPE CONSTRAINT,
  PRIMARY KEY (column_name1),
  FOREIGN KEY (column_name2) REFERENCES other_table(id)
);
```

#### Common Data Types

| PostgreSQL Type | Description | Example |
|----------------|-------------|---------|
| `SERIAL` | Auto-incrementing integer | `id SERIAL PRIMARY KEY` |
| `INTEGER` | Whole numbers | `age INTEGER` |
| `VARCHAR(n)` | Variable-length string (max n) | `name VARCHAR(100)` |
| `TEXT` | Unlimited length string | `description TEXT` |
| `BOOLEAN` | True/false | `is_active BOOLEAN` |
| `TIMESTAMP` | Date and time | `created_at TIMESTAMP` |
| `NUMERIC(p,s)` | Decimal numbers | `price NUMERIC(10,2)` |
| `JSON` / `JSONB` | JSON data | `metadata JSONB` |
| `UUID` | Universally unique identifier | `uuid UUID` |

#### Common Constraints

```sql
-- NOT NULL: Field must have a value
column_name VARCHAR(100) NOT NULL

-- UNIQUE: No duplicate values
column_name VARCHAR(100) UNIQUE

-- DEFAULT: Value if none provided
column_name TIMESTAMP DEFAULT CURRENT_TIMESTAMP

-- CHECK: Custom validation rule
column_name INTEGER CHECK (column_name > 0)

-- PRIMARY KEY: Unique identifier (automatically NOT NULL + UNIQUE)
id SERIAL PRIMARY KEY

-- FOREIGN KEY: Reference to another table
other_id INTEGER REFERENCES other_table(id) ON DELETE CASCADE
```

**ON DELETE options**:
- `CASCADE`: Delete related rows automatically
- `SET NULL`: Set foreign key to NULL
- `RESTRICT`: Prevent deletion if references exist
- `NO ACTION`: Similar to RESTRICT

#### Creating Tables Programmatically

**File**: `src/models/init.sql` or in code:

```js
const pool = require("../config/database");

async function createTables() {
  await pool.query(`
    CREATE TABLE IF NOT EXISTS table_name (
      id SERIAL PRIMARY KEY,
      column1 VARCHAR(255) NOT NULL,
      column2 TEXT,
      column3 INTEGER DEFAULT 0,
      column4 BOOLEAN DEFAULT false,
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
      updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
  `);
}

module.exports = { createTables };
```

#### Model Pattern for PostgreSQL

Since PostgreSQL doesn't have built-in models like Mongoose, create a model layer:

**File**: `src/models/Model.js` (generic pattern)

```js
const pool = require("../config/database");

class Model {
  constructor(tableName) {
    this.tableName = tableName;
  }

  async findAll() {
    const result = await pool.query(
      `SELECT * FROM ${this.tableName}`
    );
    return result.rows;
  }

  async findById(id) {
    const result = await pool.query(
      `SELECT * FROM ${this.tableName} WHERE id = $1`,
      [id]
    );
    return result.rows[0];
  }

  async create(data) {
    const columns = Object.keys(data).join(", ");
    const values = Object.values(data);
    const placeholders = values.map((_, i) => `$${i + 1}`).join(", ");

    const result = await pool.query(
      `INSERT INTO ${this.tableName} (${columns}) 
       VALUES (${placeholders}) 
       RETURNING *`,
      values
    );
    return result.rows[0];
  }

  async update(id, data) {
    const entries = Object.entries(data);
    const setClause = entries
      .map(([key], i) => `${key} = $${i + 1}`)
      .join(", ");
    const values = [...entries.map(([, val]) => val), id];

    const result = await pool.query(
      `UPDATE ${this.tableName} 
       SET ${setClause} 
       WHERE id = $${values.length} 
       RETURNING *`,
      values
    );
    return result.rows[0];
  }

  async delete(id) {
    const result = await pool.query(
      `DELETE FROM ${this.tableName} WHERE id = $1 RETURNING *`,
      [id]
    );
    return result.rows[0];
  }
}

module.exports = Model;
```

**Usage**:
```js
const Model = require("./Model");
const EntityModel = new Model("table_name");

// Now use: EntityModel.findAll(), EntityModel.create(), etc.
```

### MongoDB Schema Definition

#### Basic Schema Syntax

```js
const mongoose = require("mongoose");

const schemaName = new mongoose.Schema({
  fieldName1: {
    type: DataType,
    required: true,
    unique: true,
    default: defaultValue
  },
  fieldName2: DataType,  // Shorthand when only specifying type
  fieldName3: {
    type: DataType,
    enum: [value1, value2],
    validate: {
      validator: function(value) {
        return validationLogic;
      },
      message: "Error message"
    }
  }
}, {
  timestamps: true  // Adds createdAt and updatedAt automatically
});

const ModelName = mongoose.model("CollectionName", schemaName);
module.exports = ModelName;
```

#### Common Data Types

| Mongoose Type | Description | Example |
|--------------|-------------|---------|
| `String` | Text data | `name: String` |
| `Number` | Integers or decimals | `age: Number` |
| `Boolean` | True/false | `isActive: Boolean` |
| `Date` | Date/time | `createdAt: Date` |
| `ObjectId` | MongoDB's unique ID | `userId: mongoose.Schema.Types.ObjectId` |
| `Array` | List of values | `tags: [String]` |
| `Mixed` | Any type | `metadata: mongoose.Schema.Types.Mixed` |
| `Buffer` | Binary data | `file: Buffer` |

#### Schema Options and Validators

```js
const schema = new mongoose.Schema({
  // String validators
  fieldName: {
    type: String,
    required: [true, "Custom error message"],
    minlength: 3,
    maxlength: 100,
    trim: true,           // Remove whitespace
    lowercase: true,      // Convert to lowercase
    uppercase: true,      // Convert to uppercase
    match: /regex/,       // Must match pattern
    enum: ["value1", "value2"]  // Must be one of these
  },

  // Number validators
  fieldName: {
    type: Number,
    min: 0,
    max: 100,
    default: 0
  },

  // Custom validator
  fieldName: {
    type: String,
    validate: {
      validator: function(value) {
        return value.length > 5;
      },
      message: "Must be longer than 5 characters"
    }
  },

  // Nested object
  nestedField: {
    subField1: String,
    subField2: Number
  },

  // Array of strings
  arrayField: [String],

  // Array of objects
  arrayField: [{
    field1: String,
    field2: Number
  }]
}, {
  timestamps: true,        // Auto createdAt/updatedAt
  toJSON: { virtuals: true },   // Include virtuals in JSON
  toObject: { virtuals: true }  // Include virtuals in objects
});
```

#### Schema Methods and Middleware

```js
// Instance method (called on document)
schema.methods.methodName = function() {
  // 'this' refers to the document
  return someValue;
};

// Static method (called on model)
schema.statics.staticMethodName = async function(params) {
  // 'this' refers to the model
  return await this.find({ condition });
};

// Pre-save middleware (runs before saving)
schema.pre("save", async function(next) {
  // 'this' refers to the document being saved
  // Example: Hash password before saving
  if (this.isModified("password")) {
    // Do something
  }
  next();
});

// Post-save middleware (runs after saving)
schema.post("save", function(doc, next) {
  // 'doc' is the saved document
  console.log("Document saved:", doc._id);
  next();
});

// Virtual property (computed field not stored in DB)
schema.virtual("virtualFieldName").get(function() {
  return computedValue;
});
```

**Why use middleware?**: Automatically perform actions (hashing passwords, updating timestamps, logging) without cluttering controller code.

---

## 5. Relationships and References

### PostgreSQL Foreign Keys

#### One-to-Many Relationship

```sql
-- Parent table
CREATE TABLE parent_table (
  id SERIAL PRIMARY KEY,
  field1 VARCHAR(100) NOT NULL
);

-- Child table with foreign key
CREATE TABLE child_table (
  id SERIAL PRIMARY KEY,
  parent_id INTEGER NOT NULL REFERENCES parent_table(id) ON DELETE CASCADE,
  field2 VARCHAR(100)
);
```

**Explanation**: Each child has exactly one parent, but a parent can have multiple children.

#### Many-to-Many Relationship

```sql
-- First entity
CREATE TABLE entity_one (
  id SERIAL PRIMARY KEY,
  field1 VARCHAR(100)
);

-- Second entity
CREATE TABLE entity_two (
  id SERIAL PRIMARY KEY,
  field2 VARCHAR(100)
);

-- Junction table (connecting the two)
CREATE TABLE entity_one_entity_two (
  entity_one_id INTEGER REFERENCES entity_one(id) ON DELETE CASCADE,
  entity_two_id INTEGER REFERENCES entity_two(id) ON DELETE CASCADE,
  PRIMARY KEY (entity_one_id, entity_two_id)
);
```

**Explanation**: A junction table connects two entities where each can have multiple of the other.

#### Querying with Joins

```js
// INNER JOIN: Only rows with matches in both tables
const result = await pool.query(`
  SELECT 
    parent_table.field1,
    child_table.field2
  FROM parent_table
  INNER JOIN child_table ON parent_table.id = child_table.parent_id
  WHERE parent_table.id = $1
`, [id]);

// LEFT JOIN: All parent rows, matching child rows (or NULL)
const result = await pool.query(`
  SELECT 
    parent_table.*,
    child_table.*
  FROM parent_table
  LEFT JOIN child_table ON parent_table.id = child_table.parent_id
  WHERE parent_table.id = $1
`, [id]);

// Many-to-Many query
const result = await pool.query(`
  SELECT 
    entity_one.*,
    entity_two.*
  FROM entity_one
  INNER JOIN entity_one_entity_two ON entity_one.id = entity_one_entity_two.entity_one_id
  INNER JOIN entity_two ON entity_one_entity_two.entity_two_id = entity_two.id
  WHERE entity_one.id = $1
`, [id]);
```

**Join types**:
- `INNER JOIN`: Only matching rows from both tables
- `LEFT JOIN`: All rows from left table, matching from right (NULL if no match)
- `RIGHT JOIN`: All rows from right table, matching from left
- `FULL OUTER JOIN`: All rows from both, NULL where no match

### MongoDB References

#### One-to-Many with References

```js
// Parent schema
const parentSchema = new mongoose.Schema({
  field1: String,
  field2: String
});

const ParentModel = mongoose.model("Parent", parentSchema);

// Child schema with reference
const childSchema = new mongoose.Schema({
  parentId: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "Parent",         // Reference to Parent model
    required: true
  },
  field3: String
});

const ChildModel = mongoose.model("Child", childSchema);
```

#### Populating References

```js
// Find child and populate parent data
const child = await ChildModel
  .findById(childId)
  .populate("parentId");  // Replaces parentId with actual parent document

// result: { _id: ..., parentId: { field1: "...", field2: "..." }, field3: "..." }

// Populate with selected fields only
const child = await ChildModel
  .findById(childId)
  .populate("parentId", "field1 field2");  // Only include these fields

// Populate multiple references
const document = await Model
  .findById(id)
  .populate("reference1")
  .populate("reference2");

// Nested population
const document = await Model
  .findById(id)
  .populate({
    path: "reference1",
    populate: { path: "nestedReference" }
  });
```

**What is populate?**: Mongoose replaces ObjectId references with actual documents from referenced collections. Similar to SQL joins but happens in application layer.

#### Many-to-Many with Arrays

```js
// First entity with array of references
const entityOneSchema = new mongoose.Schema({
  field1: String,
  entityTwoIds: [{
    type: mongoose.Schema.Types.ObjectId,
    ref: "EntityTwo"
  }]
});

const EntityOne = mongoose.model("EntityOne", entityOneSchema);

// Second entity with array of references (bidirectional)
const entityTwoSchema = new mongoose.Schema({
  field2: String,
  entityOneIds: [{
    type: mongoose.Schema.Types.ObjectId,
    ref: "EntityOne"
  }]
});

const EntityTwo = mongoose.model("EntityTwo", entityTwoSchema);

// Query with population
const entityOne = await EntityOne
  .findById(id)
  .populate("entityTwoIds");
```

#### Embedded Documents (Alternative Approach)

Instead of referencing, embed related data directly:

```js
// Embedding approach (denormalized)
const parentSchema = new mongoose.Schema({
  field1: String,
  childrenData: [{
    field2: String,
    field3: Number
  }]
});

// Use when:
// - Related data is always accessed together
// - Child data is small and doesn't change often
// - Read performance is critical
```

**Reference vs Embed decision**:
- **Reference**: Data is large, changes frequently, or shared across documents
- **Embed**: Data is small, rarely changes, and always accessed together

---

## 6. Using Databases in Controllers

### PostgreSQL Controller Patterns

```js
const pool = require("../config/database");

// GET all records
exports.getAll = async (req, res) => {
  try {
    const result = await pool.query(
      "SELECT * FROM table_name ORDER BY created_at DESC"
    );
    
    res.json({
      success: true,
      data: result.rows
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
};

// GET single record by ID
exports.getById = async (req, res) => {
  try {
    const { id } = req.params;
    
    const result = await pool.query(
      "SELECT * FROM table_name WHERE id = $1",
      [id]
    );
    
    if (result.rows.length === 0) {
      return res.status(404).json({
        success: false,
        error: "Record not found"
      });
    }
    
    res.json({
      success: true,
      data: result.rows[0]
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
};

// POST create new record
exports.create = async (req, res) => {
  try {
    const { field1, field2, field3 } = req.body;
    
    const result = await pool.query(
      `INSERT INTO table_name (field1, field2, field3) 
       VALUES ($1, $2, $3) 
       RETURNING *`,
      [field1, field2, field3]
    );
    
    res.status(201).json({
      success: true,
      data: result.rows[0]
    });
  } catch (error) {
    // Handle unique constraint violation
    if (error.code === "23505") {
      return res.status(409).json({
        success: false,
        error: "Record already exists"
      });
    }
    
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
};

// PUT/PATCH update record
exports.update = async (req, res) => {
  try {
    const { id } = req.params;
    const { field1, field2 } = req.body;
    
    const result = await pool.query(
      `UPDATE table_name 
       SET field1 = $1, field2 = $2, updated_at = CURRENT_TIMESTAMP 
       WHERE id = $3 
       RETURNING *`,
      [field1, field2, id]
    );
    
    if (result.rows.length === 0) {
      return res.status(404).json({
        success: false,
        error: "Record not found"
      });
    }
    
    res.json({
      success: true,
      data: result.rows[0]
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
};

// DELETE record
exports.delete = async (req, res) => {
  try {
    const { id } = req.params;
    
    const result = await pool.query(
      "DELETE FROM table_name WHERE id = $1 RETURNING *",
      [id]
    );
    
    if (result.rows.length === 0) {
      return res.status(404).json({
        success: false,
        error: "Record not found"
      });
    }
    
    res.status(204).send();
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
};

// Query with filters
exports.search = async (req, res) => {
  try {
    const { filterField, sortBy, order, limit, offset } = req.query;
    
    let query = "SELECT * FROM table_name WHERE 1=1";
    const params = [];
    let paramCount = 1;
    
    if (filterField) {
      query += ` AND field_name = $${paramCount}`;
      params.push(filterField);
      paramCount++;
    }
    
    if (sortBy) {
      query += ` ORDER BY ${sortBy} ${order === "desc" ? "DESC" : "ASC"}`;
    }
    
    if (limit) {
      query += ` LIMIT $${paramCount}`;
      params.push(parseInt(limit));
      paramCount++;
    }
    
    if (offset) {
      query += ` OFFSET $${paramCount}`;
      params.push(parseInt(offset));
    }
    
    const result = await pool.query(query, params);
    
    res.json({
      success: true,
      data: result.rows,
      count: result.rowCount
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
};
```

**PostgreSQL error codes** (useful for handling):
- `23505`: Unique violation
- `23503`: Foreign key violation
- `23502`: Not null violation
- `22P02`: Invalid text representation

### MongoDB Controller Patterns

```js
const Model = require("../models/Model");

// GET all records
exports.getAll = async (req, res) => {
  try {
    const records = await Model.find()
      .sort({ createdAt: -1 });
    
    res.json({
      success: true,
      data: records
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
};

// GET single record by ID
exports.getById = async (req, res) => {
  try {
    const { id } = req.params;
    
    const record = await Model.findById(id);
    
    if (!record) {
      return res.status(404).json({
        success: false,
        error: "Record not found"
      });
    }
    
    res.json({
      success: true,
      data: record
    });
  } catch (error) {
    // Invalid ObjectId format
    if (error.kind === "ObjectId") {
      return res.status(400).json({
        success: false,
        error: "Invalid ID format"
      });
    }
    
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
};

// POST create new record
exports.create = async (req, res) => {
  try {
    const record = await Model.create(req.body);
    
    res.status(201).json({
      success: true,
      data: record
    });
  } catch (error) {
    // Validation error
    if (error.name === "ValidationError") {
      return res.status(400).json({
        success: false,
        error: error.message
      });
    }
    
    // Duplicate key error
    if (error.code === 11000) {
      return res.status(409).json({
        success: false,
        error: "Record already exists"
      });
    }
    
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
};

// PUT/PATCH update record
exports.update = async (req, res) => {
  try {
    const { id } = req.params;
    
    const record = await Model.findByIdAndUpdate(
      id,
      req.body,
      { new: true, runValidators: true }  // Return updated doc, validate
    );
    
    if (!record) {
      return res.status(404).json({
        success: false,
        error: "Record not found"
      });
    }
    
    res.json({
      success: true,
      data: record
    });
  } catch (error) {
    if (error.name === "ValidationError") {
      return res.status(400).json({
        success: false,
        error: error.message
      });
    }
    
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
};

// DELETE record
exports.delete = async (req, res) => {
  try {
    const { id } = req.params;
    
    const record = await Model.findByIdAndDelete(id);
    
    if (!record) {
      return res.status(404).json({
        success: false,
        error: "Record not found"
      });
    }
    
    res.status(204).send();
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
};

// Query with filters
exports.search = async (req, res) => {
  try {
    const { filterField, sortBy, order, limit, offset } = req.query;
    
    // Build query object
    const queryObj = {};
    if (filterField) {
      queryObj.fieldName = filterField;
    }
    
    // Build query
    let query = Model.find(queryObj);
    
    // Sorting
    if (sortBy) {
      const sortOrder = order === "desc" ? "-" : "";
      query = query.sort(`${sortOrder}${sortBy}`);
    }
    
    // Pagination
    if (limit) {
      query = query.limit(parseInt(limit));
    }
    if (offset) {
      query = query.skip(parseInt(offset));
    }
    
    const records = await query;
    const count = await Model.countDocuments(queryObj);
    
    res.json({
      success: true,
      data: records,
      count
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
};
```

**Common Mongoose query methods**:
- `.find()`: Find multiple documents
- `.findOne()`: Find first matching document
- `.findById()`: Find by MongoDB _id
- `.create()`: Insert new document
- `.updateOne()`: Update without returning document
- `.findByIdAndUpdate()`: Update and return document
- `.deleteOne()`: Delete without returning
- `.findByIdAndDelete()`: Delete and return document
- `.countDocuments()`: Count matching documents

---

## 7. Complete Examples

### Example A: Inventory Management System (PostgreSQL)

This example demonstrates a warehouse inventory system tracking products, suppliers, and stock transactions with proper relational design.

#### Database Setup

**File**: `src/config/database.js`

```js
const { Pool } = require("pg");

const pool = new Pool({
  host: process.env.DB_HOST || "localhost",
  port: process.env.DB_PORT || 5432,
  database: process.env.DB_NAME || "inventory_db",
  user: process.env.DB_USER || "postgres",
  password: process.env.DB_PASSWORD,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

pool.on("connect", () => {
  console.log("PostgreSQL connected");
});

pool.on("error", (err) => {
  console.error("PostgreSQL error:", err);
  process.exit(-1);
});

module.exports = pool;
```

#### Schema Definition

**File**: `src/models/init.sql`

```sql
-- Suppliers table
CREATE TABLE suppliers (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  contact_email VARCHAR(255) UNIQUE NOT NULL,
  contact_phone VARCHAR(50),
  address TEXT,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Products table
CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  sku VARCHAR(50) UNIQUE NOT NULL,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  supplier_id INTEGER NOT NULL REFERENCES suppliers(id) ON DELETE RESTRICT,
  unit_price NUMERIC(10, 2) NOT NULL CHECK (unit_price >= 0),
  quantity_in_stock INTEGER DEFAULT 0 CHECK (quantity_in_stock >= 0),
  reorder_level INTEGER DEFAULT 10 CHECK (reorder_level >= 0),
  category VARCHAR(100),
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Stock transactions table (audit trail)
CREATE TABLE stock_transactions (
  id SERIAL PRIMARY KEY,
  product_id INTEGER NOT NULL REFERENCES products(id) ON DELETE CASCADE,
  transaction_type VARCHAR(20) NOT NULL CHECK (transaction_type IN ('IN', 'OUT', 'ADJUSTMENT')),
  quantity INTEGER NOT NULL,
  transaction_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  reference_number VARCHAR(100),
  notes TEXT,
  performed_by VARCHAR(255) NOT NULL
);

-- Indexes for better query performance
CREATE INDEX idx_products_supplier ON products(supplier_id);
CREATE INDEX idx_products_sku ON products(sku);
CREATE INDEX idx_stock_transactions_product ON stock_transactions(product_id);
CREATE INDEX idx_stock_transactions_date ON stock_transactions(transaction_date);

-- Trigger to update updated_at timestamp
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = CURRENT_TIMESTAMP;
  RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_suppliers_updated_at BEFORE UPDATE ON suppliers
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_products_updated_at BEFORE UPDATE ON products
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

**Design decisions**:
- `ON DELETE RESTRICT` for suppliers: Prevents deleting supplier if products exist
- `ON DELETE CASCADE` for stock transactions: Audit trail is removed with product
- `CHECK` constraints: Enforce business rules at database level
- Indexes: Speed up common queries (filtering by supplier, SKU lookup)
- Trigger: Automatically update `updated_at` on row changes

#### Model Layer

**File**: `src/models/Product.model.js`

```js
const pool = require("../config/database");

class Product {
  // Get all products with supplier info
  static async findAll() {
    const result = await pool.query(`
      SELECT 
        p.*,
        s.name as supplier_name,
        s.contact_email as supplier_email
      FROM products p
      LEFT JOIN suppliers s ON p.supplier_id = s.id
      WHERE p.is_active = true
      ORDER BY p.created_at DESC
    `);
    return result.rows;
  }

  // Get single product with full details
  static async findById(id) {
    const result = await pool.query(`
      SELECT 
        p.*,
        s.name as supplier_name,
        s.contact_email as supplier_email,
        s.contact_phone as supplier_phone
      FROM products p
      LEFT JOIN suppliers s ON p.supplier_id = s.id
      WHERE p.id = $1
    `, [id]);
    return result.rows[0];
  }

  // Create new product
  static async create(productData) {
    const { sku, name, description, supplier_id, unit_price, quantity_in_stock, reorder_level, category } = productData;
    
    const result = await pool.query(`
      INSERT INTO products (sku, name, description, supplier_id, unit_price, quantity_in_stock, reorder_level, category)
      VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
      RETURNING *
    `, [sku, name, description, supplier_id, unit_price, quantity_in_stock, reorder_level, category]);
    
    return result.rows[0];
  }

  // Update stock with transaction recording
  static async updateStock(productId, quantity, transactionType, performedBy, notes = null) {
    const client = await pool.connect();
    
    try {
      await client.query("BEGIN");
      
      // Update product stock
      const updateQuery = transactionType === "IN" 
        ? "UPDATE products SET quantity_in_stock = quantity_in_stock + $1 WHERE id = $2 RETURNING *"
        : "UPDATE products SET quantity_in_stock = quantity_in_stock - $1 WHERE id = $2 RETURNING *";
      
      const productResult = await client.query(updateQuery, [quantity, productId]);
      
      if (productResult.rows.length === 0) {
        throw new Error("Product not found");
      }
      
      // Record transaction
      await client.query(`
        INSERT INTO stock_transactions (product_id, transaction_type, quantity, performed_by, notes)
        VALUES ($1, $2, $3, $4, $5)
      `, [productId, transactionType, quantity, performedBy, notes]);
      
      await client.query("COMMIT");
      return productResult.rows[0];
      
    } catch (error) {
      await client.query("ROLLBACK");
      throw error;
    } finally {
      client.release();
    }
  }

  // Get products below reorder level
  static async getLowStock() {
    const result = await pool.query(`
      SELECT 
        p.*,
        s.name as supplier_name,
        s.contact_email as supplier_email
      FROM products p
      LEFT JOIN suppliers s ON p.supplier_id = s.id
      WHERE p.quantity_in_stock <= p.reorder_level
        AND p.is_active = true
      ORDER BY p.quantity_in_stock ASC
    `);
    return result.rows;
  }

  // Get stock transaction history for a product
  static async getTransactionHistory(productId) {
    const result = await pool.query(`
      SELECT * FROM stock_transactions
      WHERE product_id = $1
      ORDER BY transaction_date DESC
      LIMIT 50
    `, [productId]);
    return result.rows;
  }
}

module.exports = Product;
```

**Why transactions?**: Stock updates must be atomic—if updating quantity fails, the transaction record shouldn't be created. `BEGIN`/`COMMIT`/`ROLLBACK` ensure both operations succeed or both fail.

#### Controller Implementation

**File**: `src/controllers/product.controller.js`

```js
const Product = require("../models/Product.model");

exports.getAllProducts = async (req, res) => {
  try {
    const products = await Product.findAll();
    
    res.json({
      success: true,
      count: products.length,
      data: products
    });
  } catch (error) {
    console.error("Error fetching products:", error);
    res.status(500).json({
      success: false,
      error: "Failed to fetch products"
    });
  }
};

exports.getProductById = async (req, res) => {
  try {
    const { id } = req.params;
    const product = await Product.findById(id);
    
    if (!product) {
      return res.status(404).json({
        success: false,
        error: "Product not found"
      });
    }
    
    res.json({
      success: true,
      data: product
    });
  } catch (error) {
    console.error("Error fetching product:", error);
    res.status(500).json({
      success: false,
      error: "Failed to fetch product"
    });
  }
};

exports.createProduct = async (req, res) => {
  try {
    const product = await Product.create(req.body);
    
    res.status(201).json({
      success: true,
      data: product,
      message: "Product created successfully"
    });
  } catch (error) {
    console.error("Error creating product:", error);
    
    // Handle duplicate SKU
    if (error.code === "23505") {
      return res.status(409).json({
        success: false,
        error: "Product with this SKU already exists"
      });
    }
    
    // Handle foreign key violation (invalid supplier_id)
    if (error.code === "23503") {
      return res.status(400).json({
        success: false,
        error: "Invalid supplier ID"
      });
    }
    
    res.status(500).json({
      success: false,
      error: "Failed to create product"
    });
  }
};

exports.updateStock = async (req, res) => {
  try {
    const { id } = req.params;
    const { quantity, transaction_type, notes } = req.body;
    const performedBy = req.user.email; // From auth middleware
    
    // Validate transaction type
    if (!["IN", "OUT", "ADJUSTMENT"].includes(transaction_type)) {
      return res.status(400).json({
        success: false,
        error: "Invalid transaction type. Must be IN, OUT, or ADJUSTMENT"
      });
    }
    
    // Validate quantity
    if (!quantity || quantity <= 0) {
      return res.status(400).json({
        success: false,
        error: "Quantity must be a positive number"
      });
    }
    
    const product = await Product.updateStock(id, quantity, transaction_type, performedBy, notes);
    
    res.json({
      success: true,
      data: product,
      message: "Stock updated successfully"
    });
  } catch (error) {
    console.error("Error updating stock:", error);
    
    if (error.message === "Product not found") {
      return res.status(404).json({
        success: false,
        error: error.message
      });
    }
    
    res.status(500).json({
      success: false,
      error: "Failed to update stock"
    });
  }
};

exports.getLowStock = async (req, res) => {
  try {
    const products = await Product.getLowStock();
    
    res.json({
      success: true,
      count: products.length,
      data: products,
      message: products.length > 0 ? "Products below reorder level found" : "All products adequately stocked"
    });
  } catch (error) {
    console.error("Error fetching low stock products:", error);
    res.status(500).json({
      success: false,
      error: "Failed to fetch low stock products"
    });
  }
};

exports.getTransactionHistory = async (req, res) => {
  try {
    const { id } = req.params;
    const transactions = await Product.getTransactionHistory(id);
    
    res.json({
      success: true,
      count: transactions.length,
      data: transactions
    });
  } catch (error) {
    console.error("Error fetching transaction history:", error);
    res.status(500).json({
      success: false,
      error: "Failed to fetch transaction history"
    });
  }
};
```

**Real-world considerations**:
- Validation before database operations (prevents unnecessary queries)
- Specific error handling for different PostgreSQL error codes
- Transaction history for audit compliance
- Low stock alerts for inventory management

---

### Example B: Content Management System (MongoDB)

This example demonstrates a blog/article platform with flexible content types, author management, and revision tracking.

#### Database Setup

**File**: `src/config/database.js`

```js
const mongoose = require("mongoose");

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGODB_URI, {
      maxPoolSize: 10,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,
    });

    console.log(`MongoDB connected: ${conn.connection.host}`);

    mongoose.connection.on("error", (err) => {
      console.error("MongoDB error:", err);
    });

    mongoose.connection.on("disconnected", () => {
      console.log("MongoDB disconnected");
    });

    process.on("SIGINT", async () => {
      await mongoose.connection.close();
      console.log("MongoDB connection closed");
      process.exit(0);
    });

  } catch (error) {
    console.error("MongoDB connection error:", error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

#### Schema Definitions

**File**: `src/models/Author.model.js`

```js
const mongoose = require("mongoose");

const authorSchema = new mongoose.Schema({
  username: {
    type: String,
    required: [true, "Username is required"],
    unique: true,
    trim: true,
    minlength: [3, "Username must be at least 3 characters"],
    maxlength: [30, "Username cannot exceed 30 characters"],
    match: [/^[a-zA-Z0-9_]+$/, "Username can only contain letters, numbers, and underscores"]
  },
  email: {
    type: String,
    required: [true, "Email is required"],
    unique: true,
    lowercase: true,
    trim: true,
    match: [/^\S+@\S+\.\S+$/, "Please provide a valid email"]
  },
  fullName: {
    type: String,
    required: [true, "Full name is required"],
    trim: true
  },
  bio: {
    type: String,
    maxlength: [500, "Bio cannot exceed 500 characters"]
  },
  profilePicture: {
    type: String,
    default: null
  },
  socialLinks: {
    twitter: String,
    linkedin: String,
    website: String
  },
  isActive: {
    type: Boolean,
    default: true
  },
  role: {
    type: String,
    enum: ["author", "editor", "admin"],
    default: "author"
  },
  articlesPublished: {
    type: Number,
    default: 0
  }
}, {
  timestamps: true
});

// Virtual for author URL
authorSchema.virtual("profileUrl").get(function() {
  return `/authors/${this.username}`;
});

// Instance method - Check if author can publish
authorSchema.methods.canPublish = function() {
  return this.isActive && ["author", "editor", "admin"].includes(this.role);
};

// Static method - Find active authors
authorSchema.statics.findActive = function() {
  return this.find({ isActive: true });
};

const Author = mongoose.model("Author", authorSchema);
module.exports = Author;
```

**File**: `src/models/Article.model.js`

```js
const mongoose = require("mongoose");

const revisionSchema = new mongoose.Schema({
  content: {
    type: String,
    required: true
  },
  modifiedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "Author",
    required: true
  },
  modifiedAt: {
    type: Date,
    default: Date.now
  },
  changeNote: String
});

const articleSchema = new mongoose.Schema({
  title: {
    type: String,
    required: [true, "Title is required"],
    trim: true,
    minlength: [10, "Title must be at least 10 characters"],
    maxlength: [200, "Title cannot exceed 200 characters"]
  },
  slug: {
    type: String,
    required: true,
    unique: true,
    lowercase: true
  },
  content: {
    type: String,
    required: [true, "Content is required"],
    minlength: [100, "Content must be at least 100 characters"]
  },
  excerpt: {
    type: String,
    maxlength: [300, "Excerpt cannot exceed 300 characters"]
  },
  author: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "Author",
    required: [true, "Author is required"]
  },
  coAuthors: [{
    type: mongoose.Schema.Types.ObjectId,
    ref: "Author"
  }],
  status: {
    type: String,
    enum: ["draft", "published", "archived"],
    default: "draft"
  },
  publishedAt: {
    type: Date,
    default: null
  },
  tags: [{
    type: String,
    trim: true,
    lowercase: true
  }],
  categories: [{
    type: String,
    trim: true
  }],
  featuredImage: {
    url: String,
    alt: String
  },
  metadata: {
    readingTimeMinutes: Number,
    wordCount: Number,
    views: {
      type: Number,
      default: 0
    }
  },
  seo: {
    metaTitle: String,
    metaDescription: String,
    keywords: [String]
  },
  revisions: [revisionSchema],
  isDeleted: {
    type: Boolean,
    default: false
  }
}, {
  timestamps: true,
  toJSON: { virtuals: true },
  toObject: { virtuals: true }
});

// Indexes for better query performance
articleSchema.index({ slug: 1 });
articleSchema.index({ author: 1, status: 1 });
articleSchema.index({ tags: 1 });
articleSchema.index({ publishedAt: -1 });
articleSchema.index({ "metadata.views": -1 });

// Virtual for article URL
articleSchema.virtual("url").get(function() {
  return `/articles/${this.slug}`;
});

// Pre-save middleware - Calculate reading time and word count
articleSchema.pre("save", function(next) {
  if (this.isModified("content")) {
    const wordCount = this.content.split(/\s+/).length;
    const readingTimeMinutes = Math.ceil(wordCount / 200); // Average reading speed
    
    this.metadata.wordCount = wordCount;
    this.metadata.readingTimeMinutes = readingTimeMinutes;
    
    // Auto-generate excerpt if not provided
    if (!this.excerpt) {
      this.excerpt = this.content.substring(0, 200) + "...";
    }
  }
  next();
});

// Pre-save middleware - Record revision
articleSchema.pre("save", function(next) {
  // Only record revision if content changed and article already exists
  if (this.isModified("content") && !this.isNew) {
    this.revisions.push({
      content: this.content,
      modifiedBy: this._modifiedBy, // Must be set before save
      changeNote: this._changeNote
    });
  }
  next();
});

// Instance method - Publish article
articleSchema.methods.publish = async function() {
  this.status = "published";
  this.publishedAt = new Date();
  await this.save();
  
  // Increment author's article count
  await mongoose.model("Author").findByIdAndUpdate(this.author, {
    $inc: { articlesPublished: 1 }
  });
};

// Instance method - Increment view count
articleSchema.methods.incrementViews = function() {
  return this.updateOne({ $inc: { "metadata.views": 1 } });
};

// Static method - Find published articles
articleSchema.statics.findPublished = function(options = {}) {
  const { limit = 10, skip = 0, tags, category } = options;
  
  const query = { status: "published", isDeleted: false };
  
  if (tags && tags.length > 0) {
    query.tags = { $in: tags };
  }
  
  if (category) {
    query.categories = category;
  }
  
  return this.find(query)
    .populate("author", "username fullName profilePicture")
    .populate("coAuthors", "username fullName")
    .sort({ publishedAt: -1 })
    .limit(limit)
    .skip(skip);
};

// Static method - Get trending articles
articleSchema.statics.getTrending = function(limit = 10) {
  return this.find({
    status: "published",
    isDeleted: false,
    publishedAt: { $gte: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000) } // Last 7 days
  })
    .sort({ "metadata.views": -1 })
    .limit(limit)
    .populate("author", "username fullName profilePicture");
};

const Article = mongoose.model("Article", articleSchema);
module.exports = Article;
```

**Design decisions**:
- **Embedded revisions**: Revision history is always accessed with article, so embed it
- **Referenced authors**: Authors exist independently and are shared across articles
- **Indexes**: Speed up common queries (by slug, author+status, trending)
- **Middleware**: Automatically calculate metadata and track revisions
- **Virtual fields**: Computed properties that don't need database storage
- **Instance/Static methods**: Encapsulate business logic in model

#### Controller Implementation

**File**: `src/controllers/article.controller.js`

```js
const Article = require("../models/Article.model");
const Author = require("../models/Author.model");

exports.getAllArticles = async (req, res) => {
  try {
    const { status, tags, category, limit = 20, page = 1 } = req.query;
    
    const query = { isDeleted: false };
    
    if (status) {
      query.status = status;
    }
    
    if (tags) {
      query.tags = { $in: tags.split(",") };
    }
    
    if (category) {
      query.categories = category;
    }
    
    const skip = (page - 1) * limit;
    
    const articles = await Article.find(query)
      .populate("author", "username fullName profilePicture")
      .populate("coAuthors", "username fullName")
      .sort({ createdAt: -1 })
      .limit(parseInt(limit))
      .skip(skip);
    
    const total = await Article.countDocuments(query);
    
    res.json({
      success: true,
      data: articles,
      pagination: {
        total,
        page: parseInt(page),
        limit: parseInt(limit),
        pages: Math.ceil(total / limit)
      }
    });
  } catch (error) {
    console.error("Error fetching articles:", error);
    res.status(500).json({
      success: false,
      error: "Failed to fetch articles"
    });
  }
};

exports.getArticleBySlug = async (req, res) => {
  try {
    const { slug } = req.params;
    
    const article = await Article.findOne({ slug, isDeleted: false })
      .populate("author", "username fullName bio profilePicture socialLinks")
      .populate("coAuthors", "username fullName profilePicture");
    
    if (!article) {
      return res.status(404).json({
        success: false,
        error: "Article not found"
      });
    }
    
    // Increment view count (fire and forget - don't await)
    article.incrementViews().catch(err => console.error("Failed to increment views:", err));
    
    res.json({
      success: true,
      data: article
    });
  } catch (error) {
    console.error("Error fetching article:", error);
    res.status(500).json({
      success: false,
      error: "Failed to fetch article"
    });
  }
};

exports.createArticle = async (req, res) => {
  try {
    const { title, content, author, coAuthors, tags, categories, featuredImage, seo } = req.body;
    
    // Generate slug from title
    const slug = title
      .toLowerCase()
      .replace(/[^a-z0-9]+/g, "-")
      .replace(/^-|-$/g, "");
    
    // Verify author exists
    const authorExists = await Author.findById(author);
    if (!authorExists) {
      return res.status(400).json({
        success: false,
        error: "Author not found"
      });
    }
    
    // Verify author can publish
    if (!authorExists.canPublish()) {
      return res.status(403).json({
        success: false,
        error: "Author does not have permission to publish"
      });
    }
    
    const article = await Article.create({
      title,
      slug,
      content,
      author,
      coAuthors,
      tags,
      categories,
      featuredImage,
      seo
    });
    
    // Populate author info before returning
    await article.populate("author", "username fullName profilePicture");
    
    res.status(201).json({
      success: true,
      data: article,
      message: "Article created successfully"
    });
  } catch (error) {
    console.error("Error creating article:", error);
    
    // Handle duplicate slug
    if (error.code === 11000) {
      return res.status(409).json({
        success: false,
        error: "Article with similar title already exists"
      });
    }
    
    // Handle validation errors
    if (error.name === "ValidationError") {
      const messages = Object.values(error.errors).map(err => err.message);
      return res.status(400).json({
        success: false,
        error: messages.join(", ")
      });
    }
    
    res.status(500).json({
      success: false,
      error: "Failed to create article"
    });
  }
};

exports.updateArticle = async (req, res) => {
  try {
    const { slug } = req.params;
    const { content, changeNote, ...updateData } = req.body;
    const modifiedBy = req.user._id; // From auth middleware
    
    const article = await Article.findOne({ slug, isDeleted: false });
    
    if (!article) {
      return res.status(404).json({
        success: false,
        error: "Article not found"
      });
    }
    
    // Update fields
    Object.assign(article, updateData);
    
    // If content is being updated, set revision metadata
    if (content && content !== article.content) {
      article.content = content;
      article._modifiedBy = modifiedBy;
      article._changeNote = changeNote;
    }
    
    await article.save();
    await article.populate("author", "username fullName profilePicture");
    
    res.json({
      success: true,
      data: article,
      message: "Article updated successfully"
    });
  } catch (error) {
    console.error("Error updating article:", error);
    
    if (error.name === "ValidationError") {
      const messages = Object.values(error.errors).map(err => err.message);
      return res.status(400).json({
        success: false,
        error: messages.join(", ")
      });
    }
    
    res.status(500).json({
      success: false,
      error: "Failed to update article"
    });
  }
};

exports.publishArticle = async (req, res) => {
  try {
    const { slug } = req.params;
    
    const article = await Article.findOne({ slug, isDeleted: false });
    
    if (!article) {
      return res.status(404).json({
        success: false,
        error: "Article not found"
      });
    }
    
    if (article.status === "published") {
      return res.status(400).json({
        success: false,
        error: "Article is already published"
      });
    }
    
    await article.publish();
    await article.populate("author", "username fullName profilePicture");
    
    res.json({
      success: true,
      data: article,
      message: "Article published successfully"
    });
  } catch (error) {
    console.error("Error publishing article:", error);
    res.status(500).json({
      success: false,
      error: "Failed to publish article"
    });
  }
};

exports.deleteArticle = async (req, res) => {
  try {
    const { slug } = req.params;
    
    // Soft delete
    const article = await Article.findOneAndUpdate(
      { slug, isDeleted: false },
      { isDeleted: true },
      { new: true }
    );
    
    if (!article) {
      return res.status(404).json({
        success: false,
        error: "Article not found"
      });
    }
    
    res.status(204).send();
  } catch (error) {
    console.error("Error deleting article:", error);
    res.status(500).json({
      success: false,
      error: "Failed to delete article"
    });
  }
};

exports.getTrendingArticles = async (req, res) => {
  try {
    const { limit = 10 } = req.query;
    
    const articles = await Article.getTrending(parseInt(limit));
    
    res.json({
      success: true,
      count: articles.length,
      data: articles
    });
  } catch (error) {
    console.error("Error fetching trending articles:", error);
    res.status(500).json({
      success: false,
      error: "Failed to fetch trending articles"
    });
  }
};

exports.getArticleRevisions = async (req, res) => {
  try {
    const { slug } = req.params;
    
    const article = await Article.findOne({ slug, isDeleted: false })
      .select("revisions")
      .populate("revisions.modifiedBy", "username fullName");
    
    if (!article) {
      return res.status(404).json({
        success: false,
        error: "Article not found"
      });
    }
    
    res.json({
      success: true,
      count: article.revisions.length,
      data: article.revisions
    });
  } catch (error) {
    console.error("Error fetching revisions:", error);
    res.status(500).json({
      success: false,
      error: "Failed to fetch revisions"
    });
  }
};
```

**Real-world features demonstrated**:
- **Soft delete**: Mark as deleted instead of removing (data retention)
- **Pagination**: Handle large result sets efficiently
- **Slug-based URLs**: SEO-friendly identifiers
- **View tracking**: Analytics for content performance
- **Revision history**: Content audit trail with change notes
- **Permission checks**: Verify author can perform action
- **Auto-calculations**: Reading time and word count computed automatically
- **Populate optimization**: Only select needed fields from referenced documents

---

## Summary

You now understand database integration for both PostgreSQL and MongoDB:

**PostgreSQL strengths**:
- ✅ Strict schema enforcement and data integrity
- ✅ Complex joins and relationships
- ✅ ACID transactions for critical operations
- ✅ Powerful querying with SQL
- ✅ Best for: Structured data, financial systems, complex relationships

**MongoDB strengths**:
- ✅ Flexible schema for evolving data
- ✅ Fast document-based operations
- ✅ Easy horizontal scaling
- ✅ Natural fit for hierarchical data
- ✅ Best for: Content management, real-time data, rapid development

**Key takeaways**:
1. **Choose based on data structure**: Relational vs document-oriented
2. **Connection pooling**: Essential for production performance
3. **Schema design**: PostgreSQL requires upfront planning, MongoDB allows evolution
4. **Relationships**: Foreign keys (PostgreSQL) vs references/embedding (MongoDB)
5. **Transactions**: Native in PostgreSQL, document-level in MongoDB
6. **Model layer**: Abstracts database operations from controllers
7. **Error handling**: Database-specific error codes and validation

**Next steps**:
- Add data validation middleware
- Implement database migrations
- Set up database backups
- Add connection retry logic
- Optimize queries with explain/analyze
- Set up monitoring and logging
