# 3. API Design

## API Overview

**Base URL**: `https://api.medicalshop.com/v1`
**Authentication**: JWT Bearer Token
**Response Format**: JSON
**API Versioning**: URL-based (v1, v2, etc.)

## API Standards

### Request/Response Format

```json
// Standard Success Response
{
  "success": true,
  "data": { ... },
  "meta": {
    "timestamp": "2024-01-15T10:30:00Z",
    "requestId": "req_123456"
  }
}

// Standard Error Response
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request parameters",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      }
    ]
  },
  "meta": {
    "timestamp": "2024-01-15T10:30:00Z",
    "requestId": "req_123456"
  }
}

// Paginated Response
{
  "success": true,
  "data": [...],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "pages": 8,
    "hasNext": true,
    "hasPrev": false
  },
  "meta": { ... }
}
```

### HTTP Status Codes

| Code | Meaning | Use Case |
|------|---------|----------|
| 200 | OK | Successful GET, PUT, PATCH |
| 201 | Created | Successful POST creating resource |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Invalid request parameters |
| 401 | Unauthorized | Missing/invalid authentication |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource not found |
| 409 | Conflict | Duplicate/conflict (e.g., email exists) |
| 422 | Unprocessable Entity | Validation errors |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Error | Server error |
| 503 | Service Unavailable | Maintenance/service down |

### Common Headers

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
Content-Type: application/json
X-Request-ID: req_123456
X-API-Version: 1.0
X-Client-Version: 1.0.0
```

## Authentication Endpoints

### POST /auth/register
Register a new customer

```http
POST /auth/register
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "SecurePass123!",
  "first_name": "John",
  "last_name": "Doe",
  "phone": "+919876543210"
}
```

**Response (201)**:
```json
{
  "success": true,
  "data": {
    "id": "user_123",
    "email": "user@example.com",
    "first_name": "John",
    "last_name": "Doe",
    "token": "eyJhbGciOiJIUzI1NiIs...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
    "expiresIn": 3600
  }
}
```

### POST /auth/login
Login user

```http
POST /auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "SecurePass123!"
}
```

### POST /auth/refresh
Refresh JWT token

```http
POST /auth/refresh
Content-Type: application/json
Authorization: Bearer <refresh_token>

{}
```

### POST /auth/logout
Logout user (token blacklist)

```http
POST /auth/logout
Authorization: Bearer <token>

{}
```

### POST /auth/forgot-password
Request password reset

```http
POST /auth/forgot-password
Content-Type: application/json

{
  "email": "user@example.com"
}
```

### POST /auth/reset-password
Reset password with token

```http
POST /auth/reset-password
Content-Type: application/json

{
  "token": "reset_token_123",
  "newPassword": "NewPassword123!"
}
```

### POST /auth/verify-email
Verify email with OTP

```http
POST /auth/verify-email
Content-Type: application/json

{
  "email": "user@example.com",
  "otp": "123456"
}
```

## Customer Endpoints

### GET /customers/me
Get current customer profile

```http
GET /customers/me
Authorization: Bearer <token>
```

**Response (200)**:
```json
{
  "success": true,
  "data": {
    "id": "cust_123",
    "user": {
      "id": "user_123",
      "email": "user@example.com",
      "first_name": "John",
      "last_name": "Doe",
      "phone": "+919876543210"
    },
    "loyaltyPoints": 150,
    "totalSpent": 5000,
    "totalOrders": 3,
    "addresses": [...],
    "preferences": {...}
  }
}
```

### PUT /customers/me
Update customer profile

```http
PUT /customers/me
Authorization: Bearer <token>
Content-Type: application/json

{
  "first_name": "John",
  "last_name": "Doe",
  "phone": "+919876543210"
}
```

### GET /customers/me/addresses
List customer addresses

```http
GET /customers/me/addresses?type=home&isDefault=true
Authorization: Bearer <token>
```

### POST /customers/me/addresses
Create new address

```http
POST /customers/me/addresses
Authorization: Bearer <token>
Content-Type: application/json

{
  "addressType": "home",
  "streetAddress": "123 Main St",
  "city": "Mumbai",
  "stateProvince": "Maharashtra",
  "postalCode": "400001",
  "country": "India",
  "latitude": 19.0760,
  "longitude": 72.8777,
  "isDefault": true,
  "isBillingAddress": false
}
```

### PUT /customers/me/addresses/:id
Update address

### DELETE /customers/me/addresses/:id
Delete address

### GET /customers/me/preferences
Get user preferences

### PUT /customers/me/preferences
Update user preferences

```http
PUT /customers/me/preferences
Authorization: Bearer <token>
Content-Type: application/json

