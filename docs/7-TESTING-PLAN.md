# 7. Testing Plan

## Testing Strategy

**Approach**: Shift-left testing with emphasis on automation
**Coverage Goal**: > 80% code coverage
**Test Pyramid**:
- Unit Tests: 70%
- Integration Tests: 20%
- E2E Tests: 10%

## Test Pyramid

```
        ┌─────────────┐
        │   E2E (10%) │ (Slow, expensive, full system)
        ├─────────────┤
        │Integration  │ (Medium speed, component interaction)
        │  Tests(20%) │
        ├──────────────┤
        │ Unit Tests   │ (Fast, isolated, cheap)
        │   (70%)      │
        └──────────────┘
```

## 1. Unit Tests

### Testing Framework

**Stack**:
- Jest (test runner)
- @testing-library/react (component testing)
- Sinon (spies, stubs, mocks)
- Faker (test data generation)

### Backend Unit Tests

#### Example: User Service Tests

```javascript
// src/services/__tests__/userService.test.js
const UserService = require('../userService');
const User = require('../../models/User');
const bcrypt = require('bcrypt');

jest.mock('../../models/User');
jest.mock('bcrypt');

describe('UserService', () => {
  afterEach(() => {
    jest.clearAllMocks();
  });

  describe('createUser', () => {
    it('should create a user with hashed password', async () => {
      const userData = {
        email: 'test@example.com',
        password: 'SecurePass123!',
        firstName: 'John',
        lastName: 'Doe'
      };

      bcrypt.hash.mockResolvedValue('hashedPassword');
      User.create.mockResolvedValue({
        id: 'user_123',
        ...userData,
        password: 'hashedPassword'
      });

      const result = await UserService.createUser(userData);

      expect(bcrypt.hash).toHaveBeenCalledWith(userData.password, 12);
      expect(User.create).toHaveBeenCalledWith({
        ...userData,
        password: 'hashedPassword'
      });
      expect(result.id).toBe('user_123');
    });

    it('should throw error for invalid email', async () => {
      const userData = {
        email: 'invalid-email',
        password: 'SecurePass123!'
      };

      await expect(UserService.createUser(userData))
        .rejects
        .toThrow('Invalid email format');
    });

    it('should throw error if user already exists', async () => {
      const userData = {
        email: 'existing@example.com',
        password: 'SecurePass123!'
      };

      User.findOne.mockResolvedValue({ id: 'user_123' });

      await expect(UserService.createUser(userData))
        .rejects
        .toThrow('User already exists');
    });
  });

  describe('authenticateUser', () => {
    it('should return user token on successful login', async () => {
      const user = {
        id: 'user_123',
        email: 'test@example.com',
        password: 'hashedPassword'
      };

      User.findOne.mockResolvedValue(user);
      bcrypt.compare.mockResolvedValue(true);

      const result = await UserService.authenticateUser(
        'test@example.com',
        'SecurePass123!'
      );

      expect(result.token).toBeDefined();
      expect(result.user.id).toBe('user_123');
    });

    it('should throw error for invalid password', async () => {
      const user = {
        id: 'user_123',
        email: 'test@example.com',
        password: 'hashedPassword'
      };

      User.findOne.mockResolvedValue(user);
      bcrypt.compare.mockResolvedValue(false);

      await expect(UserService.authenticateUser(
        'test@example.com',
        'WrongPassword'
      )).rejects.toThrow('Invalid credentials');
    });
  });
});
```

### Frontend Component Unit Tests

#### Example: Product Card Component Tests

