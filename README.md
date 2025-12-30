# Backend Engineering Intern – Case Study

Name: Anshu Babhure
Role: Backend Engineering Intern
Company: Bynry Inc

## 1. Introduction

This case study was completed to demonstrate my backend engineering skills, including API design, database modeling, and production-level problem solving. The focus of this assignment is to evaluate how backend systems should be designed for scalability, data consistency, and real-world business requirements.

The solution covers:
- Code review and debugging
- Corrected API implementation
- Scalable database schema design
- Business-driven API logic for low-stock alerts

---

## 2. Part 1 – Code Review & Debugging

### Issues Identified

While reviewing the provided API endpoint for product creation, I identified the following issues that could cause failures or inconsistencies in a production environment:

- Missing input validation for required fields
- SKU uniqueness not enforced
- Incorrect product-warehouse relationship
- Lack of transaction management
- No proper error or exception handling
- Price data type not handled safely
- Optional fields assumed to always exist

### Impact in Production

These issues can lead to:
- API crashes due to malformed requests
- Duplicate product entries
- Data inconsistency between products and inventory
- Financial calculation inaccuracies
- Poor user experience and difficult debugging

### Fixes Implemented

To resolve these issues:
- Input validation was added
- SKU uniqueness was enforced
- Product and warehouse dependency was removed
- Atomic database transactions were implemented
- Decimal was used for accurate price handling
- Exception handling with rollback was added
- Optional fields were safely handled

---

## 3. Corrected API Implementation (Product Creation)


from flask import request
from decimal import Decimal
from sqlalchemy.exc import IntegrityError

@app.route('/api/products', methods=['POST'])
def create_product():
    try:
        data = request.json

        # Validate required fields
        required_fields = ['name', 'sku', 'price']
        for field in required_fields:
            if field not in data:
                return {"error": f"{field} is required"}, 400

        # Enforce SKU uniqueness
        if Product.query.filter_by(sku=data['sku']).first():
            return {"error": "SKU already exists"}, 409

        # Create product
        product = Product(
            name=data['name'],
            sku=data['sku'],
            price=Decimal(data['price'])
        )

        db.session.add(product)
        db.session.flush()

        # Create inventory if warehouse data is provided
        if 'warehouse_id' in data and 'initial_quantity' in data:
            inventory = Inventory(
                product_id=product.id,
                warehouse_id=data['warehouse_id'],
                quantity=data['initial_quantity']
            )
            db.session.add(inventory)

        db.session.commit()

        return {
            "message": "Product created successfully",
            "product_id": product.id
        }, 201

    except IntegrityError:
        db.session.rollback()
        return {"error": "Database constraint violation"}, 400

    except Exception:
        db.session.rollback()
        return {"error": "Internal server error"}, 500

## 4. Part 2 – Database Design

### Design Goals
The database schema is designed to:
- Support multiple companies
- Allow multiple warehouses per company
- Store products across multiple warehouses
- Track inventory changes over time
- Support supplier relationships
- Enable bundled products

---

### Core Tables

**Companies**  
Stores company-level information.

**Warehouses**  
Each company can own multiple warehouses.

**Products**  
Stores product details. Products are not directly tied to warehouses.

**Inventory**  
Maps products to warehouses and tracks stock quantity per warehouse.

**Inventory History**  
Tracks every inventory change for audit and reporting purposes.

**Suppliers**  
Stores supplier details.

**Product Suppliers**  
Many-to-many mapping between products and suppliers.

**Product Bundles**  
Supports bundled or combo products.

---

### Key Design Decisions
- Products are decoupled from warehouses
- Inventory table manages product–warehouse relationships
- SKU uniqueness is enforced
- Decimal is used for financial accuracy
- Inventory history ensures auditability and traceability

---

### Assumptions
- SKU is globally unique
- Inventory is tracked per warehouse
- Products are company-specific
- Price precision is critical

---

### Questions for Product Team
- Is low-stock threshold defined per product or per warehouse?
- How is recent sales activity calculated?
- Are inventory updates manual, automated, or both?
- Can suppliers supply products to multiple companies?
- Should bundle pricing be calculated automatically or manually?



## 5. Part 3 – Low Stock Alerts API

### Problem Understanding
The goal of this API is to:
- Identify products running low on stock
- Work across multiple warehouses
- Consider only products with recent sales activity
- Include supplier details to assist reordering



### API Endpoint

    GET /api/companies/{company_id}/alerts/low-stock


### Business Assumptions
- Each product has a low_stock_threshold
- Recent sales refers to sales in the last 30 days
- One primary supplier exists per product
- Average daily sales is available for calculation

### API Logic
- Fetch inventory across all warehouses for a company
- Ignore products with zero recent sales
- Compare stock quantity against threshold
- Calculate days until stockout
- Include supplier details in the response

---

### Implementation


@app.route('/api/companies/<int:company_id>/alerts/low-stock', methods=['GET'])
def low_stock_alerts(company_id):
    alerts = []

    inventories = db.session.query(
        Inventory,
        Product,
        Warehouse,
        Supplier
    ).join(Product) \
     .join(Warehouse) \
     .join(ProductSupplier) \
     .join(Supplier) \
     .filter(Warehouse.company_id == company_id) \
     .all()

    for inventory, product, warehouse, supplier in inventories:
        threshold = product.low_stock_threshold
        avg_daily_sales = product.avg_daily_sales

        if avg_daily_sales == 0:
            continue

        days_until_stockout = inventory.quantity // avg_daily_sales

        if inventory.quantity <= threshold:
            alerts.append({
                "product_id": product.id,
                "product_name": product.name,
                "sku": product.sku,
                "warehouse_id": warehouse.id,
                "warehouse_name": warehouse.name,
                "current_stock": inventory.quantity,
                "threshold": threshold,
                "days_until_stockout": days_until_stockout,
                "supplier": {
                    "id": supplier.id,
                    "name": supplier.name,
                    "contact_email": supplier.contact_email
                }
            })

    return {
        "alerts": alerts,
        "total_alerts": len(alerts)
    }, 200

### Edge Cases Handled
- Products with zero sales are ignored
- Multiple warehouses are supported
- Division by zero is avoided
- Only company-specific data is returned

### Why This Approach
- Scalable for large datasets
- Business rules enforced at backend level
- Clean and readable logic
- Easy to extend with caching and background jobs
