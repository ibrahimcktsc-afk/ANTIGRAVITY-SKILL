# 2. Database Schema

## Database Overview

**Database Type**: PostgreSQL 14+
**ORM**: TypeORM / Sequelize
**Connection Pooling**: PgBouncer
**Replication**: Master-Slave setup for high availability

## Entity Relationship Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      USERS TABLE                             │
│  ├─ id (PK)                                                  │
│  ├─ email (UNIQUE)                                           │
│  ├─ password_hash                                            │
│  ├─ first_name, last_name                                    │
│  ├─ phone                                                     │
│  ├─ role (admin, shop_manager, customer)                     │
│  ├─ status (active, inactive, suspended)                     │
│  ├─ created_at, updated_at                                   │
│  └─ deleted_at (soft delete)                                 │
└──────────────────────┬──────────────────────────────────────┘
                       │
          ┌────────────┼────────────┐
          │            │            │
    ┌─────▼──────┐ ┌──▼──────┐ ┌──▼──────┐
    │ CUSTOMERS  │ │ PAYMENTS│ │ ORDERS  │
    └─────┬──────┘ └─────────┘ └────┬────┘
          │                         │
    ┌─────▼──────────┐         ┌────▼────────┐
    │ ADDRESSES      │         │ ORDER_ITEMS │
    │ PREFERENCES    │         └─────────────┘
    │ WISHLIST       │
    └────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   PRODUCTS TABLE                             │
│  ├─ id (PK)                                                  │
│  ├─ name                                                     │
│  ├─ description                                              │
│  ├─ sku (UNIQUE)                                             │
│  ├─ category_id (FK)                                         │
│  ├─ price                                                    │
│  ├─ cost_price                                               │
│  ├─ status (active, inactive, discontinued)                  │
│  └─ created_at, updated_at                                   │
└──────────────────────┬──────────────────────────────────────┘
          ┌────────────┼────────────┐
          │            │            │
    ┌─────▼──────┐ ┌──▼──────┐ ┌──▼──────────┐
    │ CATEGORIES │ │ INVENTORY│ │ PRODUCT_IMG │
    └────────────┘ └─────────┘ └─────────────┘
```

## Detailed Schema

### 1. Users Table

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  first_name VARCHAR(100),
  last_name VARCHAR(100),
  phone VARCHAR(20),
  profile_image_url VARCHAR(500),
  role ENUM('admin', 'shop_manager', 'delivery_partner', 'customer') DEFAULT 'customer',
  status ENUM('active', 'inactive', 'suspended') DEFAULT 'active',
  email_verified BOOLEAN DEFAULT FALSE,
  email_verified_at TIMESTAMP,
  phone_verified BOOLEAN DEFAULT FALSE,
  phone_verified_at TIMESTAMP,
  last_login_at TIMESTAMP,
  is_2fa_enabled BOOLEAN DEFAULT FALSE,
  two_fa_secret VARCHAR(255),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  CONSTRAINT email_format CHECK (email ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$')
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_phone ON users(phone);
CREATE INDEX idx_users_status ON users(status);
```

### 2. Customers Table

```sql
CREATE TABLE customers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,
  loyalty_points INT DEFAULT 0,
  total_spent DECIMAL(15, 2) DEFAULT 0,
  total_orders INT DEFAULT 0,
  preferred_language VARCHAR(5) DEFAULT 'en',
  newsletter_subscribed BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_customers_user_id ON customers(user_id);
CREATE INDEX idx_customers_loyalty_points ON customers(loyalty_points);
```

### 3. Addresses Table

```sql
CREATE TABLE addresses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  address_type ENUM('home', 'work', 'other') DEFAULT 'home',
  street_address VARCHAR(255) NOT NULL,
  city VARCHAR(100) NOT NULL,
  state_province VARCHAR(100) NOT NULL,
  postal_code VARCHAR(20) NOT NULL,
  country VARCHAR(100) NOT NULL DEFAULT 'India',
  latitude DECIMAL(10, 8),
  longitude DECIMAL(11, 8),
  is_default BOOLEAN DEFAULT FALSE,
  is_billing_address BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_addresses_user_id ON addresses(user_id);
CREATE INDEX idx_addresses_coordinates ON addresses USING GIST (ll_to_earth(latitude, longitude));
```

### 4. Categories Table

```sql
CREATE TABLE categories (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL UNIQUE,
  slug VARCHAR(100) NOT NULL UNIQUE,
  description TEXT,
  parent_category_id UUID REFERENCES categories(id),
  image_url VARCHAR(500),
  is_active BOOLEAN DEFAULT TRUE,
  display_order INT DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_categories_parent_id ON categories(parent_category_id);
CREATE INDEX idx_categories_slug ON categories(slug);
```

### 5. Products Table