{
  "notificationEmail": true,
  "notificationSms": false,
  "notificationPush": true,
  "marketingEmails": false,
  "theme": "dark",
  "language": "en"
}
```

## Product Endpoints

### GET /products
List products with filtering

```http
GET /products?page=1&limit=20&category=vitamins&minPrice=100&maxPrice=1000&search=vitamin+c&sortBy=price&order=asc
```

**Response (200)**:
```json
{
  "success": true,
  "data": [
    {
      "id": "prod_123",
      "name": "Vitamin C 500mg",
      "slug": "vitamin-c-500mg",
      "description": "High potency vitamin C supplement",
      "category": {
        "id": "cat_123",
        "name": "Vitamins"
      },
      "brand": "HealthCare Pro",
      "sku": "VC500-001",
      "price": 299,
      "discountPercentage": 10,
      "finalPrice": 269.1,
      "rating": 4.5,
      "ratingCount": 120,
      "requiresPrescription": false,
      "images": [
        {
          "id": "img_123",
          "url": "https://cdn.example.com/product1.jpg",
          "altText": "Vitamin C bottle",
          "isPrimary": true
        }
      ],
      "inventory": {
        "quantityAvailable": 50,
        "reorderLevel": 10
      }
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 500,
    "pages": 25,
    "hasNext": true
  }
}
```

### GET /products/:id
Get product details

```http
GET /products/prod_123
```

### GET /products/:id/reviews
Get product reviews

```http
GET /products/prod_123/reviews?page=1&limit=10&sortBy=helpful
```

### POST /products/:id/reviews
Add product review (authenticated)

```http
POST /products/prod_123/reviews
Authorization: Bearer <token>
Content-Type: application/json

{
  "rating": 5,
  "title": "Excellent quality",
  "comment": "Great vitamin C supplement, good quality",
  "orderId": "order_123"
}
```

### GET /categories
List product categories

```http
GET /categories?includeSubcategories=true
```

### GET /search
Global search

```http
GET /search?q=vitamin&type=product&limit=20
```

## Cart Endpoints

### GET /cart
Get current cart

```http
GET /cart
Authorization: Bearer <token>
```

**Response (200)**:
```json
{
  "success": true,
  "data": {
    "id": "cart_123",
    "items": [
      {
        "id": "item_123",
        "product": {
          "id": "prod_123",
          "name": "Vitamin C 500mg",
          "price": 299
        },
        "quantity": 2,
        "unitPrice": 299,
        "subtotal": 598
      }
    ],
    "subtotal": 598,
    "tax": 107.64,
    "discount": 59.8,
    "total": 645.84,
    "itemCount": 2
  }
}
```

### POST /cart/items
Add item to cart

```http
POST /cart/items
Authorization: Bearer <token>
Content-Type: application/json

{
  "productId": "prod_123",
  "quantity": 2
}
```

### PUT /cart/items/:itemId
Update cart item quantity

```http
PUT /cart/items/item_123
Authorization: Bearer <token>
Content-Type: application/json

{
  "quantity": 3
}
```

### DELETE /cart/items/:itemId
Remove item from cart

```http
DELETE /cart/items/item_123
Authorization: Bearer <token>
```

### DELETE /cart
Clear entire cart

```http
DELETE /cart
Authorization: Bearer <token>
```

### POST /cart/coupon
Apply coupon code

```http
POST /cart/coupon
Authorization: Bearer <token>
Content-Type: application/json

{
  "couponCode": "SAVE10"
}
```

## Order Endpoints

### POST /orders
Create order

```http
POST /orders
Authorization: Bearer <token>
Content-Type: application/json

{
  "cartId": "cart_123",
  "deliveryAddressId": "addr_123",
  "paymentMethod": "credit_card",
  "notes": "Please deliver after 5 PM"
}
```

**Response (201)**:
```json
{
  "success": true,
  "data": {
    "id": "order_123",
    "orderNumber": "ORD-2024-001234",
    "customer": { ... },
    "items": [...],
    "status": "pending",
    "paymentStatus": "pending",
    "subtotal": 598,
    "tax": 107.64,
    "discountAmount": 59.8,
    "shippingCost": 50,
    "totalAmount": 695.84,
    "estimatedDeliveryDate": "2024-01-17",
    "createdAt": "2024-01-15T10:30:00Z"
  }
}
```

### GET /orders
List customer orders

```http
GET /orders?page=1&limit=10&status=delivered&sortBy=createdAt&order=desc
Authorization: Bearer <token>
```

### GET /orders/:id
Get order details

```http
GET /orders/order_123
Authorization: Bearer <token>
```

### PUT /orders/:id/cancel
Cancel order

```http
PUT /orders/order_123/cancel
Authorization: Bearer <token>
Content-Type: application/json

{
  "reason": "Changed my mind"
}
```

### GET /orders/:id/delivery
Get delivery tracking

```http
GET /orders/order_123/delivery
Authorization: Bearer <token>
```

**Response (200)**:
```json
{
  "success": true,
  "data": {
    "id": "delivery_123",
    "orderId": "order_123",
    "status": "in_transit",
    "deliveryPartner": {
      "name": "John Smith",
      "phone": "+919876543210",
      "rating": 4.8
    },
    "estimatedDeliveryDate": "2024-01-17",
    "actualDeliveryDate": null,
    "currentLocation": {
      "latitude": 19.0915,
      "longitude": 72.8984
    },
    "route": [
      {
        "latitude": 19.0760,
        "longitude": 72.8777,
        "timestamp": "2024-01-15T10:30:00Z"
      }
    ]
  }
}
```

## Payment Endpoints

### POST /payments/process
Process payment

```http
POST /payments/process
Authorization: Bearer <token>
Content-Type: application/json

{
  "orderId": "order_123",
  "paymentMethod": "credit_card",
  "amount": 695.84,
  "cardToken": "tok_123456",
  "billingAddressId": "addr_123"
}
```

### GET /payments/:id
Get payment details

```http
GET /payments/pay_123
Authorization: Bearer <token>
```

### POST /payments/:id/refund
Request refund

```http
POST /payments/pay_123/refund
Authorization: Bearer <token>
Content-Type: application/json

{
  "reason": "Order cancelled",
  "amount": 695.84
}
```

## Admin Endpoints

### GET /admin/dashboard
Admin dashboard metrics

```http
GET /admin/dashboard?dateRange=30days
Authorization: Bearer <admin_token>
```

**Response (200)**:
```json
{
  "success": true,
  "data": {
    "totalRevenue": 500000,
    "totalOrders": 1250,
    "totalCustomers": 500,
    "averageOrderValue": 400,
    "conversionRate": 3.5,
    "topProducts": [...],
    "orderTrend": [
      {
        "date": "2024-01-01",
        "orders": 45,
        "revenue": 18000
      }
    ],
    "pendingOrders": 23,
    "lowStockProducts": [...]
  }
}
```

### GET /admin/users
List all users

```http
GET /admin/users?page=1&limit=20&role=customer&status=active&search=john
Authorization: Bearer <admin_token>
```

### POST /admin/users/:id/suspend
Suspend user account

```http
POST /admin/users/user_123/suspend
Authorization: Bearer <admin_token>
Content-Type: application/json

{
  "reason": "Suspicious activity"
}
```

### GET /admin/inventory
Manage inventory

```http
GET /admin/inventory?lowStockOnly=true&sortBy=quantity
Authorization: Bearer <admin_token>
```

### PUT /admin/inventory/:productId
Update inventory

```http
PUT /admin/inventory/prod_123
Authorization: Bearer <admin_token>
Content-Type: application/json

{
  "quantityAvailable": 100,
  "reorderLevel": 10,
  "reorderQuantity": 50
}
```

### GET /admin/reports
Generate reports

```http
GET /admin/reports?type=sales&dateFrom=2024-01-01&dateTo=2024-01-31&format=pdf
Authorization: Bearer <admin_token>
```

### POST /admin/products
Create new product

```http
POST /admin/products
Authorization: Bearer <admin_token>
Content-Type: application/json

{
  "name": "New Product",
  "description": "Product description",
  "categoryId": "cat_123",
  "brand": "Brand Name",
  "sku": "SKU-001",
  "price": 299,
  "costPrice": 150,
  "requiresPrescription": false
}
```

## Rate Limiting

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1642248000
```

**Limits**:
- Authenticated users: 1000 requests/hour
- Anonymous users: 100 requests/hour
- Login endpoint: 5 attempts/5 minutes

## Error Codes

| Code | Message | Status |
|------|---------|--------|
| VALIDATION_ERROR | Request validation failed | 422 |
| AUTHENTICATION_FAILED | Invalid credentials | 401 |
| UNAUTHORIZED | Insufficient permissions | 403 |
| NOT_FOUND | Resource not found | 404 |
| CONFLICT | Resource already exists | 409 |
| RATE_LIMIT_EXCEEDED | Too many requests | 429 |
| INTERNAL_ERROR | Internal server error | 500 |
| PRODUCT_OUT_OF_STOCK | Product not available | 400 |
| INVALID_COUPON | Coupon code invalid/expired | 400 |
| PAYMENT_FAILED | Payment processing failed | 400 |

## API Documentation Tools

- **Swagger/OpenAPI**: https://api.medicalshop.com/swagger
- **Postman Collection**: Available in `/docs/postman-collection.json`
- **ReDoc**: https://api.medicalshop.com/redoc

## Backward Compatibility

- Deprecated endpoints marked with deprecation notice header
- Minimum 6-month notice before endpoint removal
- API versioning maintained for 2 versions minimum
- Client migration guide provided for breaking changes
