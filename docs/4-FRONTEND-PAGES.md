# 4. Frontend Pages & UI Design

## Technology Stack

- **Framework**: React 18+ with TypeScript
- **State Management**: Redux Toolkit
- **Routing**: React Router v6
- **UI Framework**: Tailwind CSS / Material-UI
- **Forms**: React Hook Form + Zod validation
- **HTTP Client**: Axios
- **Real-time**: Socket.io for notifications
- **Charts**: Chart.js / Recharts
- **Testing**: Jest + React Testing Library

## Page Structure

```
src/
├── pages/
│   ├── auth/
│   │   ├── Login.tsx
│   │   ├── Register.tsx
│   │   ├── ForgotPassword.tsx
│   │   └── ResetPassword.tsx
│   ├── customer/
│   │   ├── Dashboard.tsx
│   │   ├── Browse.tsx
│   │   ├── ProductDetail.tsx
│   │   ├── Cart.tsx
│   │   ├── Checkout.tsx
│   │   ├── Orders.tsx
│   │   ├── OrderDetails.tsx
│   │   ├── Profile.tsx
│   │   ├── Addresses.tsx
│   │   ├── Wishlist.tsx
│   │   └── TrackDelivery.tsx
│   ├── admin/
│   │   ├── Dashboard.tsx
│   │   ├── Users.tsx
│   │   ├── Products.tsx
│   │   ├── Inventory.tsx
│   │   ├── Orders.tsx
│   │   ├── Reports.tsx
│   │   ├── Settings.tsx
│   │   └── Analytics.tsx
│   └── common/
│       ├── NotFound.tsx
│       └── Error.tsx
├── components/
├── hooks/
├── services/
├── store/
├── utils/
└── styles/
```

## Customer Portal Pages

### 1. **Authentication Pages**

#### Login Page
- Email/Password login
- "Remember me" checkbox
- "Forgot password" link
- Social login (Google, Facebook)
- Sign up link
- Error messages and validation
- Password visibility toggle

#### Register Page
- Email input with validation
- Password strength meter
- Confirm password
- First name, Last name
- Phone number
- Terms & conditions checkbox
- CAPTCHA verification
- Login link

#### Forgot Password Page
- Email input
- Submit button
- Success message with verification link
- Resend email option

#### Reset Password Page
- New password input
- Confirm password input
- Password strength requirements display
- Success redirect to login

### 2. **Dashboard Page (Customer)**

**Components**:
- Welcome header with customer name
- Quick actions (Browse, Orders, Wishlist)
- Recent orders summary
- Delivery tracking widget
- Loyalty points display
- Recommended products carousel
- Promotional banners
- Saved addresses quick access
- Account navigation sidebar

**Features**:
- Responsive grid layout
- Real-time order status updates
- Quick order reorder button
- Address selection for next order
- Loyalty points conversion option

### 3. **Product Browsing Pages**

#### Product List/Browse Page
**Layout**: Grid or List view
**Components**:
- Search bar with autocomplete
- Sidebar filters (Category, Price, Rating, Brand)
- Sorting options (Price, Rating, Newest, Popularity)
- Product cards showing:
  - Product image
  - Product name
  - Brand
  - Price and discount
  - Star rating with count
  - Quick "Add to Cart" button
  - Heart icon for wishlist
- Pagination controls
- Results count
- View toggle (Grid/List)

**Features**:
- Real-time inventory status badge
- Prescription requirement indicator
- Quick preview modal on hover
- Filter persistence
- Search history
- Recently viewed products

#### Product Detail Page
**Sections**:
- **Header**: Breadcrumb navigation
- **Main Image**: Large product image with zoom
- **Image Gallery**: Thumbnail carousel
- **Product Info**:
  - Name and brand
  - Star rating with review count
  - Price, discount, and final price
  - Availability status
  - SKU and batch info
  - Manufacturer details
- **Description**: Expandable detailed description
- **Specifications**: Table of specs
- **Reviews Section**:
  - Average rating
  - Filter by rating
  - Review cards with:
    - Reviewer name and date
    - Rating
    - Title and comment
    - Helpful/Unhelpful buttons
    - Verified purchase badge
  - Add review button (if customer purchased)
- **Recommendations**: Related products carousel
- **Sidebar**:
  - Quantity selector
  - Add to Cart button (prominent)
  - Add to Wishlist button
  - Share buttons
  - Prescription requirement warning (if applicable)

**Features**:
- Image zoom and fullscreen
- Quantity adjustable before adding to cart
- Real-time stock updates
- Similar products recommendations
- Customer Q&A section
- Share via social media
- Print product info

### 4. **Shopping Cart Page**

**Sections**:
- **Cart Summary**:
  - Product list with:
    - Image thumbnail
    - Product name
    - Price per unit
    - Quantity selector
    - Delete button
  - Quantity totals
  - Stock availability warnings
- **Order Summary** (right sidebar):
  - Subtotal
  - Tax calculation
  - Discount breakdown
  - Shipping cost
  - Total amount
  - Continue Shopping button
  - Proceed to Checkout button