```sql
CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  slug VARCHAR(255) NOT NULL UNIQUE,
  description TEXT,
  detailed_description TEXT,
  category_id UUID NOT NULL REFERENCES categories(id),
  brand VARCHAR(100),
  sku VARCHAR(50) NOT NULL UNIQUE,
  barcode VARCHAR(50),
  price DECIMAL(12, 2) NOT NULL,
  cost_price DECIMAL(12, 2),
  discount_percentage DECIMAL(5, 2) DEFAULT 0,
  rating DECIMAL(3, 2) DEFAULT 0,
  rating_count INT DEFAULT 0,
  status ENUM('active', 'inactive', 'discontinued') DEFAULT 'active',
  requires_prescription BOOLEAN DEFAULT FALSE,
  expiry_date DATE,
  manufacturer VARCHAR(255),
  batch_number VARCHAR(100),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP
);

CREATE INDEX idx_products_category_id ON products(category_id);
CREATE INDEX idx_products_slug ON products(slug);
CREATE INDEX idx_products_sku ON products(sku);
CREATE INDEX idx_products_status ON products(status);
CREATE INDEX idx_products_search ON products USING gin(to_tsvector('english', name || ' ' || description));
```

### 6. Product Images Table

```sql
CREATE TABLE product_images (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
  image_url VARCHAR(500) NOT NULL,
  alt_text VARCHAR(255),
  is_primary BOOLEAN DEFAULT FALSE,
  display_order INT DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_product_images_product_id ON product_images(product_id);
```

### 7. Inventory Table

```sql
CREATE TABLE inventory (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id UUID NOT NULL UNIQUE REFERENCES products(id) ON DELETE CASCADE,
  quantity_available INT DEFAULT 0,
  quantity_reserved INT DEFAULT 0,
  quantity_damaged INT DEFAULT 0,
  reorder_level INT DEFAULT 10,
  reorder_quantity INT DEFAULT 50,
  warehouse_location VARCHAR(100),
  last_restock_date TIMESTAMP,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_inventory_product_id ON inventory(product_id);
CREATE INDEX idx_inventory_quantity_available ON inventory(quantity_available);
```

### 8. Inventory History Table

```sql
CREATE TABLE inventory_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id UUID NOT NULL REFERENCES products(id),
  transaction_type ENUM('in', 'out', 'adjust', 'damage') NOT NULL,
  quantity INT NOT NULL,
  reference_id UUID,
  reference_type VARCHAR(50),
  notes TEXT,
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_inventory_history_product_id ON inventory_history(product_id);
CREATE INDEX idx_inventory_history_created_at ON inventory_history(created_at);
```

### 9. Orders Table

```sql
CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_number VARCHAR(50) NOT NULL UNIQUE,
  customer_id UUID NOT NULL REFERENCES customers(id),
  delivery_address_id UUID REFERENCES addresses(id),
  status ENUM('pending', 'confirmed', 'processing', 'shipped', 'delivered', 'cancelled', 'returned') DEFAULT 'pending',
  payment_status ENUM('pending', 'completed', 'failed', 'refunded') DEFAULT 'pending',
  subtotal DECIMAL(15, 2) NOT NULL,
  tax_amount DECIMAL(15, 2) DEFAULT 0,
  discount_amount DECIMAL(15, 2) DEFAULT 0,
  shipping_cost DECIMAL(10, 2) DEFAULT 0,
  total_amount DECIMAL(15, 2) NOT NULL,
  notes TEXT,
  special_instructions TEXT,
  estimated_delivery_date DATE,
  actual_delivery_date DATE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  confirmed_at TIMESTAMP,
  shipped_at TIMESTAMP,
  delivered_at TIMESTAMP
);

CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_order_number ON orders(order_number);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_payment_status ON orders(payment_status);
CREATE INDEX idx_orders_created_at ON orders(created_at);
```

### 10. Order Items Table

```sql
CREATE TABLE order_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id UUID NOT NULL REFERENCES products(id),
  quantity INT NOT NULL,
  unit_price DECIMAL(12, 2) NOT NULL,
  discount_amount DECIMAL(12, 2) DEFAULT 0,
  tax_amount DECIMAL(12, 2) DEFAULT 0,
  subtotal DECIMAL(15, 2) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);
```

### 11. Payments Table

```sql
CREATE TABLE payments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  payment_method ENUM('credit_card', 'debit_card', 'upi', 'net_banking', 'wallet', 'cod') NOT NULL,
  amount DECIMAL(15, 2) NOT NULL,
  status ENUM('pending', 'completed', 'failed', 'cancelled') DEFAULT 'pending',
  gateway_name VARCHAR(100),
  gateway_transaction_id VARCHAR(255),
  receipt_number VARCHAR(100),
  notes TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  completed_at TIMESTAMP
);

CREATE INDEX idx_payments_order_id ON payments(order_id);
CREATE INDEX idx_payments_status ON payments(status);
```

### 12. Deliveries Table