```javascript
// src/components/__tests__/ProductCard.test.jsx
import { render, screen, fireEvent } from '@testing-library/react';
import { Provider } from 'react-redux';
import { configureStore } from '@reduxjs/toolkit';
import ProductCard from '../ProductCard';
import cartSlice from '../../store/slices/cartSlice';

describe('ProductCard', () => {
  let store;

  beforeEach(() => {
    store = configureStore({
      reducer: {
        cart: cartSlice
      }
    });
  });

  const mockProduct = {
    id: 'prod_123',
    name: 'Vitamin C 500mg',
    price: 299,
    discountPercentage: 10,
    rating: 4.5,
    ratingCount: 120,
    image: {
      url: 'https://example.com/product.jpg',
      altText: 'Product image'
    },
    inventory: {
      quantityAvailable: 50
    }
  };

  it('should render product information', () => {
    render(
      <Provider store={store}>
        <ProductCard product={mockProduct} />
      </Provider>
    );

    expect(screen.getByText('Vitamin C 500mg')).toBeInTheDocument();
    expect(screen.getByText('₹299')).toBeInTheDocument();
    expect(screen.getByText(/10% off/i)).toBeInTheDocument();
  });

  it('should display rating', () => {
    render(
      <Provider store={store}>
        <ProductCard product={mockProduct} />
      </Provider>
    );

    expect(screen.getByText('4.5')).toBeInTheDocument();
    expect(screen.getByText(/120 reviews/i)).toBeInTheDocument();
  });

  it('should add product to cart when button clicked', () => {
    render(
      <Provider store={store}>
        <ProductCard product={mockProduct} />
      </Provider>
    );

    const addButton = screen.getByRole('button', { name: /add to cart/i });
    fireEvent.click(addButton);

    const state = store.getState();
    expect(state.cart.items).toHaveLength(1);
    expect(state.cart.items[0].productId).toBe('prod_123');
  });

  it('should show out of stock message', () => {
    const outOfStockProduct = {
      ...mockProduct,
      inventory: { quantityAvailable: 0 }
    };

    render(
      <Provider store={store}>
        <ProductCard product={outOfStockProduct} />
      </Provider>
    );

    expect(screen.getByText(/out of stock/i)).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /add to cart/i }))
      .toBeDisabled();
  });
});
```

## 2. Integration Tests

### API Integration Tests

#### Example: Order API Tests

```javascript
// src/services/__tests__/orderAPI.integration.test.js
const request = require('supertest');
const app = require('../../app');
const User = require('../../models/User');
const Order = require('../../models/Order');
const Product = require('../../models/Product');
const db = require('../../db');

describe('Order API Integration Tests', () => {
  let authToken;
  let testUser;
  let testProduct;
  let testCart;

  beforeAll(async () => {
    // Setup test database
    await db.connect();
    await db.seed('test');
  });

  afterAll(async () => {
    await db.disconnect();
  });

  beforeEach(async () => {
    // Create test user
    testUser = await User.create({
      email: 'test@example.com',
      password: 'hashedPassword',
      firstName: 'Test',
      lastName: 'User'
    });

    // Login and get token
    const loginRes = await request(app)
      .post('/api/v1/auth/login')
      .send({
        email: 'test@example.com',
        password: 'TestPassword123!'
      });

    authToken = loginRes.body.data.token;

    // Create test product
    testProduct = await Product.create({
      name: 'Test Product',
      price: 299,
      sku: 'TEST-001'
    });
  });

  describe('POST /orders', () => {
    it('should create order with valid data', async () => {
      const orderData = {
        cartId: 'cart_123',
        deliveryAddressId: 'addr_123',
        paymentMethod: 'credit_card'
      };

      const res = await request(app)
        .post('/api/v1/orders')
        .set('Authorization', `Bearer ${authToken}`)
        .send(orderData);

      expect(res.status).toBe(201);
      expect(res.body.success).toBe(true);
      expect(res.body.data.orderNumber).toBeDefined();
      expect(res.body.data.status).toBe('pending');
    });

    it('should validate required fields', async () => {
      const invalidOrderData = {
        deliveryAddressId: 'addr_123'
        // Missing cartId and paymentMethod
      };

      const res = await request(app)
        .post('/api/v1/orders')
        .set('Authorization', `Bearer ${authToken}`)
        .send(invalidOrderData);

      expect(res.status).toBe(422);
      expect(res.body.success).toBe(false);
      expect(res.body.error.code).toBe('VALIDATION_ERROR');
    });

    it('should require authentication', async () => {
      const orderData = {
        cartId: 'cart_123',
        deliveryAddressId: 'addr_123',
        paymentMethod: 'credit_card'
      };

      const res = await request(app)
        .post('/api/v1/orders')
        .send(orderData);

      expect(res.status).toBe(401);
      expect(res.body.success).toBe(false);
    });
  });

  describe('GET /orders/:id', () => {
    it('should return order details for authorized user', async () => {
      const order = await Order.create({
        customerId: testUser.id,
        orderNumber: 'ORD-123',
        totalAmount: 299,
        status: 'pending'
      });

      const res = await request(app)
        .get(`/api/v1/orders/${order.id}`)
        .set('Authorization', `Bearer ${authToken}`);

      expect(res.status).toBe(200);
      expect(res.body.data.orderNumber).toBe('ORD-123');
      expect(res.body.data.totalAmount).toBe(299);
    });

    it('should prevent unauthorized access to other orders', async () => {
      const otherUser = await User.create({
        email: 'other@example.com',
        password: 'hashedPassword'
      });

      const order = await Order.create({
        customerId: otherUser.id,
        orderNumber: 'ORD-456',
        totalAmount: 499
      });

      const res = await request(app)
        .get(`/api/v1/orders/${order.id}`)
        .set('Authorization', `Bearer ${authToken}`);

      expect(res.status).toBe(403);
    });
  });
});
```

