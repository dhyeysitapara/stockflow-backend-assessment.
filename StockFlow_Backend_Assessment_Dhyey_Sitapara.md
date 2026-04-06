# StockFlow Backend Assessment

**By: Dhyey Sitapara**

## Part 1: Code Review & Debugging

### Issues I Found

When reviewing the `create_product` endpoint, I noticed a few problems that could cause issues in production:

1. **Missing transaction rollback/atomicity**: Right now, the code commits the product first, then tries to create the inventory. If the inventory creation fails (maybe a DB connection issue or validation error), we'll be left with a product that has no inventory record. They should be committed together.
2. **`warehouse_id` on the Product model**: The instructions say products can be in multiple warehouses. Saving `warehouse_id` on the Product model forces a one-to-one relationship. The warehouse assignment should only live in the `Inventory` table.
3. **No input validation**: It just assumes `data['name']`, `data['sku']`, etc. are present. If someone sends an empty or partial JSON payload, it will throw a 500 server error (KeyError).
4. **Duplicate SKU handling**: If a user submits an SKU that already exists, SQLAlchemy will throw an `IntegrityError` instead of returning a clean HTTP 400 or 409 response.

### Corrected Code

Here is how I would fix these issues. I grouped the DB operations into a single transaction and added some basic validation.

```python
from flask import request, jsonify
from sqlalchemy.exc import IntegrityError
from decimal import Decimal

@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.json

    # Check for missing fields
    if not all(k in data for k in ['name', 'sku', 'warehouse_id']):
        return jsonify({"error": "name, sku, and warehouse_id are required"}), 400

    # Safely parse price, default to 0 if not provided
    try:
        price = Decimal(str(data.get('price', 0)))
    except:
        return jsonify({"error": "Invalid price format"}), 400

    try:
        # Create product (removed warehouse_id from here)
        product = Product(
            name=data['name'],
            sku=data['sku'],
            price=price
        )

        db.session.add(product)
        db.session.flush() # Get the product ID without committing yet

        # Create initial inventory mapping
        inventory = Inventory(
            product_id=product.id,
            warehouse_id=data['warehouse_id'],
            quantity=data.get('initial_quantity', 0)
        )

        db.session.add(inventory)

        # Commit everything together
        db.session.commit()
        return jsonify({"message": "Product created", "product_id": product.id}), 201

    except IntegrityError:
        db.session.rollback()
        return jsonify({"error": "SKU already exists or invalid warehouse ID."}), 409
    except Exception as e:
        db.session.rollback()
        return jsonify({"error": "Something went wrong"}), 500
```

