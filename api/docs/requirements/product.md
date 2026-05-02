# Product Model Requirements

## Overview
The Product model represents physical or digital items available for purchase in the e-commerce system. Products can be searched, filtered by category, and tracked by inventory levels. Each product has pricing information, descriptions, and can be marked as active or discontinued.

## Fields

| Field Name | Type | Required | Unique | Default | Validation Rules | Notes |
|-----------|------|----------|--------|---------|------------------|-------|
| name | String | Yes | No | N/A | Min 3 chars, max 150 chars, trimmed | Product display name |
| slug | String | Yes | Yes | N/A | Lowercase, kebab-case, auto-generated from name | URL-friendly identifier for SEO |
| description | String | No | No | N/A | Max 2000 chars | Detailed product description |
| shortDescription | String | No | No | N/A | Max 500 chars | Brief product summary for listings |
| sku | String | Yes | Yes | N/A | Uppercase alphanumeric, 5-20 chars | Stock Keeping Unit for inventory tracking |
| price | Number | Yes | No | N/A | Min 0, must be > 0 | Product price in dollars |
| discountPrice | Number | No | No | null | Min 0, if provided must be < price | Discounted price (optional sale price) |
| category | ObjectId (ref: 'Category') | Yes | No | N/A | N/A | Reference to product category |
| tags | Array[String] | No | No | [] | Max 10 tags, each max 50 chars | Search and filter tags |
| images | Array[Object] | No | No | [] | Min 1 image if product is active | Array of image objects with url, alt text, order |
| inventory | Object | Yes | No | N/A | N/A | Embedded inventory tracking object |
| inventory.quantity | Number | Yes | No | 0 | Min 0 | Current stock quantity |
| inventory.reorderLevel | Number | No | No | 10 | Min 0 | Quantity threshold for reorder alerts |
| inventory.warehouseId | String | No | No | N/A | N/A | Warehouse location identifier |
| status | String | Yes | No | 'draft' | Enum: ['draft', 'active', 'discontinued'] | Product availability status |
| rating | Number | No | No | 0 | Min 0, max 5, decimal to 1 place | Average customer rating (0-5 stars) |
| reviewCount | Number | No | No | 0 | Min 0 | Total number of customer reviews |
| createdBy | ObjectId (ref: 'User') | Yes | No | N/A | N/A | Reference to user who created product |
| updatedBy | ObjectId (ref: 'User') | No | No | null | N/A | Reference to user who last updated product |
| publishedAt | Date | No | No | null | Must be >= createdAt if provided | When product was published to storefront |
| metadata | Object | No | No | {} | N/A | Flexible metadata object (seo, custom fields) |

## Business Rules

1. **Auto-generated Slug**: When a product is created, automatically generate a URL-friendly slug from the product name (convert to lowercase, replace spaces with hyphens, remove special chars). The slug field should be unique.

2. **Price Validation**: If `discountPrice` is provided, it must always be less than `price`. Cannot have a discount equal to or greater than the regular price.

3. **Inventory Tracking**: The `inventory` object tracks stock levels. When inventory quantity reaches `reorderLevel`, a reorder alert should be triggered (handled by services/controllers, not the model).

4. **Status Workflow**: 
   - Products can be created as 'draft' (not visible to customers)
   - When `publishedAt` is set, status should be 'active'
   - Status can be changed to 'discontinued' to remove from sale
   - A discontinued product should still be queryable (for historical/audit purposes)

5. **Images Required When Active**: If a product is active, it MUST have at least one image. Draft products can have no images.

6. **Rating Calculation**: The rating field is calculated from customer reviews. It should be updated when new reviews are added (calculated/updated by services, not the model directly). Range is 0-5 with 1 decimal place (e.g., 4.5).

7. **Timestamps**: Automatically track when product was created and last updated via `createdAt` and `updatedAt` fields.

8. **Category Required**: Every product must belong to a category. If the referenced category is deleted, this should be handled by services (e.g., move product to "Uncategorized" or prevent deletion).

## Relationships

1. **Product → Category (Many-to-One)**
   - Type: Reference (normalized)
   - Field: `category` - ObjectId reference to Category model
   - Rationale: Multiple products belong to one category; category data is accessed frequently with products
   - Populate: Include category details when fetching product for detail views

2. **Product → User (Many-to-One for creator)**
   - Type: Reference (normalized)
   - Field: `createdBy` - ObjectId reference to User model who created the product
   - Field: `updatedBy` - ObjectId reference to User model who last modified (optional)
   - Rationale: Audit trail for who created/modified each product; User data is accessed infrequently
   - Populate: Include user details when viewing product history/audit logs

3. **Product ← Review (One-to-Many)**
   - Type: Implicit (handled by Review model)
   - Rationale: Reviews are stored separately in Review collection with productId reference
   - Query: When displaying product, fetch related reviews separately or pre-calculate rating

## Indexes