## 3. End-to-End (E2E) Tests

### E2E Testing Framework

**Tools**:
- Cypress (recommended) or Playwright
- Percy (visual regression)
- Lighthouse (performance)

### Cypress Test Suite

```javascript
// cypress/e2e/checkout.cy.js
describe('Checkout Flow', () => {
  beforeEach(() => {
    cy.visit('http://localhost:3000');
    cy.login('test@example.com', 'TestPassword123!');
  });

  it('should complete full checkout flow', () => {
    // Browse products
    cy.visit('http://localhost:3000/products');
    cy.get('[data-testid="product-card"]').first().click();

    // Add to cart
    cy.get('[data-testid="quantity-input"]').clear().type('2');
    cy.get('[data-testid="add-to-cart-btn"]').click();
    cy.get('[data-testid="cart-notification"]')
      .should('contain', 'Added to cart');

    // View cart
    cy.get('[data-testid="cart-icon"]').click();
    cy.url().should('include', '/cart');
    cy.get('[data-testid="cart-item"]').should('have.length', 1);

    // Checkout
    cy.get('[data-testid="checkout-btn"]').click();
    cy.url().should('include', '/checkout');

    // Fill shipping address
    cy.get('[data-testid="address-select"]')
      .select('addr_123');

    // Select payment method
    cy.get('input[value="credit_card"]').check();

    // Complete order
    cy.get('[data-testid="place-order-btn"]').click();

    // Verify order confirmation
    cy.url().should('include', '/order-confirmation');
    cy.get('[data-testid="order-number"]')
      .should('contain', 'ORD-');
  });

  it('should handle payment failures gracefully', () => {
    cy.visit('http://localhost:3000/cart');
    cy.get('[data-testid="checkout-btn"]').click();

    // Simulate payment failure
    cy.intercept('POST', '/api/v1/payments/process', {
      statusCode: 400,
      body: {
        success: false,
        error: {
          code: 'PAYMENT_FAILED',
          message: 'Card declined'
        }
      }
    });

    cy.get('[data-testid="place-order-btn"]').click();
    cy.get('[data-testid="error-message"]')
      .should('contain', 'Card declined');
  });
});
```

## 4. Performance Testing

### Load Testing with k6

```javascript
// tests/performance/load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },   // Ramp up
    { duration: '5m', target: 100 },   // Stay at 100
    { duration: '2m', target: 0 }      // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    http_req_failed: ['rate<0.1']
  }
};

export default function () {
  // Test product list endpoint
  const res = http.get('https://api.medicalshop.com/v1/products?page=1&limit=20');
  
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
    'valid response': (r) => {
      const body = JSON.parse(r.body);
      return body.success && body.data.length > 0;
    }
  });

  sleep(1);

  // Test product detail endpoint
  const productRes = http.get(
    'https://api.medicalshop.com/v1/products/prod_123'
  );
  
  check(productRes, {
    'product detail status 200': (r) => r.status === 200,
    'product has required fields': (r) => {
      const body = JSON.parse(r.body);
      return body.data.name && body.data.price;
    }
  });

  sleep(1);
}
```

## 5. Security Testing

### OWASP Security Test Suite

