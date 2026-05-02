File name: SKILL-MODEL-GENERATOR-BUSINESS.md
Language: 
---
name: model-business
description: Convert business requirements from design/product teams into MongoDB Mongoose models for Node.js Express backend, translating business language into technical schema specifications
user_invocable: true
command: model-business
---

# Business Requirements to MongoDB Model Converter

Convert high-level business and product requirements (from design teams, product managers, stakeholders) into production-ready MongoDB Mongoose models for a Node.js + Express backend. This skill translates business language into technical schema specifications, eliminating the need for engineers to interpret requirements.

## Pre-Flight Checklist (STOP GATE)

**NEVER work on assumptions. NEVER skip this checklist.**

Before converting ANY business requirements, ensure the core business context is clear. Only ask the user about what's genuinely unclear — derive everything else from standard engineering best practices and the guidelines in this skill file.

### Core Checklist (ask the user only for these)

```
- [ ] What is the business entity? (name + what it represents in plain business language)
- [ ] What information needs to be captured? (list what fields/data as a non-technical person would describe it)
- [ ] What are the key workflows or business processes? (what happens with this data, who accesses it, key events)
```

**That's it.** Only these 3 items need explicit user input. Everything else — field types, validation rules, relationships, indexes, methods, technical constraints — is derived automatically by translating business concepts into technical implementations.

### What you derive (do NOT ask the user about these)

- **Field types** — translate business descriptions to Mongoose types:
  - "Price in dollars" → `Number`
  - "Yes/no status" → `Boolean`
  - "Date when published" → `Date`
  - "Link to another entity" → `ObjectId` with `ref`
  - "List of tags/labels" → `Array[String]`
  - "Address or location info" → Nested object with sub-fields
  - "Multiple images or files" → `Array[Object]` with url, alt, order
  - "Notes or detailed text" → `String` with appropriate length limit