- **Coupon Section**:
  - Coupon code input
  - Apply button
  - Applied coupon display with remove option

**Features**:
- Save for later functionality
- Quantity updates in real-time
- Subtotal updates automatically
- Empty cart message with recommendations
- Stock warnings for unavailable items
- Continue shopping link
- Coupon code validation

### 5. **Checkout Page**

**Multi-step Process**:
1. **Shipping Address**:
   - Select from saved addresses
   - Add new address form
   - Address type (Home/Work/Other)
   - Delivery date selection
   - Special instructions textarea

2. **Delivery Options**:
   - Estimated delivery date
   - Delivery partner info (once assigned)
   - Delivery instructions

3. **Payment Method**:
   - Payment method selection:
     - Credit/Debit Card
     - UPI
     - Net Banking
     - Digital Wallet
     - Cash on Delivery
   - Card details (if card selected)
   - Billing address same as shipping

4. **Order Review**:
   - Cart summary
   - Address confirmation
   - Delivery details
   - Payment method confirmation
   - Total amount
   - Terms and conditions checkbox
   - Place Order button

**Features**:
- Progress indicator
- Form validation with error messages
- Address autocomplete (Google Maps)
- Save address for future use
- Secure payment processing
- Order number generation
- Confirmation notification

### 6. **Order Pages**

#### Orders List Page
**Components**:
- **Filters**:
  - Order status (All, Pending, Confirmed, Processing, Shipped, Delivered)
  - Date range picker
  - Search by order number
- **Order Cards** showing:
  - Order number
  - Order date
  - Status with badge color
  - Items count
  - Total amount
  - Delivery date (if applicable)
  - "View Details" button
  - "Track Delivery" button
- **Pagination**

**Features**:
- Filter and search functionality
- Status-based grouping
- Quick reorder button
- Download invoice option
- Cancel order option (if eligible)

#### Order Details Page
**Sections**:
- **Order Header**:
  - Order number
  - Order date
  - Status timeline (Pending → Confirmed → Processing → Shipped → Delivered)
  - Payment status
- **Items List**:
  - Product image and name
  - Quantity and price
  - Item subtotal
- **Shipping Details**:
  - Delivery address
  - Estimated/Actual delivery date
  - Delivery partner info (if assigned)
- **Payment Summary**:
  - Subtotal
  - Tax
  - Discount
  - Shipping
  - Total
  - Payment method used
- **Actions**:
  - Track delivery button
  - Download invoice button
  - Return/Cancel button (if eligible)
  - Contact support button
- **Reviews Section** (if delivered):
  - Add review button
  - Existing reviews

### 7. **Delivery Tracking Page**

**Components**:
- **Status Timeline**:
  - Order Placed → Confirmed → Processing → Picked Up → In Transit → Delivered
  - Current status highlighted
  - Timestamps for each status
- **Map**:
  - Real-time delivery partner location
  - Route visualization
  - Destination marker
  - Current location marker
- **Delivery Partner Card**:
  - Photo
  - Name
  - Rating
  - Phone number (masked call/chat option)
  - Vehicle info
  - Current location text
- **Estimated Delivery**:
  - Date and time range
  - Last update timestamp
- **Actions**:
  - Contact delivery partner (Chat/Call)
  - Report issue
  - Share tracking link
  - Delivery instructions

**Features**:
- Real-time location updates
- Route optimization visualization
- SMS/Email notifications on status change
- Call/Chat with delivery partner
- Live support chat
- Proof of delivery photo

### 8. **Profile Pages**

#### Profile Page
- **Personal Information**:
  - First name, Last name
  - Email
  - Phone number
  - Profile picture upload
  - Edit button
- **Account Status**:
  - Account created date
  - Last login
  - Verification status
- **Quick Links**:
  - Manage Addresses
  - Preferences
  - Saved Wishlist
  - Download Invoice

#### Addresses Page
- **Address List**:
  - Address cards with:
    - Type badge (Home/Work/Other)
    - Full address
    - Default indicator
    - Edit button
    - Delete button
- **Add New Address** button
- **Add Address Form** (modal/page):
  - Address type
  - Street address with autocomplete
  - City, State, Postal Code
  - Country
  - Latitude/Longitude (auto-filled from map)
  - Set as default checkbox
  - Set as billing address checkbox

#### Preferences Page
- **Notification Settings**:
  - Email notifications toggle
  - SMS notifications toggle
  - Push notifications toggle
  - Marketing emails toggle
- **Display Settings**:
  - Theme selector (Light/Dark)
  - Language selector
- **Privacy Settings**:
  - Data sharing preferences
  - Activity visibility

#### Wishlist Page
- **Wishlist Items** (grid view):
  - Product image
  - Product name
  - Price
  - Add to Cart button
  - Remove from wishlist button
  - Share wishlist link option

### 9. **Search Results Page**
- Search input with history
- Filter and sort options
- Results grid with product cards
- No results message with suggestions
- Search analytics

## Admin Dashboard Pages

### 1. **Dashboard Overview**

