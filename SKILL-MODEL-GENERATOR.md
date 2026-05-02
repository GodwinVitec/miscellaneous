File name: SKILL-MODEL-GENERATOR.md
Language: 
---
name: model
description: Generate MongoDB Mongoose models/schemas for Node.js Express backend following project conventions and best practices
user_invocable: true
command: model
---

# MongoDB Model Generator

Generate Mongoose schemas and models for MongoDB collections in a Node.js + Express backend, following project conventions and layered architecture.

## Pre-Flight Checklist (STOP GATE)

**NEVER work on assumptions. NEVER skip this checklist.**

Before writing ANY code, ensure these core details are known. Only ask the user about what's genuinely unclear — derive everything else from the guidelines in this skill file.

### Core Checklist (ask the user only for these)

```
- [ ] What is the entity/collection? (name + what data it represents)
- [ ] What fields does it have? (names, types, constraints, relationships)
- [ ] Any special business rules? (enums, defaults, validations, indexing strategies)
```

**That's it.** Only these 3 items need explicit user input. Everything else — schema structure, validation rules, indexing, timestamps, relationships, instance methods, static methods — is generated automatically by following the rules in this skill file.

### What you derive (do NOT ask the user about these)

- Schema field types — map user descriptions to Mongoose types (String, Number, Boolean, Date, ObjectId, Array, etc.)
- Required vs. optional — infer from context (e.g., email is required, phone is optional)
- Validation rules — apply sensible constraints (min/max length, enums, custom validators)
- Indexes — add indexes for frequently queried fields (searches, filters, sorting)
- Timestamps — add `createdAt` and `updatedAt` by default
- Relationships — use `ref` for document relationships, populate helpers
- Instance methods — add helpers like `toJSON()` for response serialization
- Static methods — add query helpers like `findActive()`, `countByStatus()`, etc.
- Error handling — follow standard patterns (validation errors, not-found errors)

### Jira Ticket Integration

If the user provides a Jira ticket key (e.g., `PROJ-123`) or URL (e.g., `https://vitecgmbh.atlassian.net/browse/PROJ-123`):

1. **Extract the ticket key** from the input — parse it from the URL if a full URL is provided, or use the key directly
2. **Fetch the ticket** using the Atlassian MCP tool:
   ```
   mcp__claude_ai_Atlassian__getJiraIssue with cloudId: "vitecgmbh.atlassian.net", issueKey: "<TICKET-KEY>"
   ```
3. **Extract from the ticket**: summary, description, acceptance criteria, subtasks, linked issues, custom fields (data model, entity fields, business rules)
4. **Use the ticket content** as the model specification — acceptance criteria and descriptions map directly to what schema fields and validations are needed
5. Run the core checklist against the extracted ticket content — if the 3 core items are still unclear, ask only about those gaps

### When to ask vs. when to proceed

**STOP and ask** — entity is too vague to identify required fields:
- `/model product` — Ask: What fields does a product have (name, price, category, etc.)? What types?
- `/model user` — Ask: What profile fields are needed? Authentication? Preferences?

**Proceed to planning** — core checklist is clear, derive the rest:
- `/model product with name (string), price (number), category (string), description (string), inStock (boolean), image (string), created by (user reference)`
- `/model order with items (array of products), status (pending/processing/shipped/delivered), total (number), buyer (user reference), shipping address (object)`
- `PROJ-123` (Jira ticket with clear entity definition and field list after fetching)

**Rules:**
- Ask at most 2-3 focused questions, not a wall of 8+ questions
- Present questions as a short numbered list and WAIT for answers
- Do not mix questions with partial code generation
- If in doubt about field types or constraints, ask. For everything else (indexes, timestamps, methods), follow the guidelines and decide

---

## Existing Model Detection

Before generating anything, check if a model with the same name already exists in the codebase. Search for:
- `api/src/models/<EntityName>.js` (Mongoose model file)

### If the model already exists:

**1. Determine the type of request** — compare what the user is asking for against what already exists:

**Change Request (modify/enhance existing model):**
The user wants to add fields, change types, add validations, add indexes, or enhance the existing model.

