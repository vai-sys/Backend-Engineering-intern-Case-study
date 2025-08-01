##  Technical Issues

- `request.json` is used without checking if it exists.  
  → The code crashes if the request body is empty or not in JSON format.

- `price = data['price']` is not type-checked.  
  → If `price` is sent as a string instead of a number, it can cause bugs.

- SKU is not checked for uniqueness.  
  → This can lead to duplicate product entries in the system.

- No input validation.  
  → Required fields like `price`, `name`, and `sku` are not checked before processing.

---

##  Business Logic Issues

- SKU should be unique across the entire system.  

- Inventory is created even if the warehouse doesn’t exist.

- Inventory is created even if quantity is `0` or missing.  
  

---

##  Fixes

- Add a database-level uniqueness constraint for SKU.
- Use transactions to commit both product and inventory together atomically.
- Check if the provided `warehouse_id` exists before creating the inventory record.
- Provide default values for optional fields and validate required fields properly.