| Fields | Type | Purpose |
|--------|------|---------|
| `slug` | unique | Fast lookup by URL-friendly identifier for route parameters |
| `sku` | unique | Inventory system lookup by stock code |
| `category` | ascending | Filter products by category (frequently used query) |
| `status` | ascending | Filter by draft/active/discontinued (frequently used) |
| `status`, `publishedAt` | compound, desc | Find active published products sorted by newest first |
| `tags` | ascending | Query products by tags (array index) |
| `price` | ascending | Sort products by price in listings |
| `rating`, `reviewCount` | compound, desc | Find trending/popular products by rating and review count |
| `createdAt` | descending | Sort products by newest first (common query pattern) |
| `category`, `status`, `price` | compound | Complex filter: products in category, active, with price range |

## Instance Methods

### toJSON()
Serializes the product document for API responses. Includes all public fields. No sensitive fields to exclude for products.

```javascript
productSchema.methods.toJSON = function() {
  const obj = this.toObject();
  return obj;
};
```

### isActive()
Returns true if product status is 'active'.

```javascript
productSchema.methods.isActive = function() {
  return this.status === 'active';
};
```

### isDraft()
Returns true if product status is 'draft'.

```javascript
productSchema.methods.isDraft = function() {
  return this.status === 'draft';
};
```

### isDiscontinued()
Returns true if product status is 'discontinued'.

```javascript
productSchema.methods.isDiscontinued = function() {
  return this.status === 'discontinued';
};
```

### isLowStock()
Returns true if current inventory quantity is at or below reorder level.

```javascript
productSchema.methods.isLowStock = function() {
  return this.inventory.quantity <= this.inventory.reorderLevel;
};
```

### hasDiscount()
Returns true if discountPrice is set and valid (less than regular price).

```javascript
productSchema.methods.hasDiscount = function() {
  return this.discountPrice && this.discountPrice < this.price;
};
```

### getEffectivePrice()
Returns the current effective price (discounted price if available, otherwise regular price).

```javascript
productSchema.methods.getEffectivePrice = function() {
  return this.hasDiscount() ? this.discountPrice : this.price;
};
```

## Static Methods

### findActive()
Find all active products.

```javascript
productSchema.statics.findActive = function() {
  return this.find({ status: 'active' });
};
```

### findBySku(sku)
Find product by SKU code.

```javascript
productSchema.statics.findBySku = function(sku) {
  return this.findOne({ sku: sku.toUpperCase() });
};
```

### findBySlug(slug)
Find product by URL slug.

```javascript
productSchema.statics.findBySlug = function(slug) {
  return this.findOne({ slug: slug.toLowerCase() });
};
```

### findByCategory(categoryId)
Find all products in a specific category.

```javascript
productSchema.statics.findByCategory = function(categoryId) {
  return this.find({ category: categoryId, status: 'active' });
};
```

### findLowStock()
Find all products with inventory below reorder level.

```javascript
productSchema.statics.findLowStock = function() {
  return this.find({ 'inventory.quantity': { $lte: { $ref: 'inventory.reorderLevel' } } });
};
```

### findByTag(tag)
Find all products with a specific tag.

```javascript
productSchema.statics.findByTag = function(tag) {
  return this.find({ tags: tag.toLowerCase(), status: 'active' });
};
```

### countByCategory(categoryId)
Count products in a specific category.

```javascript
productSchema.statics.countByCategory = function(categoryId) {
  return this.countDocuments({ category: categoryId, status: 'active' });
};
```

## Hooks

### pre('save') - Generate slug from name
Before saving, automatically generate the `slug` field from the product name if the name has changed. Only generate if slug is not already set or name is modified.

```javascript
productSchema.pre('save', function(next) {
  if (this.isModified('name') && !this.slug) {
    this.slug = this.name
      .toLowerCase()
      .trim()
      .replace(/\s+/g, '-')
      .replace(/[^\w\-]/g, '');
  }
  next();
});
```

### pre('save') - Validate images for active products
Before saving, check that if status is 'active', the product has at least one image. If not, throw validation error.

```javascript
productSchema.pre('save', function(next) {
  if (this.status === 'active' && (!this.images || this.images.length === 0)) {
    return next(new Error('Active products must have at least one image'));
  }
  next();
});
```

### pre('save') - Validate discount price
Before saving, ensure that if discountPrice is provided, it is less than the regular price.

```javascript
productSchema.pre('save', function(next) {
  if (this.discountPrice && this.discountPrice >= this.price) {
    return next(new Error('Discount price must be less than regular price'));
  }
  next();
});
```

## Example Documents

