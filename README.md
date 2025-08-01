
## Part 1


## Technical Issues

- `request.json` is used without checking if it exists.  
  → The code crashes if the request body is empty or not in JSON format.

- `price = data['price']` is not type-checked.  
  → If `price` is sent as a string instead of a number, it can cause bugs.

- SKU is not checked for uniqueness.  
  → This can lead to duplicate product entries in the system.

- No input validation.  
  → Required fields like `price`, `name`, and `sku` are not checked before processing.


## Business Logic Issues

- SKU should be unique across the entire system.

- Inventory is created even if the warehouse doesn’t exist.

- Inventory is created even if quantity is `0` or missing.


## Fixes

- Add a database-level uniqueness constraint for SKU.
- Use transactions to commit both product and inventory together atomically.
- Check if the provided `warehouse_id` exists before creating the inventory record.
- Provide default values for optional fields and validate required fields properly.


## Corrected Version

```js
router.post('/api/products', async (req, res) => {
  const session = await mongoose.startSession();
  session.startTransaction();

  try {
    const { name, sku, price, warehouse_id, initial_quantity } = req.body;

    if (!name || !sku || price == null || !warehouse_id || initial_quantity == null) {
      return res.status(400).json({ error: 'Missing required fields' });
    }

    const existing = await Product.findOne({ sku }).session(session);
    if (existing) {
      return res.status(409).json({ error: 'SKU already exists. Must be unique.' });
    }

    const product = new Product({
      name,
      sku,
      price,
      warehouses: [warehouse_id]
    });

    await product.save({ session });

    const inventory = new Inventory({
      product: product._id,
      warehouse: warehouse_id,
      quantity: initial_quantity
    });

    await inventory.save({ session });

    await session.commitTransaction();
    session.endSession();

    return res.status(201).json({
      message: 'Product created successfully',
      product_id: product._id
    });

  } catch (error) {
    await session.abortTransaction();
    session.endSession();

    console.error('Transaction failed:', error);

    return res.status(500).json({
      error: error.message || 'Something went wrong while creating product'
    });
  }
});

```

## Issues and Their Impact in Production

1. **No validation**  
   → If request fields are not properly checked, it can result in HTTP 500 errors and a bad user experience.

2. **No proper error handling**  
   → If something goes wrong, the backend might crash or save incorrect data in the database.

3. **SKU not unique**  
   → If duplicate SKUs are allowed, it can cause confusion in inventory and incorrect reports.



## Part 2


# Companies
  | Column | Type         | Description         |
| ------ | ------------ | ------------------- |
| id     | SERIAL       | Primary key         |
| name   | VARCHAR(255) | Name of the company |


# Warehouses
| Column      | Type         | Description                    |
| ----------- | ------------ | ------------------------------ |
| id          | SERIAL       | Primary key                    |
| company\_id | INT          | Foreign key → `companies(id)`  |
| name        | VARCHAR(255) | Name of the warehouse          |
| location    | VARCHAR(255) | Address or location (optional) |


# Products


| Column     | Type         | Description                          |
| ---------- | ------------ | ------------------------------------ |
| id         | SERIAL       | Primary key                          |
| name       | VARCHAR(255) | Name of the product                  |
| sku        | VARCHAR(100) | Unique SKU identifier                |
| is\_bundle | BOOLEAN      | Indicates if the product is a bundle |


# product_bundles

| Column      | Type | Description                                   |
| ----------- | ---- | --------------------------------------------- |
| bundle\_id  | INT  | Foreign key → `products(id)` (the bundle)     |
| product\_id | INT  | Foreign key → `products(id)` (contained item) |
| quantity    | INT  | Number of contained items in the bundle       |


# suppliers
| Column | Type         | Description          |
| ------ | ------------ | -------------------- |
| id     | SERIAL       | Primary key          |
| name   | VARCHAR(255) | Name of the supplier |


# supplier_products

| Column       | Type | Description                   |
| ------------ | ---- | ----------------------------- |
| supplier\_id | INT  | Foreign key → `suppliers(id)` |
| product\_id  | INT  | Foreign key → `products(id)`  |


# inventory
| Column        | Type   | Description                    |
| ------------- | ------ | ------------------------------ |
| id            | SERIAL | Primary key                    |
| warehouse\_id | INT    | Foreign key → `warehouses(id)` |
| product\_id   | INT    | Foreign key → `products(id)`   |
| quantity      | INT    | Current quantity in stock      |



## Indexes and Constaints

sku in products should be UNIQUE for faster lookups.

Foreign keys ensure referential integrity.

Consider adding ON DELETE CASCADE where appropriate .

Indexes on warehouse_id, product_id in inventory for fast queries.


## questions to be asked to Product team
1. Can the products from the bundles be saled individually.
2.  Do suppliers have priority or different pricing?
3.  Do products need units ?


### Part 3

 ```js 

const express = require("express");
const router = express.Router();
const Product = require("../models/Product");
const Inventory = require("../models/Inventory");
const Sale = require("../models/Sale");
const Supplier = require("../models/Supplier");

router.get("/:companyId/alerts/low-stock", async (req, res) => {
  const { companyId } = req.params;
  const DAYS_LIMIT = 30;
  const thresholdDefault = 10;
  const dateLimit = new Date();
  dateLimit.setDate(dateLimit.getDate() - DAYS_LIMIT);

  try {
    // Step 1: Get recent sales
    const recentSales = await Sale.find({ saleDate: { $gte: dateLimit } });

    const soldProductIds = [...new Set(recentSales.map(s => s.productId.toString()))];

    // Step 2: Find products that are low in stock and were sold recently
    const lowStockInventories = await Inventory.find({ productId: { $in: soldProductIds } });

    const alerts = [];

    for (const inventory of lowStockInventories) {
      const product = await Product.findById(inventory.productId);
      if (!product || product.companyId !== companyId) continue;

      const threshold = product.threshold || thresholdDefault;

      if (inventory.quantity < threshold) {
        // Calculate avg daily sales
        const productSales = recentSales.filter(s => s.productId.toString() === product.id);
        const totalQty = productSales.reduce((sum, s) => sum + s.quantity, 0);
        const avgDailySales = totalQty / DAYS_LIMIT;
        const daysUntilStockout = avgDailySales > 0 ? Math.ceil(inventory.quantity / avgDailySales) : null;

        // Get supplier
        const supplier = await Supplier.findOne({ products: product._id });

        alerts.push({
          product_id: product.id,
          product_name: product.name,
          sku: product.sku,
          current_stock: inventory.quantity,
          threshold,
          days_until_stockout: daysUntilStockout,
          supplier: supplier
            ? {
                id: supplier.id,
                name: supplier.name,
                contact_email: supplier.contactEmail,
              }
            : null,
        });
      }
    }

    res.json({ alerts, total_alerts: alerts.length });

  } catch (err) {
    console.error(err.message);
    res.status(500).json({ error: "Server Error" });
  }
});

module.exports = router;




```


## Assumptions

-  We only care about **products that have been sold in the last 30 days** (these are considered "active").
-  Each product may have a field called `threshold`. If the **current stock** is less than this threshold, we consider it **low in stock**.
-  If a product doesn't have a `threshold`, we use a **default value **.
-  The API is called with a `company_id`, so we only show **alerts for that company’s warehouses and products**.