```sql
CREATE TABLE deliveries (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL UNIQUE REFERENCES orders(id),
  delivery_partner_id UUID REFERENCES users(id),
  status ENUM('pending', 'assigned', 'picked_up', 'in_transit', 'delivered', 'failed', 'returned') DEFAULT 'pending',
  estimated_delivery_date DATE,
  actual_delivery_date DATE,
  current_latitude DECIMAL(10, 8),
  current_longitude DECIMAL(11, 8),
  delivery_notes TEXT,
  signature_image_url VARCHAR(500),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_deliveries_order_id ON deliveries(order_id);
CREATE INDEX idx_deliveries_delivery_partner_id ON deliveries(delivery_partner_id);
CREATE INDEX idx_deliveries_status ON deliveries(status);
```

### 13. Notifications Table

```sql
CREATE TABLE notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  type ENUM('order_status', 'payment', 'delivery', 'promotion', 'system') NOT NULL,
  title VARCHAR(255) NOT NULL,
  message TEXT NOT NULL,
  data JSONB,
  is_read BOOLEAN DEFAULT FALSE,
  read_at TIMESTAMP,
  channel ENUM('email', 'sms', 'push', 'in_app') DEFAULT 'in_app',
  status ENUM('pending', 'sent', 'failed') DEFAULT 'pending',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_notifications_user_id ON notifications(user_id);
CREATE INDEX idx_notifications_is_read ON notifications(is_read);
CREATE INDEX idx_notifications_created_at ON notifications(created_at);
```

### 14. Reviews & Ratings Table

```sql
CREATE TABLE reviews (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
  customer_id UUID NOT NULL REFERENCES customers(id),
  order_id UUID REFERENCES orders(id),
  rating INT NOT NULL CHECK (rating BETWEEN 1 AND 5),
  title VARCHAR(255),
  comment TEXT,
  is_verified_purchase BOOLEAN DEFAULT TRUE,
  helpful_count INT DEFAULT 0,
  unhelpful_count INT DEFAULT 0,
  status ENUM('pending', 'approved', 'rejected') DEFAULT 'pending',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_reviews_product_id ON reviews(product_id);
CREATE INDEX idx_reviews_customer_id ON reviews(customer_id);
```

### 15. Audit Log Table

```sql
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  entity_type VARCHAR(100) NOT NULL,
  entity_id UUID NOT NULL,
  action ENUM('create', 'update', 'delete', 'view') NOT NULL,
  old_values JSONB,
  new_values JSONB,
  ip_address INET,
  user_agent TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_entity ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at);
```

### 16. Preferences Table

```sql
CREATE TABLE user_preferences (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,
  notification_email BOOLEAN DEFAULT TRUE,
  notification_sms BOOLEAN DEFAULT TRUE,
  notification_push BOOLEAN DEFAULT TRUE,
  marketing_emails BOOLEAN DEFAULT TRUE,
  theme ENUM('light', 'dark') DEFAULT 'light',
  language VARCHAR(10) DEFAULT 'en',
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_user_preferences_user_id ON user_preferences(user_id);
```

## Indexes Strategy

### Performance Indexes
- Foreign keys: All FK relationships indexed
- Status fields: Frequently filtered status columns
- Timestamps: Date range queries
- Full-text search: Product names and descriptions
- Geospatial: Address coordinates for location-based queries

### Partitioning Strategy
- Orders table: Partitioned by month
- Notifications: Partitioned by year
- Audit logs: Partitioned by quarter

## Data Integrity Constraints

1. **Referential Integrity**: All FKs enforced with CASCADE delete where appropriate
2. **Check Constraints**: Valid enum values, phone format, price ranges
3. **Unique Constraints**: Email, SKU, Order number, Product slug
4. **Default Values**: Timestamps, boolean flags, status enums

## Backup Strategy

- Full backup: Weekly
- Incremental backup: Daily
- Transaction logs: Continuous archiving
- Backup location: Separate region
- Recovery time objective: < 4 hours
- Recovery point objective: < 15 minutes

## Query Optimization

```sql
-- Examples of optimized queries

-- Get customer orders with items
SELECT o.id, o.order_number, o.status, 
       COUNT(oi.id) as item_count, 
       SUM(oi.quantity) as total_quantity
FROM orders o
LEFT JOIN order_items oi ON o.id = oi.order_id
WHERE o.customer_id = $1
GROUP BY o.id
ORDER BY o.created_at DESC;

-- Product search with inventory
SELECT p.*, i.quantity_available
FROM products p
JOIN inventory i ON p.id = i.product_id
WHERE p.status = 'active'
  AND i.quantity_available > 0
  AND to_tsvector('english', p.name || ' ' || COALESCE(p.description, '')) 
      @@ plainto_tsquery('english', $1)
LIMIT 20;
```

## Migration Strategy

- Sequelize migrations or TypeORM migrations
- Version control for all schema changes
- Automated backup before migrations
- Rollback scripts for all changes
- Schema validation tests