- Read and understand the existing model completely
- Make changes incrementally — modify the existing file, don't recreate from scratch
- **NEVER break existing functionality** — all current fields and behavior must continue to work
- If new fields are added, use sensible defaults or make them optional
- If field types change, ensure backward compatibility (e.g., allow both old and new formats during migration period)
- If model is used in routes/controllers, verify changes don't break existing API contracts
- Add a migration comment in the code documenting what changed and why

**Complete Redefinition (entirely different entity with the same name):**
The user's description is fundamentally different from what exists — different purpose, completely different field set.

- **STOP and warn the user** before proceeding. Present:
  ```
  WARNING: Model "<EntityName>" already exists with a different definition.

  Existing model:
  - <summarize current fields: names, types, relationships>
  - <summarize current indexes and methods>
  - <list validation rules>

  Your request describes a completely different entity. Proceeding will REPLACE:
  - <list of files/routes that depend on this model>
  - <list of API endpoints that will be affected>

  Do you want to:
  1. Replace the existing model entirely (destructive — existing data model will change)
  2. Rename the new entity to avoid conflict
  3. Cancel and modify the existing model instead
  ```
- **Wait for explicit confirmation** — do NOT proceed without the user choosing an option
- If the user chooses to replace, create a backup list of what was changed in the output summary

### If the model does NOT exist:

Proceed normally with the pre-flight checklist and planning steps.

---

## Input

The user provides a model/entity description. Examples:
- `/model product with name, price, category, description`
- `/model order with items array, status enum, buyer reference to user`
- `/model blog-post with title, content, author reference, published boolean, tags array`

Parse the description to determine:
- **Entity name** (PascalCase for model class, kebab-case for file)
- **Required fields**: names, types, constraints, relationships, defaults
- **Indexes**: fields that need indexing for queries and sorting
- **Methods**: instance and static helpers for common queries

---

## Backend Architecture

**The backend is a Node.js + Express application** located at `api/` directory at the project root. It follows a layered architecture with Models, Repositories, Services, and Controllers.

```
project-root/
  api/                         (Express backend, port 5000)
    src/
      models/                  (Mongoose schemas - THIS SKILL GENERATES HERE)
      repositories/            (Data access layer - use models)
      services/                (Business logic layer - use repositories)
      controllers/             (Request handling layer - use services)
      routes/                  (Route mapping - use controllers)
      middlewares/             (Cross-cutting concerns - auth, validation, error handling)
      validations/             (Joi validation schemas)
      config/                  (Configuration - env, db, logger)
      utils/                   (Shared utilities)
    app.js                     (Express app setup)
    server.js                  (HTTP server entry point)
    .env                       (Environment variables)
```

### Database Connection

- MongoDB connection is configured in `api/src/config/db.js` using `MONGO_URI` from environment variables
- Models are Mongoose schemas that connect to MongoDB via Mongoose
- All data access must go through models and repositories (never raw MongoDB in controllers or services)

---

## Mongoose Model Rules

All Mongoose models MUST follow these rules. **Fetch and read the Confluence reference** before generating models:

> https://vitecgmbh.atlassian.net/wiki/spaces/TWD/pages/2737045509/BE+-+Project+Structure+-+NodeJS

Use the Atlassian MCP tool (`mcp__claude_ai_Atlassian__getConfluencePage` with `cloudId: "vitecgmbh.atlassian.net"`, `pageId: "2737045509"`, `contentFormat: "markdown"`) to fetch the page content. This page defines base patterns, error handling, and best practices for BE code structure.

### Schema Design Rules

- **File location**: `api/src/models/<EntityName>.js` (PascalCase filename matching the model name)
- **Export pattern**: `module.exports = mongoose.model('<EntityName>', <entityName>Schema);`
- **Schema naming**: `const <entityName>Schema = new Schema({ ... });`
- **Timestamps**: Add `timestamps: true` to automatically create `createdAt` and `updatedAt` fields
- **Field naming**: Use camelCase for all field names
- **No business logic in schemas** — schemas define structure only. Logic goes in instance methods, static methods, or repositories

### Field Types and Constraints