- **Required vs. optional** — infer from business context:
  - "Customer name" → Required (can't sell without knowing who)
  - "Phone number" → Optional (customers often don't provide)
  - "Order total" → Required (needed for billing)
  - "Tracking number" → Optional until shipped

- **Validation rules** — apply sensible business constraints:
  - Email addresses → Must match email format
  - Prices → Must be >= 0
  - Status values → Restricted to allowed states (draft, published, archived)
  - Names → Reasonable length limits (min 3, max 150 chars)
  - Quantities → Must be whole numbers, >= 0

- **Relationships** — identify connections between entities:
  - "Product belongs to a category" → Reference to Category model
  - "Order contains multiple items" → Either embedded items array or reference to OrderItem collection
  - "User creates orders" → Reference to User model
  - Decision: Embedded (for always-accessed data) vs. Referenced (for separate concerns)

- **Indexes** — optimize for business queries:
  - "Find all active products" → Index on status field
  - "Search by customer email" → Index on email (likely unique)
  - "Show newest items first" → Index on createdAt descending
  - "Filter by category and status" → Compound index on both
  - "Autocomplete search" → Index on searchable text fields

- **Timestamps** — add createdAt and updatedAt automatically for audit trails

- **Methods** — generate helpers based on business processes:
  - Status checks: `isActive()`, `isDraft()`, `isPublished()`
  - Business logic: `getEffectivePrice()`, `isInStock()`, `isExpired()`
  - Serialization: `toJSON()` for API responses
  - Static finders: `findActive()`, `findByEmail()`, `findRecent()`

- **Hooks** — auto-generate technical implementations of business rules:
  - Pre-save validation: Enforce constraints (price validation, required fields when status changes)
  - Auto-generation: Slug from name, SKU generation, status transitions
  - Denormalization: Update counters, calculate derived fields

- **Error handling** — follow standard patterns for validation failures and business rule violations

### Jira Ticket Integration

If the user provides a Jira ticket key (e.g., `PROJ-123`) or URL (e.g., `https://vitecgmbh.atlassian.net/browse/PROJ-123`):

1. **Extract the ticket key** from the input — parse it from the URL if a full URL is provided, or use the key directly
2. **Fetch the ticket** using the Atlassian MCP tool:
   ```
   mcp__claude_ai_Atlassian__getJiraIssue with cloudId: "vitecgmbh.atlassian.net", issueKey: "<TICKET-KEY>"
   ```
3. **Extract from the ticket**: summary, description (business requirements), acceptance criteria, wireframes/designs, custom fields, linked issues
4. **Translate business language** — convert the ticket requirements into technical schema specifications
5. Run the core checklist against the extracted ticket content — if the 3 core items are still unclear, ask only about those gaps

### When to ask vs. when to proceed

**STOP and ask** — business context is too vague to identify entities and workflows:
- "We need a product system" — Ask: What information about each product? Who uses it? How do they search/filter?
- "Build an order tracking feature" — Ask: What happens when an order is placed? What status states? Who sees what?

**Proceed to planning** — core checklist is clear, derive the rest:
- "We need to catalog products with name, price, category, and track inventory levels. Customers search by category and price range."
- "We track customer orders with items, status (pending/shipped/delivered), and total cost. Customers can see their order history and current status."
- `PROJ-123` (Jira ticket with clear product/design requirements after fetching)

**Rules:**
- Ask at most 2-3 focused questions in plain language, NOT technical jargon
- Present questions as a short numbered list and WAIT for answers
- Never ask about "database types," "normalization," "indexing strategies," etc. — these are implementation details, not business questions
- Focus only on: What is it? What data? What workflows/processes?

---

## Existing Model Detection

Before generating anything, check if a model with the same name already exists in the codebase. Search for:
- `api/src/models/<EntityName>.js` (Mongoose model file)

### If the model already exists:

**1. Determine the type of request** — compare what the user is asking for against what already exists:

**Enhancement Request (add/modify business features):**
The user wants to add new business capabilities, capture new information, or change workflows in the existing entity.

- Read and understand the existing model completely
- Translate the new business requirements into schema additions/modifications
- Make changes incrementally — modify existing file, don't recreate from scratch
- **NEVER break existing functionality** — all current business processes must continue to work
- If new fields are added to support new workflows, make them optional with sensible defaults
- If validation rules change, ensure backward compatibility
- Document what changed and why in code comments

**Completely Different Business Entity (different purpose, different data model):**
The user's description is fundamentally different from what exists — different business purpose, different data captured.

- **STOP and warn the user** before proceeding. Present in plain business language:
  ```
  ⚠️ WARNING: A model called "<EntityName>" already exists, but it represents something different.

  What we have:
  - Represents: <plain description of what it currently models>
  - Captures: <list of business information currently tracked>
  - Business processes: <what workflows it supports>

  What you're asking for:
  - Represents: <plain description of what you want>
  - Captures: <list of business information you need>
  - Business processes: <different workflows>

  These are two different things. Do you want to:
  1. Replace the existing model (this will change how we store this data)
  2. Create a new model with a different name (recommended to keep both)
  3. Cancel and enhance the existing model instead
  ```
- **Wait for explicit confirmation** — do NOT proceed without the user choosing an option
- If the user chooses to replace, document what business processes will be affected

### If the model does NOT exist:

Proceed normally with the pre-flight checklist and planning steps.

---

## Input

The user provides business requirements in plain language. Examples:
- "We need to list products for sale. Each product has a name, price, category, and we track how many we have in stock. Customers should be able to see which products are available."
- "Customers place orders. An order contains items they're buying, the total cost, and we need to track if it's been shipped yet. Customers want to see their order history."
- "We're launching a blog. Each blog post has a title, body text, author, and we need to know when it was published and if it's publicly visible or still in draft mode."

Parse the description to determine:
- **What business entity** is being modeled (Product, Order, BlogPost, etc.)
- **What information** needs to be captured (in business terms, not technical)
- **What processes/workflows** use this data (who accesses it, what they do, what changes)

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

## Translation Guide: Business Language to Technical Schema

This guide shows how to interpret common business descriptions and translate them into technical schema specifications.

### Entity Names
| Business Term | Technical Term | Example |
|---------------|----------------|---------|
| Item, thing, record | Entity/Model | "We track Product records" → `Product` model |
| List of X | Array field | "Multiple phone numbers" → `phones: [String]` |
| Link to another | Relationship/Reference | "Each order belongs to a customer" → `customerId: ObjectId ref User` |
| Grouping or category | Nested object or reference | "Address with street, city, zip" → Nested object or reference to Address |
| Yes/no, on/off | Boolean flag | "Is the product in stock?" → `inStock: Boolean` |
| Count or measurement | Number | "How many in stock?" → `quantity: Number` |
| Money/price | Number | "Price in dollars" → `price: Number` |
| Date/time | Date | "When was it published?" → `publishedAt: Date` |
| Text/description | String | "Product description" → `description: String` |
| Status/state | String with enum | "Order is pending or shipped" → `status: String enum: ['pending', 'shipped']` |
| Unique identifier | String, often unique | "Customer email" → `email: String unique` |
| For record-keeping | Audit fields | "Who created this and when?" → `createdBy: ObjectId, createdAt: Date` |

### Business Workflows to Technical Implementation

#### Example 1: Inventory Management
**Business:** "We track product inventory. When inventory gets low, we need alerts."

**Translation:**
- Captures: current quantity, reorder threshold
- Implementation:
  ```
  inventory: {
    quantity: Number,
    reorderLevel: Number
  }
  ```
- Method: `isLowStock()` returns true if `quantity <= reorderLevel`
- Hook: Pre-save validation ensures reorderLevel is >= 0

#### Example 2: Status Workflows
**Business:** "Orders go through states: customer places it (pending), we process it (processing), send it out (shipped), then delivered."

**Translation:**
- Status field: `status: String enum: ['pending', 'processing', 'shipped', 'delivered']`
- Methods: `isPending()`, `isProcessing()`, `isShipped()`, `isDelivered()`
- Hooks: Pre-save validation ensures status can only change to valid next states
- Timestamps:
  - `createdAt` — when order was placed (pending)
  - `shippedAt` — when status changed to shipped
  - `deliveredAt` — when status changed to delivered

#### Example 3: Search and Filtering
**Business:** "Customers search products by name, filter by category and price range."

**Translation:**
- Fields needed: `name`, `category`, `price` — all must be indexed
- Methods:
  - `Product.findByCategory(categoryId)` — search by category
  - `Product.findByPriceRange(min, max)` — filter by price
  - Text index on name for search autocomplete
- Indexes: `category`, `price`, text index on `name`

#### Example 4: Audit Trails
**Business:** "We need to know who created/modified records and when for compliance."

**Translation:**
- Fields:
  - `createdAt` — automatically set on creation
  - `updatedAt` — automatically updated on every change
  - `createdBy: ObjectId ref User` — who created it
  - `updatedBy: ObjectId ref User` — who last modified it
- Hook: Pre-save middleware sets `updatedBy` from authenticated user
- Index: `createdAt` descending for "newest first" queries

#### Example 5: Denormalization for Performance
**Business:** "Products show customer ratings. Customers want to see highly-rated products first."

**Translation:**
- Fields on Product:
  - `rating: Number` (0-5)
  - `reviewCount: Number` (total reviews)
- These are calculated from Review collection but stored on Product for fast queries
- Index: Compound index `{ rating desc, reviewCount desc }` for trending queries
- Service layer updates these fields when new reviews are added

### Common Validations

| Business Rule | Technical Validation |
|---------------|----------------------|
| "Name is required" | `required: true`, `minlength: 3`, `maxlength: 150` |
| "Email must be valid" | `match: /^[^\s@]+@[^\s@]+\.[^\s@]+$/` |
| "Price must be positive" | `min: 0` |
| "Quantity is a whole number" | `type: Number`, integer check |
| "Status must be one of X options" | `enum: ['pending', 'shipped', ...]` |
| "Email must be unique" | `unique: true` |
| "Discount can't exceed regular price" | Custom pre-save hook validation |
| "Can't ship order that's cancelled" | Hook checks current status before allowing transition |
| "Product must have at least one image when published" | Pre-save: if status is 'active', images array length > 0 |

---

## Steps

### 0. Project setup (run ONCE if not already set up)

Before generating any model, check if the project infrastructure is in place. If any of the following are missing, set them up first:

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
       routes/
       controllers/
       services/
       repositories/
       middlewares/
       validations/
       config/
       utils/
       models/
     __tests__/
       setup.js
       teardown.js
     app.js
     server.js
     .env
     .env.example
   ```

   **Note:** `api/` does NOT have its own `package.json` or `node_modules/`. It uses the root project's `package.json` and `node_modules/`. All dependencies (FE and BE) are managed from the project root.

3. **Create base files** (fetch patterns from Confluence page):
   - `api/app.js` — Express app setup with CORS, middleware, route mounting, and error handler
   - `api/server.js` — HTTP server entry point (listens on its own port)
   - `api/src/config/env.js` — environment variables (PORT, MONGO_URI, JWT_SECRET, FRONTEND_URL)
   - `api/src/config/db.js` — MongoDB connection via Mongoose
   - `api/src/controllers/base.controller.js` — BaseController with sendResponse/sendError/sendPaginated
   - `api/src/repositories/base.repository.js` — BaseRepository with aggregatePaginate and helpers
   - `api/src/middlewares/error.middleware.js` — centralized error handler
   - `api/src/middlewares/validate.js` — Joi validation middleware
   - `api/src/validations/base.validator.js` — shared validation schema factory

4. **Configure CORS** in `api/app.js` — allow requests from the Angular frontend:
   ```javascript
   const cors = require('cors');
   app.use(cors({
     origin: process.env.FRONTEND_URL || 'http://localhost:4200',
     credentials: true,
   }));
   ```

5. **Create `.env` and `.env.example`** in `api/`:
   ```
   PORT=5000
   MONGO_URI=mongodb://localhost:27017/hackathon
   JWT_SECRET=your-secret-key
   ASSETS_URL=http://localhost:5000/
   FRONTEND_URL=http://localhost:4200
   ```

6. **Add BE scripts** to the root `package.json`:
   ```json
   {
     "scripts": {
       "start:api": "node api/server.js",
       "dev:api": "nodemon api/server.js",
       "test:api": "jest --config api/jest.config.js"
     }
   }
   ```

7. **Configure Jest for BE tests** — create `api/jest.config.js`:
   ```javascript
   module.exports = {
     testEnvironment: 'node',
     roots: ['<rootDir>'],
     testMatch: ['**/*.test.js'],
     globalSetup: './__tests__/setup.js',
     globalTeardown: './__tests__/teardown.js',
   };
   ```

8. **Create Jest setup/teardown** for in-memory MongoDB:
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

This setup step only runs once. On subsequent `/model-business` invocations, check if the infrastructure exists and skip to step 1.

### 1. Validate and Translate Business Requirements

Parse the business requirements provided by the user:

1. **Identify the entity** — What is the business object? (Product, Order, BlogPost, etc.)
2. **Extract business information** — What data needs to be captured? (list in business language)
3. **Identify workflows/processes** — How is this data used? Who accesses it? What happens to it over time?
4. **Translate to technical schema** — Convert each business requirement to:
   - Field name (camelCase)
   - Field type (String, Number, Boolean, Date, ObjectId, Array, Object)
   - Validation rules (required, length limits, enums, unique, format)
   - Relationships (if links to other entities)
   - Special handling (timestamps, auto-generation, derived fields)

### 2. Identify Relationships and Connections

Based on business context, determine:
- **What other entities** does this connect to? (Users, Categories, Orders, etc.)
- **Embedded vs. Referenced?**
  - Embed if: The data is always accessed together, not shared with other entities, small documents
  - Reference if: The data is shared, independently managed, or very large arrays
- **Cardinality** — One-to-one, One-to-many, Many-to-many?

**Example translations:**
- "Each product belongs to a category" → Reference `category: ObjectId ref Category`
- "A customer has an address" → Embedded if small, Referenced if large/shared
- "An order has multiple items" → Either embedded array or reference to OrderItem collection

### 3. Plan Business Rules and Validation

Translate business workflows into technical constraints:

1. **Status transitions** — What states can an entity move through?
   - Example: Order: pending → processing → shipped → delivered
   - Implementation: Status field with enum, pre-save hooks to validate transitions

2. **Required information at different stages** — What data is required when?
   - Example: Product can be draft (name required), or active (name + image + price required)
   - Implementation: Conditional validation in pre-save hook

3. **Business constraints** — What rules must always be true?
   - Example: Discount price < regular price
   - Implementation: Pre-save validation hook

4. **Derived/calculated fields** — What data can be computed?
   - Example: Order total from items, product rating from reviews
   - Implementation: Pre-save hooks or service layer calculations

### 4. Identify Search and Filtering Needs

Based on business workflows, determine what queries will be common:
- "Customers filter products by category" → Index on `category`
- "Show newest products first" → Index on `createdAt` descending
- "Search products by name" → Text index on `name`
- "Find all active/published content" → Index on `status`
- "Find orders by customer and date" → Compound index on `customerId`, `createdAt`

### 5. Create Translation Document

Before writing any code, create a clear translation document showing:
- Original business requirement ↔ Technical implementation
- Fields with types and validations
- Relationships and why they're embedded/referenced
- Indexes and their purpose
- Methods and hooks with business purpose

Present this to the user for confirmation. Example:

```
TRANSLATION PLAN: Blog Post Model

Business: "We need a blog platform. Posts have titles, content, and an author. 
           We track if they're in draft or published, and when they go live."

Technical Translation:

1. FIELDS:
   Business: "Post has a title"
   → title: String, required, 3-200 chars, trimmed
   
   Business: "Posts have content"
   → content: String, required, min 10 chars
   
   Business: "Posts have an author"
   → authorId: ObjectId, reference to User, required
   
   Business: "Draft or published status"
   → status: String enum ['draft', 'published'], default: 'draft'
   
   Business: "We know when posts are published"
   → publishedAt: Date, null until published

2. RELATIONSHIPS:
   Blog Post → User (one-to-one) — The person who wrote it
   → authorId: ObjectId ref User
   → Reason: Author info is often accessed with post, independent User entity

3. VALIDATIONS:
   - Title is required (can't publish without title)
   - When status changes to 'published', publishedAt must be set
   - Only author or admin can modify published posts

4. METHODS (what we can ask the post):
   - post.isDraft() — is it still being worked on?
   - post.isPublished() — is it live?
   - post.getAuthorName() — get author's name without separate query

5. SEARCHES:
   - Find all published posts (for listing)
   - Find recent posts (sort by publishedAt descending)
   - Find posts by author (for author profile)
   → Indexes needed: { status }, { publishedAt }, { authorId }

Ready to generate BlogPost model? (YES/NO)
```

### 6. Generate the Model

Once the user confirms the translation, generate the Mongoose model file:

**File:** `api/src/models/<EntityName>.js`

Contains:
1. Schema definition with all fields and validations
2. Timestamps (createdAt, updatedAt)
3. All relationships with refs and populate options
4. Instance methods (toJSON, status checks, helpers)
5. Static methods (finders, counters, search helpers)
6. Pre-save hooks for validation and auto-generation
7. Indexes for search/filter performance
8. Comments explaining business purpose of each field/method

### 7. Generate Tests

**File:** `api/src/models/<EntityName>.test.js`

Test coverage includes:
1. Field type and constraint validation
2. Required field validation
3. Enum/status field validation
4. Unique field validation
5. Relationship validation
6. Instance method functionality
7. Static method functionality
8. Pre-save hook behavior
9. Timestamp auto-generation
10. Default values
11. Edge cases and error conditions

### 8. Output Summary

Present:
1. **What was generated** — List of files created/modified
2. **Business translation** — Summary of how business requirements became technical implementation
3. **What it can do** — Plain language description of capabilities
4. **Next steps** — Who does what next (frontend integration, backend routes, etc.)

---

## Example: Converting Business Requirements to Model

### Input (Business Language)

> "We need to manage blog posts for our website. Each post has a title, the content, and we need to know who wrote it. We track whether a post is in draft mode or published. When we publish it, we want to know the exact date it went live. We need to search for posts by author, find recently published posts, and see draft posts that need review. Authors should only be able to edit their own posts."

### Translation Process

**Step 1: Identify the entity**
- Entity: BlogPost (represents a single blog article)

**Step 2: Extract business information (in order of importance)**
- Title of the post
- Content/body of the post
- Who wrote it (author)
- Publication status (draft or published)
- When it was published
- Timestamps for when created and last modified

**Step 3: Translate to fields**

| Business Requirement | Field Name | Type | Validation | Why |
|----------------------|-----------|------|-----------|-----|
| Title of post | title | String | Required, 3-200 chars | Can't have untitled posts |
| Main content | content | String | Required, min 100 chars | Needs actual content |
| Author of post | authorId | ObjectId ref User | Required | Who wrote it (link to user) |
| Draft or published | status | String enum | Required, default 'draft' | Control visibility |
| When published | publishedAt | Date | Optional, null = not published | Audit trail + sorting |
| Record creation | createdAt | Date | Auto | Audit trail |
| Last change | updatedAt | Date | Auto | Audit trail |

**Step 4: Identify relationships**
- Post → User (Many-to-One): Each post has one author
- Implementation: `authorId: ObjectId ref User`
- Embedded vs. Referenced: Referenced (User is independent entity, author data not always needed)

**Step 5: Identify business rules**
- "Authors should only edit their own posts" → Enforced in controller/middleware, not model
- "Draft posts can be edited freely" → No special validation
- "Published posts need timestamps" → Pre-save hook: if status changes to 'published', set publishedAt to now if not already set
- "Can't un-publish a post" → Pre-save hook: if status was 'published' and tries to change to 'draft', reject

**Step 6: Identify search patterns**
- "Search by author" → Index on `authorId`
- "Show recent posts first" → Index on `publishedAt` descending
- "Find draft posts for review" → Index on `status`
- Possible compound index: `{ status, publishedAt }`

**Step 7: Plan methods and helpers**

Instance methods:
- `isDraft()` — Is this post still being worked on?
- `isPublished()` — Is this post live?
- `canEdit(userId)` — Can this user edit this post?
- `toJSON()` — Serialize for API responses

Static methods:
- `findPublished()` — Get all published posts
- `findByAuthor(authorId)` — Get all posts by an author
- `findDraft()` — Get unpublished/draft posts
- `findRecent()` — Get newest published posts

### Output (Technical Model Specification)

```
BlogPost Model Schema:

{
  title: {
    type: String,
    required: true,
    minlength: 3,
    maxlength: 200,
    trim: true
  },
  
  content: {
    type: String,
    required: true,
    minlength: 100
  },
  
  authorId: {
    type: Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  
  status: {
    type: String,
    enum: ['draft', 'published'],
    default: 'draft'
  },
  
  publishedAt: {
    type: Date,
    default: null
  },
  
  createdAt: { type: Date, auto: true },
  updatedAt: { type: Date, auto: true }
}

Indexes:
- { authorId }
- { status }
- { publishedAt: -1 }
- { status, publishedAt: -1 }

Methods:
- isDraft() → returns status === 'draft'
- isPublished() → returns status === 'published'
- canEdit(userId) → returns authorId === userId
- toJSON() → returns serialized object

Static Methods:
- findPublished() → { status: 'published' }
- findByAuthor(authorId) → { authorId }
- findDraft() → { status: 'draft' }
- findRecent() → { status: 'published' }.sort({ publishedAt: -1 })

Pre-save Hooks:
- If status changes from 'draft' to 'published' and publishedAt is null, set to now
- If status changes from 'published' to 'draft', throw error (can't un-publish)
```

---

## Mongoose Model Rules

All Mongoose models MUST follow these rules. **Fetch and read the Confluence reference** before generating models:

> https://vitecgmbh.atlassian.net/wiki/spaces/TWD/pages/2737045509/BE+-+Project+Structure+-+NodeJS

Use the Atlassian MCP tool (`mcp__claude_ai_Atlassian__getConfluencePage` with `cloudId: "vitecgmbh.atlassian.net"`, `pageId: "2737045509"`, `contentFormat: "markdown"`) to fetch the page content.

The Confluence page defines the **layered architecture**, base classes, middleware patterns, config setup, Joi validators, and the standard JSON response shape.

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
| String | `String` | Text fields (names, emails, descriptions) | `required`, `minlength`, `maxlength`, `enum`, `trim`, `lowercase`, `uppercase`, `match` (regex) | `{ name: { type: String, required: true, minlength: 3 } }` |
| Number | `Number` | Numeric fields (prices, quantities, ratings) | `required`, `min`, `max` | `{ price: { type: Number, required: true, min: 0 } }` |
| Boolean | `Boolean` | True/false flags | `default` | `{ isActive: { type: Boolean, default: true } }` |
| Date | `Date` | Timestamps and dates | `default: Date.now` | `{ publishedAt: { type: Date, default: null } }` |
| ObjectId | `Schema.Types.ObjectId`, `ref: '<Model>'` | References to other documents | `ref`, optional or required | `{ userId: { type: Schema.Types.ObjectId, ref: 'User', required: true } }` |
| Array | `[Type]` or `{ type: [Type] }` | Collections of values or objects | `default: []`, specify element type | `{ tags: [String] }` |
| Object/Nested | `{ field1: Type, field2: Type }` | Embedded documents (non-normalized) | Use for frequently-accessed related data | `{ address: { street: String, city: String } }` |
| Enum | `{ type: String, enum: ['value1', 'value2'] }` | Restricted set of values | Define all allowed values | `{ status: { type: String, enum: ['active', 'inactive'] } }` |

---

## Summary

This skill serves as a **translator between business requirements and technical implementation**. Non-technical stakeholders describe what they need in business language, and this skill:

1. **Validates** that the business requirements are clear enough to implement
2. **Translates** business concepts into technical schema specifications
3. **Identifies** relationships, validations, and business rules
4. **Plans** search patterns, indexes, and performance optimizations
5. **Generates** production-ready MongoDB Mongoose models with full test coverage
6. **Documents** the translation so engineers understand the business intent behind each technical decision

The output is a model that is both **technically correct** and **semantically linked to business requirements** — making future maintenance, changes, and scaling easier because everyone understands not just what the code does, but why it does it.