**Widgets**:
- **KPI Cards**:
  - Total Revenue (current period)
  - Total Orders
  - Total Customers
  - Average Order Value
  - Conversion Rate
- **Charts**:
  - Revenue trend line chart
  - Orders trend
  - Category breakdown pie chart
  - Top products bar chart
- **Tables**:
  - Recent orders
  - Top customers
  - Low stock products alert
  - Pending orders
- **Quick Actions**:
  - New product button
  - View all orders button
  - Manage inventory button

**Features**:
- Date range selector
- Export reports
- Refresh data button
- Real-time metrics

### 2. **User Management Page**
- **User List Table**:
  - ID, Name, Email, Phone
  - Role badge
  - Status badge
  - Created date
  - Last login
  - Actions (View, Edit, Suspend, Delete)
- **Filters**: Role, Status, Date range
- **Search**: User name, email, phone
- **User Detail Modal**:
  - Profile info
  - Orders count
  - Total spent
  - Account history
  - Suspend/Activate button

### 3. **Product Management Page**
- **Product List Table**:
  - SKU, Name, Category
  - Price, Cost Price
  - Current Stock
  - Rating
  - Status badge
  - Actions (Edit, Delete, Manage Images)
- **Add Product** button
- **Edit Product Form**:
  - Product details
  - Category selection
  - Pricing
  - Images upload
  - Description editor
  - Stock management
  - Prescription requirement toggle

### 4. **Inventory Management Page**
- **Inventory Table**:
  - Product SKU, Name
  - Available Quantity (with low stock indicator)
  - Reserved Quantity
  - Damaged Quantity
  - Reorder Level
  - Reorder Quantity
  - Last Restock Date
  - Actions (Edit, View History)
- **Filters**: Low stock only, Category
- **Bulk Update** option
- **Stock History Modal**:
  - Transaction log
  - In/Out movements
  - Adjustments
  - Damage records

### 5. **Orders Management Page**
- **Orders Table**:
  - Order Number, Customer Name
  - Order Date
  - Status (with badge)
  - Total Amount
  - Payment Status
  - Actions (View, Download Invoice, Cancel)
- **Filters**: Status, Payment Status, Date range
- **Search**: Order number, customer name
- **Order Details Modal**:
  - Customer info
  - Items list
  - Delivery address
  - Payment details
  - Status update option
  - Cancel option
  - Generate invoice button

### 6. **Analytics & Reports Page**
- **Report Types**:
  - Sales Report
  - Customer Report
  - Product Report
  - Inventory Report
  - Delivery Report
- **Filters**:
  - Date range
  - Category/Product
  - Status
  - Payment method
- **Export Options**: PDF, Excel, CSV
- **Visualization**:
  - Charts and graphs
  - Summary tables
  - Trend analysis

### 7. **Settings Page**
- **General Settings**:
  - Shop name
  - Shop description
  - Contact email
  - Support phone
- **Business Settings**:
  - Tax settings
  - Shipping settings
  - Return policy
  - Discount rules
- **Email Templates**: Configure email notifications
- **System Configuration**:
  - Database backup
  - Cache clear
  - System logs
  - API integrations

## Responsive Design Strategy

### Breakpoints
- **Mobile**: < 640px (sm)
- **Tablet**: 640px - 1024px (md/lg)
- **Desktop**: > 1024px (xl/2xl)

### Mobile-First Approach
- Bottom navigation for main menu
- Collapsible sidebar
- Full-width forms and inputs
- Optimized images and lazy loading
- Touch-friendly buttons (min 44x44px)

### Tablet Adaptations
- Side navigation drawer
- Multi-column layouts
- Optimized spacing

### Desktop Features
- Top navigation bar
- Fixed sidebar
- Multi-column grids
- Hover effects

## Accessibility (WCAG 2.1 AA)

- Semantic HTML elements
- ARIA labels for icons and buttons
- Keyboard navigation support
- Color contrast ratios (4.5:1 for text)
- Focus indicators
- Alt text for images
- Form error handling
- Screen reader friendly

## Performance Optimizations

- Code splitting and lazy loading
- Image optimization (WebP, srcset)
- CSS-in-JS optimization
- Bundle analysis
- Caching strategies
- Service Worker for offline support
- Progressive Web App (PWA) support

## State Management (Redux)

```
store/
├── slices/
│   ├── authSlice.ts
│   ├── customerSlice.ts
│   ├── productSlice.ts
│   ├── cartSlice.ts
│   ├── orderSlice.ts
│   ├── notificationSlice.ts
│   └── uiSlice.ts
├── thunks/
└── selectors/
```

## API Integration

- Axios instance with interceptors
- Request/response transformation
- Error handling
- Token refresh logic
- Retry mechanism
- Request cancellation
- Loading states

## Testing Strategy

- Unit tests for components
- Integration tests for pages
- E2E tests for critical flows
- Visual regression testing
- Accessibility testing

## Browser Support

- Chrome (latest)
- Firefox (latest)
- Safari (latest)
- Edge (latest)
- Mobile browsers (iOS Safari, Chrome Mobile)