_(Note: In a real app, I'd probably use Pydantic or Marshmallow for validation instead of doing it manually like this.)_

---

## Part 2: Database Design

### Questions for the Product Team

While designing this schema, I realized some requirements were missing. I'd ask the team:

- **Bundles**: Are bundles physically pre-packaged together, or is it just a virtual grouping where we deduct stock from individual items when a bundle is sold?
- **Suppliers**: Does a product only have one supplier, or can multiple suppliers provide the same product?
- **Logs**: Do we need to track _why_ inventory changed (e.g. sale, restock, damaged goods)? I assumed we do.

### Schema Design

I designed this using standard PostgreSQL syntax. I split the inventory state and the audit trail into two tables so it's easier to query current stock fast while still keeping a full history.

```sql
CREATE TABLE companies (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE warehouses (
    id SERIAL PRIMARY KEY,
    company_id INT REFERENCES companies(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    location VARCHAR(255)
);

CREATE TABLE suppliers (
    id SERIAL PRIMARY KEY,
    company_id INT REFERENCES companies(id) ON DELETE CASCADE,
    name VARCHAR(255),
    contact_email VARCHAR(255)
);

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    company_id INT REFERENCES companies(id) ON DELETE CASCADE,
    sku VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    is_bundle BOOLEAN DEFAULT FALSE,
    supplier_id INT REFERENCES suppliers(id) ON DELETE SET NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Keeps track of what items are inside a bundle
CREATE TABLE bundle_items (
    bundle_id INT REFERENCES products(id) ON DELETE CASCADE,
    product_id INT REFERENCES products(id) ON DELETE CASCADE,
    quantity INT DEFAULT 1,
    PRIMARY KEY (bundle_id, product_id)
);

-- Tracks current stock per warehouse
CREATE TABLE inventory (
    id SERIAL PRIMARY KEY,
    product_id INT REFERENCES products(id) ON DELETE CASCADE,
    warehouse_id INT REFERENCES warehouses(id) ON DELETE CASCADE,
    quantity INT DEFAULT 0,
    UNIQUE(product_id, warehouse_id)
);

-- Audit log for every time stock goes up or down
CREATE TABLE inventory_logs (
    id SERIAL PRIMARY KEY,
    inventory_id INT REFERENCES inventory(id) ON DELETE CASCADE,
    change_amount INT NOT NULL,
    reason VARCHAR(50), -- e.g., 'SALE', 'RESTOCK'
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## Part 3: API Implementation

### My Assumptions

1. Since thresholds vary by product type, I used a simple config dictionary for this MVP (e.g. standard=20, bundle=10).
2. For "recent sales activity", I'm looking for products that had a negative inventory change (a sale) in the last 30 days.
3. I used SQLAlchemy to build the query since it makes joining across this many tables a lot cleaner. I also filtered the warehouse by `company_id` to ensure tenants can't see each other's data.

### Implementation

```python
from flask import Blueprint, jsonify
from datetime import datetime, timedelta
from sqlalchemy.sql import func

alerts_bp = Blueprint('alerts', __name__)

THRESHOLDS = {
    False: 20, # standard product threshold
    True: 10   # bundle threshold
}

@alerts_bp.route('/api/companies/<int:company_id>/alerts/low-stock', methods=['GET'])
def get_low_stock_alerts(company_id):
    try:
        last_month = datetime.utcnow() - timedelta(days=30)

        # Get all inventory records for this specific company
        records = db.session.query(
            Inventory.id.label('inv_id'),
            Inventory.quantity,
            Product.id.label('prod_id'),
            Product.name,
            Product.sku,
            Product.is_bundle,
            Warehouse.id.label('wh_id'),
            Warehouse.name.label('wh_name'),
            Supplier.id.label('sup_id'),
            Supplier.name.label('sup_name'),
            Supplier.contact_email
        ).join(Product, Product.id == Inventory.product_id)\
         .join(Warehouse, Warehouse.id == Inventory.warehouse_id)\
         .outerjoin(Supplier, Supplier.id == Product.supplier_id)\
         .filter(Warehouse.company_id == company_id).all()

        # Find all sales in the last 30 days for these items
        recent_sales = dict(
            db.session.query(
                InventoryLog.inventory_id,
                func.sum(InventoryLog.change_amount)
            ).filter(
                InventoryLog.reason == 'SALE',
                InventoryLog.created_at >= last_month
            ).group_by(InventoryLog.inventory_id).all()
        )

        alerts = []

        for row in records:
            threshold = THRESHOLDS.get(row.is_bundle, 20)

            # Skip if stock is above threshold
            if row.quantity > threshold:
                continue

            # Skip if no recent sales
            sales_last_30_days = abs(recent_sales.get(row.inv_id, 0))
            if sales_last_30_days == 0:
                continue

            # Estimate how many days until stock runs out
            daily_sales_rate = sales_last_30_days / 30
            days_left = int(row.quantity / daily_sales_rate) if daily_sales_rate > 0 else 999

            alerts.append({
                "product_id": row.prod_id,
                "product_name": row.name,
                "sku": row.sku,
                "warehouse_id": row.wh_id,
                "warehouse_name": row.wh_name,
                "current_stock": row.quantity,
                "threshold": threshold,
                "days_until_stockout": days_left,
                "supplier": {
                    "id": row.sup_id,
                    "name": row.sup_name,
                    "contact_email": row.contact_email
                } if row.sup_id else None
            })

        return jsonify({"alerts": alerts, "total_alerts": len(alerts)})

    except Exception as e:
        print(f"Error fetching alerts: {e}")
        return jsonify({"error": "Failed to fetch alerts"}), 500

```