### Example 1: Active Product with Discount
```json
{
  "_id": "507f1f77bcf86cd799439011",
  "name": "Premium Wireless Headphones",
  "slug": "premium-wireless-headphones",
  "description": "High-quality wireless headphones with noise cancellation, 30-hour battery life, and premium sound quality.",
  "shortDescription": "Premium wireless headphones with noise cancellation",
  "sku": "HWPH001",
  "price": 299.99,
  "discountPrice": 199.99,
  "category": "507f1f77bcf86cd799439012",
  "tags": ["electronics", "audio", "headphones", "wireless"],
  "images": [
    {
      "url": "https://cdn.example.com/products/hwph001-main.jpg",
      "alt": "Premium Wireless Headphones - Front View",
      "order": 1
    },
    {
      "url": "https://cdn.example.com/products/hwph001-side.jpg",
      "alt": "Premium Wireless Headphones - Side View",
      "order": 2
    }
  ],
  "inventory": {
    "quantity": 150,
    "reorderLevel": 20,
    "warehouseId": "WH-NYC-01"
  },
  "status": "active",
  "rating": 4.7,
  "reviewCount": 342,
  "createdBy": "507f1f77bcf86cd799439013",
  "updatedBy": "507f1f77bcf86cd799439013",
  "publishedAt": "2026-01-15T10:30:00Z",
  "metadata": {
    "seo": {
      "metaTitle": "Premium Wireless Headphones | Best Audio Quality",
      "metaDescription": "Shop premium wireless headphones with noise cancellation and 30-hour battery.",
      "keywords": ["headphones", "wireless", "audio", "noise cancellation"]
    }
  },
  "createdAt": "2026-01-01T08:00:00Z",
  "updatedAt": "2026-04-20T14:22:30Z"
}
```

### Example 2: Draft Product (Not Published)
```json
{
  "_id": "507f1f77bcf86cd799439020",
  "name": "Upcoming Smartwatch Pro",
  "slug": "upcoming-smartwatch-pro",
  "description": "Next-generation smartwatch with advanced health tracking and extended battery life.",
  "shortDescription": "Next-generation smartwatch",
  "sku": "SWP001",
  "price": 499.99,
  "discountPrice": null,
  "category": "507f1f77bcf86cd799439021",
  "tags": ["electronics", "wearables", "smartwatch"],
  "images": [],
  "inventory": {
    "quantity": 0,
    "reorderLevel": 50,
    "warehouseId": null
  },
  "status": "draft",
  "rating": 0,
  "reviewCount": 0,
  "createdBy": "507f1f77bcf86cd799439022",
  "updatedBy": null,
  "publishedAt": null,
  "metadata": {},
  "createdAt": "2026-04-20T16:45:00Z",
  "updatedAt": "2026-04-20T16:45:00Z"
}
```

### Example 3: Discontinued Product
```json
{
  "_id": "507f1f77bcf86cd799439030",
  "name": "Classic Analog Watch",
  "slug": "classic-analog-watch",
  "description": "A timeless analog watch that has been discontinued.",
  "shortDescription": "Discontinued analog watch",
  "sku": "WAT001",
  "price": 149.99,
  "discountPrice": null,
  "category": "507f1f77bcf86cd799439031",
  "tags": ["watches", "accessories"],
  "images": [
    {
      "url": "https://cdn.example.com/products/wat001.jpg",
      "alt": "Classic Analog Watch",
      "order": 1
    }
  ],
  "inventory": {
    "quantity": 5,
    "reorderLevel": 0,
    "warehouseId": "WH-LA-02"
  },
  "status": "discontinued",
  "rating": 4.2,
  "reviewCount": 89,
  "createdBy": "507f1f77bcf86cd799439032",
  "updatedBy": "507f1f77bcf86cd799439032",
  "publishedAt": "2024-06-01T09:00:00Z",
  "metadata": {
    "discontinuedReason": "Replaced by newer model"
  },
  "createdAt": "2024-01-10T07:30:00Z",
  "updatedAt": "2026-03-15T11:20:00Z"
}
```

## Notes

1. **Slug Generation**: The slug should be generated automatically during model creation to ensure it's always URL-safe and unique. This prevents duplicate slugs and simplifies route lookups.

2. **Inventory Subdocument**: The `inventory` object is embedded (not referenced) because it's frequently accessed with the product and not shared across products. This is a one-to-one relationship best handled via denormalization.

3. **Images Array**: Images are stored as an array of objects with `url`, `alt` text, and `order` properties. This allows multiple images per product with proper ordering for gallery displays.

4. **Metadata Field**: The `metadata` object is flexible and allows for extensibility without schema changes. This is useful for SEO data, custom fields, and future features.

5. **Rating Persistence**: The `rating` and `reviewCount` fields are persisted on the Product model for faster queries. These values are calculated from the Review collection and updated via a service (not by the model directly). This is a denormalization for performance.

6. **Soft Deletes Not Used**: Discontinued products are not soft-deleted (kept in DB with deleted flag). Instead, they have a `status: 'discontinued'` which allows for audit trails and historical queries.

7. **Category Reference**: Categories are referenced by ObjectId. If a category is deleted, the product's category reference becomes invalid. This should be handled by the service layer (e.g., move to "Uncategorized" or prevent category deletion if products exist).

8. **Migration Considerations**: If adding new fields later, ensure backward compatibility by using optional fields with sensible defaults (e.g., `metadata: {}`).

9. **Index Strategy**: Indexes are optimized for the most common queries:
   - Single field indexes for slug, sku (unique lookups)
   - Compound indexes for filtering and sorting (category + status, status + publishedAt, etc.)
   - Array index on tags for tag-based searches

10. **Authentication**: The `createdBy` and `updatedBy` fields should be populated by the controller using the authenticated user ID from middleware. The model should not set these directly.