```javascript
// tests/security/security.test.js
const axios = require('axios');

describe('Security Tests', () => {
  const apiUrl = 'https://api.medicalshop.com/v1';

  it('should enforce HTTPS', async () => {
    const response = await axios.get(`http://api.medicalshop.com/v1/health`)
      .catch(err => err.response);

    expect(response.status).toBe(301);
    expect(response.headers.location).toMatch(/^https:\/\//);
  });

  it('should have security headers', async () => {
    const response = await axios.get(`${apiUrl}/health`);

    expect(response.headers['x-content-type-options']).toBe('nosniff');
    expect(response.headers['x-frame-options']).toBe('DENY');
    expect(response.headers['strict-transport-security']).toBeDefined();
  });

  it('should prevent XSS attacks', async () => {
    const xssPayload = '<script>alert("xss")</script>';
    
    const response = await axios.post(
      `${apiUrl}/products/search`,
      { query: xssPayload }
    );

    expect(response.data).not.toContain('<script>');
  });

  it('should prevent SQL injection', async () => {
    const sqlInjection = "'; DROP TABLE users; --";
    
    const response = await axios.get(
      `${apiUrl}/products`,
      { params: { search: sqlInjection } }
    );

    expect(response.status).toBe(200);
    // Database should still exist
  });
});
```

## 6. Visual Regression Testing

### Percy Integration

```javascript
// tests/visual/visual-regression.cy.js
describe('Visual Regression Tests', () => {
  it('should match product page snapshot', () => {
    cy.visit('http://localhost:3000/products/prod_123');
    cy.percySnapshot('product-detail-page');
  });

  it('should match checkout page snapshot', () => {
    cy.visit('http://localhost:3000/checkout');
    cy.percySnapshot('checkout-page');
  });

  it('should match order confirmation snapshot', () => {
    cy.visit('http://localhost:3000/order-confirmation/order_123');
    cy.percySnapshot('order-confirmation-page');
  });
});
```

## 7. Accessibility Testing

### axe-core Integration

```javascript
// tests/accessibility/accessibility.test.js
const { AxeBuilder } = require('@axe-core/playwright');

describe('Accessibility Tests', () => {
  it('product page should have no accessibility issues', async () => {
    const page = await browser.newPage();
    await page.goto('http://localhost:3000/products/prod_123');

    const results = await new AxeBuilder({ page })
      .withTags(['wcag2aa'])
      .analyze();

    expect(results.violations).toHaveLength(0);
  });

  it('checkout form should be keyboard navigable', async () => {
    const page = await browser.newPage();
    await page.goto('http://localhost:3000/checkout');

    // Tab through form
    await page.keyboard.press('Tab');
    await page.keyboard.press('Tab');
    
    // Focus should be on first input
    const focusedElement = await page.evaluate(() => {
      return document.activeElement.getAttribute('data-testid');
    });

    expect(focusedElement).toBeDefined();
  });
});
```

## Test Execution & Coverage

### Jest Configuration

```javascript
// jest.config.js
module.exports = {
  testEnvironment: 'node',
  collectCoverageFrom: [
    'src/**/*.js',
    '!src/**/*.test.js',
    '!src/**/index.js'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  },
  setupFilesAfterEnv: ['<rootDir>/jest.setup.js'],
  testMatch: ['**/__tests__/**/*.test.js']
};
```

### Coverage Reports

```bash
# Generate coverage report
npm run test:coverage

# Generate HTML report
npm run test:coverage -- --coverage-reporters=html

# View in browser
open coverage/index.html
```

## Continuous Integration Testing

### GitHub Actions Test Workflow

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: postgres
      redis:
        image: redis:7

    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: 'npm'

      - run: npm ci
      - run: npm run lint
      - run: npm run test:unit -- --coverage
      - run: npm run test:integration
      
      - uses: codecov/codecov-action@v3

  e2e:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm run test:e2e
      
      - uses: actions/upload-artifact@v3
        if: failure()
        with:
          name: cypress-videos
          path: cypress/videos
```

## Test Metrics & Goals

| Metric | Target | Tool |
|--------|--------|------|
| Code Coverage | > 80% | Jest |
| E2E Coverage | 100% critical flows | Cypress |
| Performance | < 200ms p95 | k6 |
| Security Score | A+ | OWASP |
| Accessibility | WCAG 2.1 AA | axe-core |
| Performance Score | > 90 | Lighthouse |

## Testing Checklist

- [ ] All unit tests pass
- [ ] Code coverage > 80%
- [ ] Integration tests pass
- [ ] E2E tests for critical flows pass
- [ ] Performance tests pass
- [ ] Security tests pass
- [ ] Accessibility tests pass
- [ ] No console errors/warnings
- [ ] No hardcoded secrets in tests
- [ ] Test data properly cleaned up
