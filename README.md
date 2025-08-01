##  Technical Issues

- `request.json` is used without checking if it exists.  
  → The code crashes if the request body is empty or not in JSON format.

- `price = data['price']` is not type-checked.  
  → If `price` is sent as a string instead of a number, it can cause bugs.

- SKU is not checked for uniqueness.  
  → This can lead to duplicate product entries in the system.

- No input validation.  
  → Required fields like `price`, `name`, and `sku` are not checked before processing.



##  Business Logic Issues

- SKU should be unique across the entire system.  

- Inventory is created even if the warehouse doesn’t exist.

- Inventory is created even if quantity is `0` or missing.  
  



##  Fixes

- Add a database-level uniqueness constraint for SKU.
- Use transactions to commit both product and inventory together atomically.
- Check if the provided `warehouse_id` exists before creating the inventory record.
- Provide default values for optional fields and validate required fields properly.


## corrected Version


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

module.exports = router;