| Type | Mongoose | Usage | Constraints | Example |
|------|----------|-------|-------------|---------|
| String | `String` | Text fields (names, emails, descriptions) | `required`, `minlength`, `maxlength`, `enum`, `trim`, `lowercase`, `uppercase`, `match` (regex) | `{ name: { type: String, required: true, trim: true } }` |
| Number | `Number` | Numeric fields (prices, quantities, ratings) | `required`, `min`, `max` | `{ price: { type: Number, required: true, min: 0 } }` |
| Boolean | `Boolean` | True/false flags | `default` | `{ isActive: { type: Boolean, default: true } }` |
| Date | `Date` | Timestamps and dates | `default: Date.now` | `{ publishedAt: { type: Date, default: null } }` |
| ObjectId | `Schema.Types.ObjectId`, `ref: '<Model>'` | References to other documents | `ref`, optional or required | `{ userId: { type: Schema.Types.ObjectId, ref: 'User', required: true } }` |
| Array | `[Type]` or `{ type: [Type] }` | Collections of values or objects | `default: []`, specify element type | `{ tags: [String]` or `items: [{ name: String, qty: Number }] }` |
| Object/Nested | `{ field1: Type, field2: Type }` | Embedded documents (non-normalized) | Use for frequently-accessed related data | `{ address: { street: String, city: String, zip: String } }` |
| Enum | `{ type: String, enum: ['value1', 'value2'] }` | Restricted set of values | Define all allowed values | `{ status: { type: String, enum: ['active', 'inactive', 'pending'] } }` |

### Relationships

**1-to-N (One User, Many Orders) — Use References:**
```javascript
const orderSchema = new Schema({
  userId: { type: Schema.Types.ObjectId, ref: 'User', required: true },
  items: [{ ... }],
  total: Number,
  status: String
});
```
Query with `.populate('userId')` to get user details with order.

**N-to-N (Many Users, Many Groups) — Use References:**
```javascript
const userSchema = new Schema({
  name: String,
  groupIds: [{ type: Schema.Types.ObjectId, ref: 'Group' }]
});

const groupSchema = new Schema({
  name: String,
  userIds: [{ type: Schema.Types.ObjectId, ref: 'User' }]
});
```
Query with `.populate('groupIds')` or `.populate('userIds')`.

**Embedded Documents (frequently accessed, not shared) — Use Nesting:**
```javascript
const orderSchema = new Schema({
  shippingAddress: {
    street: String,
    city: String,
    zip: String,
    country: String
  }
});
```
Access as `order.shippingAddress.city`.

**Rules for relationships:**
- Use `ref` for 1-to-N and N-to-N relationships (normalized)
- Use nested documents for frequently-accessed related data that won't be updated separately
- Always add `.select()` or `.populate()` in queries to include related data when needed
- Never store duplicate data — either reference it or embed it, never both

### Indexing Strategy

Add indexes to fields that are frequently queried, filtered, or sorted:

```javascript
const userSchema = new Schema({
  email: { type: String, required: true, index: true, unique: true },
  username: { type: String, required: true, index: true },
  status: { type: String, enum: ['active', 'inactive'], index: true },
  createdAt: { type: Date, default: Date.now, index: true }
});

// Compound indexes for common queries
userSchema.index({ status: 1, createdAt: -1 });
```

**Index rules:**
- Add `index: true` for fields used in WHERE clauses, SORT, or SEARCH queries
- Add `unique: true` for email, username, or other unique identifiers
- Use compound indexes (`.index({ field1: 1, field2: -1 })`) for multi-field queries (e.g., filter by status, sort by date)
- Use `-1` for descending order (usually for dates to get newest first)
- Only add indexes for truly common queries — avoid over-indexing (slows writes)

### Instance Methods

Add helper methods to the schema for common operations on individual documents:

```javascript
// Serialization
userSchema.methods.toJSON = function() {
  const obj = this.toObject();
  delete obj.password;  // Never expose password in responses
  return obj;
};

// Validation helpers
userSchema.methods.isActive = function() {
  return this.status === 'active';
};

// Status checks
orderSchema.methods.canBeCancelled = function() {
  return ['pending', 'processing'].includes(this.status);
};
```

**Rules for instance methods:**
- Always include `.toJSON()` to safely serialize documents for API responses (remove sensitive fields)
- Add helpers for common checks and transformations on a single document
- Never include DB queries in methods — those belong in static methods or repositories

### Static Methods

Add query helpers as static methods for common database operations:

```javascript
// Find active users
userSchema.statics.findActive = function() {
  return this.find({ status: 'active' });
};

// Find by email
userSchema.statics.findByEmail = async function(email) {
  return this.findOne({ email: email.toLowerCase() });
};

// Count by status
userSchema.statics.countByStatus = async function(status) {
  return this.countDocuments({ status });
};
```

**Rules for static methods:**
- Use for frequently-called queries to reduce boilerplate in repositories
- Keep them focused — one operation per method
- Return the query (don't `.exec()` in the static method) so calling code can chain `.lean()`, `.select()`, etc.

### Middleware (Hooks)

Add pre/post hooks for automatic data processing and validation:

```javascript
// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  try {
    const bcrypt = require('bcryptjs');
    this.password = await bcrypt.hash(this.password, 10);
    next();
  } catch (err) {
    next(err);
  }
});

// Lowercase email before saving
userSchema.pre('save', function(next) {
  if (this.isModified('email')) {
    this.email = this.email.toLowerCase();
  }
  next();
});

// Remove sensitive fields after finding
userSchema.post('find', function(docs) {
  docs.forEach(doc => {
    if (doc.toJSON) doc.toJSON();
  });
});
```

**Rules for hooks:**
- Use `pre('save')` for validation, hashing, normalization before saves
- Use `post('find')` or `post('findOne')` to clean up responses (remove sensitive fields)
- Always call `next(err)` on error or `next()` on success
- Keep hooks focused — one operation per hook

### Example: Complete User Model

```javascript
const mongoose = require('mongoose');
const Schema = mongoose.Schema;

const userSchema = new Schema(
  {
    email: {
      type: String,
      required: [true, 'Email is required'],
      unique: true,
      lowercase: true,
      trim: true,
      match: [/^\w+([.-]?\w+)*@\w+([.-]?\w+)*(\.\w{2,3})+$/, 'Email format is invalid']
    },
    username: {
      type: String,
      required: true,
      unique: true,
      minlength: 3,
      maxlength: 50,
      trim: true
    },
    password: {
      type: String,
      required: true,
      minlength: 8,
      select: false  // Don't include password in queries by default
    },
    firstName: {
      type: String,
      trim: true
    },
    lastName: {
      type: String,
      trim: true
    },
    profile: {
      bio: String,
      avatar: String,
      phone: String
    },
    role: {
      type: String,
      enum: ['user', 'admin', 'moderator'],
      default: 'user'
    },
    status: {
      type: String,
      enum: ['active', 'inactive', 'suspended'],
      default: 'active',
      index: true
    },
    preferences: {
      newsletter: { type: Boolean, default: false },
      notifications: { type: Boolean, default: true },
      theme: { type: String, enum: ['light', 'dark'], default: 'light' }
    },
    lastLoginAt: Date
  },
  { 
    timestamps: true,  // Adds createdAt, updatedAt
    collection: 'users'  // Explicit collection name
  }
);

// Indexes
userSchema.index({ email: 1, status: 1 });
userSchema.index({ createdAt: -1 });

// Instance Methods
userSchema.methods.toJSON = function() {
  const obj = this.toObject();
  delete obj.password;
  return obj;
};

userSchema.methods.isAdmin = function() {
  return this.role === 'admin';
};

// Static Methods
userSchema.statics.findByEmail = function(email) {
  return this.findOne({ email: email.toLowerCase() });
};

userSchema.statics.findActive = function() {
  return this.find({ status: 'active' });
};

// Hooks
userSchema.pre('save', function(next) {
  if (this.isModified('email')) {
    this.email = this.email.toLowerCase();
  }
  next();
});

module.exports = mongoose.model('User', userSchema);
```

---

## Steps

### 0. Project setup (run ONCE if not already set up)

Before generating any model, check if the backend infrastructure is in place. If any of the following are missing, set them up first:

#### Check and install BE dependencies:

```bash
# Check if BE dependencies are installed
npm ls express mongoose dotenv joi jsonwebtoken
```

If missing, install into the **root** `package.json` (shared `node_modules/` for both FE and BE):

1. **Install BE packages** from the project root:
   ```bash
   npm install express mongoose dotenv joi jsonwebtoken bcryptjs cors
   npm install -D jest supertest mongodb-memory-server nodemon
   ```

2. **Create BE directory structure** — `api/` at the project root (no separate `package.json`):
   ```
   api/
     src/
       models/                  (Models live here)
       routes/
       controllers/
       services/
       repositories/
       middlewares/
       validations/
       config/
       utils/
     __tests__/
       setup.js
       teardown.js
     app.js
     server.js
     .env
     .env.example
   ```

   **Note:** `api/` does NOT have its own `package.json` or `node_modules/`. It uses the root project's `package.json` and `node_modules/`. All dependencies are managed from the project root.

3. **Create base files** (fetch patterns from Confluence page):
   - `api/app.js` — Express app setup with CORS, middleware, route mounting, and error handler
   - `api/server.js` — HTTP server entry point (listens on its own port)
   - `api/src/config/env.js` — environment variables (PORT, MONGO_URI, JWT_SECRET)
   - `api/src/config/db.js` — MongoDB connection via Mongoose

4. **Create `.env` and `.env.example`** in `api/`:
   ```
   PORT=5000
   MONGO_URI=mongodb://localhost:27017/hackathon
   JWT_SECRET=your-secret-key
   ASSETS_URL=http://localhost:5000/
   FRONTEND_URL=http://localhost:4200
   ```

5. **Add BE scripts** to the root `package.json`:
   ```json
   {
     "scripts": {
       "start:api": "node api/server.js",
       "dev:api": "nodemon api/server.js",
       "test:api": "jest --config api/jest.config.js"
     }
   }
   ```

6. **Configure Jest for BE tests** — create `api/jest.config.js`:
   ```javascript
   module.exports = {
     testEnvironment: 'node',
     roots: ['<rootDir>'],
     testMatch: ['**/*.test.js'],
     globalSetup: './__tests__/setup.js',
     globalTeardown: './__tests__/teardown.js',
   };
   ```

7. **Create Jest setup/teardown** for in-memory MongoDB:
   ```javascript
   // api/__tests__/setup.js
   const { MongoMemoryServer } = require('mongodb-memory-server');

   module.exports = async () => {
     const mongod = await MongoMemoryServer.create();
     process.env.MONGO_URI = mongod.getUri();
     global.__MONGOD__ = mongod;
   };

   // api/__tests__/teardown.js
   module.exports = async () => {
     if (global.__MONGOD__) {
       await global.__MONGOD__.stop();
     }
   };
   ```

### 1. Plan the model

Before writing any code, outline what will be created:
- List the entity name and all fields (names, types, constraints, defaults)
- List indexes needed for queries and sorting
- List relationships to other entities
- List instance and static methods for common operations
- Present the plan to the user and wait for confirmation before proceeding

### 2. Generate model code

Create a single model file: `api/src/models/<EntityName>.js`

**Model generation rules:**

- **Schema definition**: Define all fields with proper types, constraints, defaults, and validation rules
- **Timestamps**: Always include `timestamps: true` for `createdAt` and `updatedAt`
- **Indexes**: Add indexes for frequently queried, filtered, or sorted fields
- **Relationships**: Use `Schema.Types.ObjectId` with `ref` for references to other models
- **Instance methods**: Add `.toJSON()` for safe serialization, add status check methods
- **Static methods**: Add query helpers for common database operations
- **Hooks**: Add `pre/post` hooks for automatic processing (hashing, normalization, validation)
- **No hardcoding**: All constraint messages and validation rules should be descriptive and user-friendly

**DO NOT:**
- Put business logic in the model — schema defines structure only
- Include DB operations like `.find()` or `.save()` in instance methods — those go in repositories
- Hardcode error messages — use descriptive validation error strings
- Forget `.toJSON()` method — always implement it to exclude sensitive fields in API responses

### 3. Generate model tests

Create a test file: `api/src/models/<EntityName>.test.js`

Use Jest and `mongodb-memory-server` for an in-memory MongoDB instance that hits a real database (not mocked).

**Model tests MUST cover:**

- **Schema validation**: Test required fields, field types, string length constraints, number min/max
- **Enums**: Test valid and invalid enum values
- **Unique constraints**: Test that duplicate values are rejected (if applicable)
- **Relationships**: Test references to other models, verify `ref` is set correctly
- **Instance methods**: Test `.toJSON()` removes sensitive fields, test status check methods
- **Static methods**: Test query helpers return correct documents, test filtering logic
- **Hooks**: Test pre/post hooks execute correctly (password hashing, email lowercase, etc.)
- **Timestamps**: Test `createdAt` and `updatedAt` are automatically set and updated
- **Defaults**: Test default values are applied correctly

Example model test:

```javascript
const mongoose = require('mongoose');
const User = require('./User');

describe('User Model', () => {
  it('should create a user with valid data', async () => {
    const user = await User.create({
      email: 'test@example.com',
      username: 'testuser',
      password: 'password123'
    });
    expect(user._id).toBeDefined();
    expect(user.email).toBe('test@example.com');
    expect(user.status).toBe('active');  // default value
  });

  it('should fail with missing required field', async () => {
    try {
      await User.create({ username: 'testuser' });
      fail('Should throw validation error');
    } catch (err) {
      expect(err.errors.email).toBeDefined();
    }
  });

  it('should fail with invalid email', async () => {
    try {
      await User.create({
        email: 'not-an-email',
        username: 'testuser',
        password: 'password123'
      });
      fail('Should throw validation error');
    } catch (err) {
      expect(err.errors.email).toBeDefined();
    }
  });

  it('should enforce unique email', async () => {
    await User.create({
      email: 'test@example.com',
      username: 'testuser1',
      password: 'password123'
    });
    try {
      await User.create({
        email: 'test@example.com',
        username: 'testuser2',
        password: 'password123'
      });
      fail('Should throw duplicate key error');
    } catch (err) {
      expect(err.code).toBe(11000);  // MongoDB duplicate key error
    }
  });

  it('should execute toJSON method and remove password', async () => {
    const user = await User.create({
      email: 'test@example.com',
      username: 'testuser',
      password: 'password123'
    });
    const json = user.toJSON();
    expect(json.password).toBeUndefined();
  });

  it('should find active users', async () => {
    await User.create({
      email: 'active@example.com',
      username: 'active',
      password: 'password123',
      status: 'active'
    });
    await User.create({
      email: 'inactive@example.com',
      username: 'inactive',
      password: 'password123',
      status: 'inactive'
    });
    const active = await User.findActive();
    expect(active.length).toBe(1);
    expect(active[0].email).toBe('active@example.com');
  });

  it('should have createdAt and updatedAt timestamps', async () => {
    const user = await User.create({
      email: 'test@example.com',
      username: 'testuser',
      password: 'password123'
    });
    expect(user.createdAt).toBeDefined();
    expect(user.updatedAt).toBeDefined();
    expect(user.createdAt).toEqual(user.updatedAt);
  });
});
```

---

## Code Style Rules

### Backend (JavaScript/Node.js)

- CommonJS module style (`require`/`module.exports`)
- 2-space indentation
- Single quotes for all strings
- Semicolons at end of statements
- No `var` — use `const` and `let` only
- `const` for all non-reassigned variables, `let` for variables that change
- PascalCase for class names and model names (e.g., `User`, `Product`, `Order`)
- camelCase for variables, functions, and field names
- UPPER_SNAKE_CASE for true constants only
- Always add JSDoc comments for complex logic or non-obvious field constraints

### Example Code Style:

```javascript
const mongoose = require('mongoose');
const Schema = mongoose.Schema;

/**
 * Product Schema
 * Represents a product in the inventory system
 * Fields: name, price, category, inventory tracking, status
 */
const productSchema = new Schema(
  {
    name: {
      type: String,
      required: [true, 'Product name is required'],
      trim: true,
      maxlength: [100, 'Product name cannot exceed 100 characters']
    },
    price: {
      type: Number,
      required: [true, 'Price is required'],
      min: [0, 'Price must be a positive number']
    },
    status: {
      type: String,
      enum: ['active', 'discontinued'],
      default: 'active',
      index: true
    }
  },
  { timestamps: true }
);

// Instance method to check if product is available
productSchema.methods.isAvailable = function() {
  return this.status === 'active';
};

// Static method to find active products
productSchema.statics.findActive = function() {
  return this.find({ status: 'active' });
};

module.exports = mongoose.model('Product', productSchema);
```

---

## Best Practices and Principles

All generated models MUST adhere to the following principles:

### SOLID Principles

- **Single Responsibility (SRP)** — each model represents ONE entity. A model only defines the schema structure and provides query helpers. Business logic and data transformation goes in services and repositories, not models.
- **Open/Closed (OCP)** — models should be extensible without modification. Use schema methods and hooks to add behavior without changing core schema.
- **Liskov Substitution (LSP)** — if a model is extended or a new model is created with similar patterns, it should work the same way.
- **Dependency Inversion (DIP)** — models should depend on MongoDB/Mongoose abstractions, not implementation details. Services depend on repositories, which depend on models.

### DRY (Don't Repeat Yourself)

- Don't duplicate field definitions across models — if multiple models need an `address` object, define it once
- Use instance methods for repeated transformations (e.g., `.toJSON()` is better than repeating serialization logic in controllers)
- Use static methods for repeated queries (e.g., `.findActive()` instead of writing `find({ status: 'active' })` everywhere)
- Share common validation rules across models via schema types or utilities

### KISS (Keep It Simple, Stupid)

- Schema definitions should be straightforward — avoid deeply nested objects unless necessary
- Don't over-engineer with multiple hooks when simple default values suffice
- Keep indexes minimal — only add indexes for fields that are actually queried frequently
- Use Mongoose built-in validation before adding custom validators

### YAGNI (You Aren't Gonna Need It)

- Only add fields the entity actually needs — don't add speculative fields "just in case"
- Don't add instance/static methods that aren't used by services or repositories
- Don't add indexes for fields that won't be queried or sorted
- Don't create relationships to other models unless data is actually accessed together

### Separation of Concerns

- **Models**: Define schema structure, field types, validation rules, indexes
- **Instance methods**: Data transformation and status checks for single documents (e.g., `.toJSON()`, `.isActive()`)
- **Static methods**: Query helpers for common database operations (e.g., `.findActive()`, `.findByEmail()`)
- **Repositories**: Complex data access logic, joins, aggregations, filtering, pagination
- **Services**: Business logic, orchestrating repositories, data transformation
- **Controllers**: Request/response handling, calling services

**Never put:**
- Business logic in models (e.g., "is this order eligible for discount?")
- Complex queries in models (those go in repositories)
- API response formatting in models (that goes in controllers or services)
- Authentication/authorization logic in models (that goes in middleware or services)

### Error Handling

- Use descriptive validation error messages in schema constraints
- Include `required: [true, 'Field name is required']` format for clear error messages
- Use `match: [regex, 'Error message']` for pattern validation with clear feedback
- Always handle pre/post hook errors properly — call `next(err)` to propagate
- Never silently fail in hooks

### Security

- Never store sensitive data (passwords, API keys) without processing — always hash passwords in pre-save hooks
- Use `select: false` on sensitive fields (password) so they aren't returned in queries by default
- Validate all input data via schema constraints — use `required`, `minlength`, `maxlength`, `match`, `enum`
- Use indexes carefully — indexed fields are searchable, so be mindful of sensitive fields
- Always include `.toJSON()` method to exclude sensitive fields from API responses
- Use lowercase and trim for email and username fields to prevent duplicates

### Naming Conventions

- **Files**: PascalCase matching entity name (`User.js`, `Product.js`, `BlogPost.js`)
- **Collections**: lowercase plural (e.g., `users`, `products`, `blog_posts`) — set via `collection: 'name'` in schema options
- **Classes/Models**: PascalCase (`User`, `Product`)
- **Schema variables**: camelCase + 'Schema' suffix (`userSchema`, `productSchema`)
- **Fields**: camelCase (`firstName`, `email`, `createdAt`)
- **Methods**: camelCase action verbs for instance methods (`.toJSON()`, `.isActive()`, `.isEligible()`)
- **Static methods**: camelCase with query intent (`findActive`, `findByEmail`, `countByStatus`)
- **Constants**: UPPER_SNAKE_CASE for true constants (`MAX_NAME_LENGTH = 100`)
- Names should be descriptive and self-documenting — avoid abbreviations (`usr`, `pwd`, `addr`) unless universally understood (`id`, `url`)

---

## Output Format

After completing the model generation, you MUST provide a detailed summary. This summary is mandatory — never skip it.

```
## Model: <EntityName>

### Summary
<2-3 sentences describing what entity was created, what it stores, and why>

### Schema Definition

| Field | Type | Required | Unique | Default | Validation | Index |
|-------|------|----------|--------|---------|------------|-------|
| `fieldName` | String | Yes/No | Yes/No | N/A | min/max length, pattern, enum | Yes/No |
| `referenceField` | ObjectId (ref: 'Model') | Yes/No | Yes/No | N/A | N/A | Yes/No |

### Relationships
- List all references to other models
- Explain 1-to-N or N-to-N relationships
- Example: "Belongs to User (one user has many orders)"

### Methods

Instance Methods:
- .toJSON() — Serializes document for API responses, removes password and sensitive fields
- .isActive() — Returns true if status is 'active'
- <list any other custom instance methods>

Static Methods:
- .findActive() — Returns all active documents
- .findByEmail(email) — Finds document by email (lowercase comparison)
- <list any other custom static methods>

### Hooks
- pre('save') — <describe what happens before document is saved (e.g., hash password, lowercase email)>
- post('find') — <describe what happens after documents are found>

### Indexes
| Fields | Order | Purpose |
|--------|-------|---------|
| `email` | ascending | Unique lookup and filtering |
| `status`, `createdAt` | asc, desc | Filter by status, sort by date |

### Validation Rules
- `email`: Required, unique, valid email format, lowercase, trimmed
- `password`: Required, minimum 8 characters, hashed before save
- `status`: Required, enum ['active', 'inactive', 'suspended'], defaults to 'active'
- <list all constraints>

### Test Coverage
- Schema validation (required fields, types, constraints)
- Enums (valid and invalid values)
- Unique constraints
- Relationships (ObjectId refs)
- Instance methods (.toJSON() removes sensitive fields, status checks)
- Static methods (query helpers return correct documents)
- Hooks (pre/post execution, data transformations)
- Timestamps (auto-created, auto-updated)
- Defaults (values applied on creation)

### Example Usage

Create a document:
```javascript
const user = await User.create({
  email: 'john@example.com',
  username: 'john_doe',
  password: 'securePassword123',
  firstName: 'John',
  lastName: 'Doe'
});
```

Find and filter:
```javascript
const activeUsers = await User.findActive();
const user = await User.findByEmail('john@example.com');
```

Populate references:
```javascript
const order = await Order.findById(orderId).populate('userId');
```

Serialize safely:
```javascript
const json = user.toJSON();  // password is excluded
```

### Decisions Made
- Decision 1: Why this choice was made over alternatives
- Decision 2: Why this approach was taken
- Example: "Used `enum` for status field to enforce valid states instead of free-text string"
- Example: "Added email unique index with lowercase transform to prevent case-sensitive duplicates"
- Example: "Created `toJSON()` instance method to ensure password is never exposed in API responses"

### Files Generated

| # | File Path | Purpose |
|---|-----------|---------|
| 1 | `api/src/models/<EntityName>.js` | Mongoose schema definition with indexes, methods, and hooks |
| 2 | `api/src/models/<EntityName>.test.js` | Comprehensive Jest tests with in-memory MongoDB |

### Next Steps
- <any manual steps, e.g., create related models first, add migrations, create indexes, seed test data>
- <if this model references other models, remind user to generate those models first>
- <if repository layer is needed next, point to repository generation skill>
```

**Rules for the summary:**
- Include EVERY file that was created
- Every file must have a clear purpose description
- List all fields with types, constraints, and indexes in a table
- Explain relationships clearly
- Include example usage showing how the model will be used
- Decisions section must explain WHY, not just WHAT
- Test coverage section must list what scenarios are covered
- If a Jira ticket was used as input, reference the ticket key in the summary

---

## Glossary

- **Schema**: The structure definition for a MongoDB collection in Mongoose
- **Model**: A constructor function that represents a MongoDB collection and provides query/CRUD methods
- **Document**: A single record (object) in a MongoDB collection
- **Relationship**: A connection between two models via references (refs) or embedded documents
- **Index**: A database index on one or more fields to speed up queries, filtering, and sorting
- **Validation**: Rules enforced on field types, lengths, patterns, and enum values before save
- **Hook/Middleware**: Pre/post functions that execute automatically before or after specific operations (save, find, etc.)
- **Ref**: A Mongoose reference to another model via ObjectId, enabling population of related documents
- **Denormalization**: Embedding related data in a document instead of referencing it (trade-off: faster reads, slower writes)
- **TTL Index**: Time-to-live index that automatically deletes documents after a specified time
