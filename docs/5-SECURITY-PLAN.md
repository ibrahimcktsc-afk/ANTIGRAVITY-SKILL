# 5. Security Plan

## Security Overview

Security is a multi-layered approach covering infrastructure, application, data, and operational aspects.

## 1. Authentication & Authorization

### Authentication Methods

#### JWT (JSON Web Tokens)
```
Token Structure:
- Header: { alg: "HS256", typ: "JWT" }
- Payload: { userId, email, role, iat, exp, aud }
- Signature: HMAC-SHA256(header.payload, secret)

Token Expiry:
- Access Token: 1 hour
- Refresh Token: 30 days
- Password Reset Token: 24 hours
```

**Implementation**:
```javascript
// Token generation
const token = jwt.sign(
  { userId, email, role },
  process.env.JWT_SECRET,
  { expiresIn: '1h', issuer: 'medicalshop', audience: 'web' }
);

// Token validation
const decoded = jwt.verify(token, process.env.JWT_SECRET, {
  issuer: 'medicalshop',
  audience: 'web'
});
```

#### OAuth2 Integration
- **Providers**: Google, Facebook, Apple
- **Flow**: Authorization Code Flow
- **Scope**: email, profile
- **Token Handling**: Exchange auth code for tokens
- **Account Linking**: Link OAuth accounts to existing users

#### Multi-Factor Authentication (MFA)

**Options**:
- TOTP (Time-based OTP) - Google Authenticator
- Email OTP (6-digit code)
- SMS OTP (6-digit code)
- Backup codes

**Implementation**:
```javascript
// Generate TOTP secret
const secret = speakeasy.generateSecret({
  name: 'Medical Shop Management',
  issuer: 'MedicalShop',
  length: 32
});

// Verify TOTP
const verified = speakeasy.totp.verify({
  secret: secret.base32,
  encoding: 'base32',
  token: userToken,
  window: 2
});
```

### Authorization (RBAC)

**Roles**:
- **Admin**: Full system access
- **Shop Manager**: Manage products, inventory, orders
- **Delivery Partner**: View assigned orders, update delivery status
- **Customer**: Browse, order, track delivery

**Permission Matrix**:

| Resource | Admin | Shop Manager | Delivery | Customer |
|----------|-------|--------------|----------|----------|
| Users | CRUD | R | R | R (self) |
| Products | CRUD | CRUD | R | R |
| Inventory | CRUD | RU | R | R |
| Orders | CRUD | CRUD | R | R (own) |
| Deliveries | CRUD | R | RU | R (own) |
| Reports | CRUD | R | - | - |
| Settings | CRUD | R | - | - |

**Implementation**:
```javascript
// Middleware for role-based access
const authorize = (allowedRoles) => {
  return (req, res, next) => {
    if (!allowedRoles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
};

// Usage
router.get('/admin/users', authorize(['admin']), getUserList);
router.post('/products', authorize(['admin', 'shop_manager']), createProduct);
```

## 2. Data Security

### Encryption

#### At Rest
- **Database**: PostgreSQL with Transparent Data Encryption (TDE)
- **Files**: S3 bucket encryption (AES-256)
- **Backups**: Encrypted with customer-managed keys (CMK)
- **Configuration**: Encrypted environment variables

#### In Transit
- **HTTPS/TLS 1.3**: All API communications
- **Certificate**: Let's Encrypt with auto-renewal
- **HSTS**: Strict-Transport-Security header
- **Certificate Pinning**: Mobile apps

#### Sensitive Data Fields
```javascript
// Encryption for sensitive fields
const sensitiveFields = [
  'password_hash',
  'ssn',
  'card_number',
  'cvv',
  'medical_history'
];

// Encryption helper
const encryptField = (value, encryptionKey) => {
  const cipher = crypto.createCipher('aes-256-cbc', encryptionKey);
  let encrypted = cipher.update(value, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  return encrypted;
};
```

### Password Security

**Requirements**:
- Minimum 8 characters
- Uppercase, lowercase, number, special character
- No common passwords (checked against breach database)
- Password history (can't reuse last 5 passwords)

**Hashing**:
```javascript
// Using bcrypt with salt rounds
const saltRounds = 12;
const hashedPassword = await bcrypt.hash(password, saltRounds);

// Verification
const isValid = await bcrypt.compare(password, hashedPassword);
```

**Reset Process**:
- Secure token generation (random 32+ bytes)
- Email verification link (token + userId)
- Token expiry: 24 hours
- One-time use (invalidate after use)

### PII (Personally Identifiable Information)

**Protected Fields**:
- Email addresses
- Phone numbers
- Names
- Addresses
- SSN/ID numbers
- Medical history
- Payment information

**Handling**:
- Encrypted at rest
- Masked in logs
- Limited access logs
- GDPR compliance (right to be forgotten)
- Data retention policies (18 months default)

## 3. API Security

### Input Validation

```javascript
// Schema validation with Zod
const CreateOrderSchema = z.object({
  cartId: z.string().uuid(),
  deliveryAddressId: z.string().uuid(),
  paymentMethod: z.enum(['credit_card', 'debit_card', 'upi', 'cod']),
  notes: z.string().max(500).optional(),
  quantity: z.number().int().positive().max(1000)
});

// Validation middleware
const validateRequest = (schema) => async (req, res, next) => {
  try {
    req.validatedData = await schema.parseAsync(req.body);
    next();
  } catch (error) {
    res.status(422).json({ errors: error.issues });
  }
};
```

### Output Encoding

```javascript
// Prevent XSS - escape HTML in responses
const escapeHtml = (text) => {
  const map = {
    '&': '&amp;',
    '<': '&lt;',
    '>': '&gt;',
    '"': '&quot;',
    "'": '&#039;'
  };
  return text.replace(/[&<>"']/g, (m) => map[m]);
};

// Content-Type header prevents MIME sniffing
res.header('X-Content-Type-Options', 'nosniff');
```

### SQL Injection Prevention

```javascript
// Use parameterized queries
// ❌ UNSAFE
const query = `SELECT * FROM users WHERE email = '${email}'`;

// ✅ SAFE - Using parameterized queries
const query = 'SELECT * FROM users WHERE email = $1';
const result = await db.query(query, [email]);

// ✅ SAFE - Using ORM
const user = await User.findOne({ where: { email } });
```

### CSRF Protection

```javascript
// CSRF token middleware
const csrfProtection = csrf({ cookie: false });

// Generate token for forms
router.get('/checkout', csrfProtection, (req, res) => {
  res.json({ csrfToken: req.csrfToken() });
});

// Verify token on state-changing requests
router.post('/orders', csrfProtection, createOrder);
```

### Rate Limiting

```javascript
// Rate limiting middleware
const rateLimit = require('express-rate-limit');

const loginLimiter = rateLimit({
  windowMs: 5 * 60 * 1000, // 5 minutes
  max: 5, // 5 attempts
  message: 'Too many login attempts, try again later',
  skipSuccessfulRequests: true
});

const apiLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 100, // 100 requests per minute
  keyGenerator: (req) => req.user?.id || req.ip
});

router.post('/auth/login', loginLimiter, login);
router.use('/api/', apiLimiter);
```

### CORS Configuration

```javascript
const cors = require('cors');

const corsOptions = {
  origin: [
    'https://medicalshop.com',
    'https://app.medicalshop.com',
    'https://admin.medicalshop.com'
  ],
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  maxAge: 86400 // 24 hours
};

app.use(cors(corsOptions));
```

## 4. Infrastructure Security

### Network Security

**Firewall Rules**:
- Whitelist known IP ranges
- DDoS protection (CloudFlare/AWS Shield)
- WAF (Web Application Firewall) rules
- Rate limiting at CDN level

**VPC Configuration**:
```
├── Public Subnet
│   ├── NAT Gateway
│   └── Load Balancer (ALB)
├── Private Subnet (Application)
│   ├── API Servers
│   ├── Background Jobs
│   └── Worker Processes
└── Private Subnet (Database)
    ├── PostgreSQL Primary
    ├── PostgreSQL Replica
    └── Redis
```

### Secrets Management

```javascript
// Using AWS Secrets Manager or HashiCorp Vault
const secretsManager = require('aws-sdk/clients/secretsmanager');

const getSecret = async (secretName) => {
  const client = new secretsManager({ region: 'us-east-1' });
  try {
    const data = await client.getSecretValue({ SecretId: secretName }).promise();
    return JSON.parse(data.SecretString);
  } catch (error) {
    console.error('Error fetching secret:', error);
    throw error;
  }
};

// Never commit secrets to version control
// Use .env.example for documentation
```

### SSL/TLS Certificate Management

```
Renewal Process:
- Automatic renewal 30 days before expiry
- Certificate chain validation
- Pinning for mobile apps
- Fallback certificates
```

### Server Hardening

```bash
# Disable unnecessary services
systemctl disable telnet
systemctl disable ftp

# Update security patches
apt-get update && apt-get upgrade -y

# Configure firewall
ufw enable
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp  # SSH
ufw allow 443/tcp # HTTPS

# SSH hardening
# /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no
X11Forwarding no
MaxAuthTries 3
ClientAliveInterval 300
```

## 5. Application Security

### Session Management

```javascript
// Secure session configuration
const sessionConfig = {
  store: new RedisStore({ client: redisClient }),
  name: 'sessionId',
  secret: process.env.SESSION_SECRET,
  cookie: {
    secure: true, // HTTPS only
    httpOnly: true, // Prevent XSS access
    sameSite: 'Strict', // CSRF protection
    maxAge: 1000 * 60 * 60 * 24 // 24 hours
  }
};

app.use(session(sessionConfig));
```

### Dependency Security

```bash
# Check for vulnerabilities
npm audit
npm audit fix

# Automated scanning
# Use Dependabot or Snyk for continuous monitoring

# Regular updates
npm update
npm outdated
```

### Security Headers

```javascript
app.use(helmet());

// Manually configure headers
app.use((req, res, next) => {
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('X-XSS-Protection', '1; mode=block');
  res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
  res.setHeader('Content-Security-Policy', "default-src 'self'");
  res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');
  next();
});
```

### Logging & Monitoring

```javascript
// Secure logging (no sensitive data)
const winston = require('winston');

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' })
  ]
});

// Mask sensitive information
const maskSensitiveData = (data) => {
  const masked = { ...data };
  ['password', 'cardNumber', 'cvv', 'ssn'].forEach(field => {
    if (masked[field]) {
      masked[field] = '***REDACTED***';
    }
  });
  return masked;
};

logger.info('Order created', maskSensitiveData(orderData));
```

## 6. Data Protection & Compliance

### GDPR Compliance

**Requirements**:
- Right to access: Data export functionality
- Right to be forgotten: Account deletion with data purge
- Data portability: Export in standard formats
- Consent management: Clear opt-in/opt-out

**Implementation**:
```javascript
// Export user data (GDPR Article 20)
router.get('/users/me/data-export', authenticate, async (req, res) => {
  const userData = await getUserDataExport(req.user.id);
  res.json({ data: userData, format: 'json' });
});

// Delete user account (GDPR Article 17)
router.delete('/users/me', authenticate, async (req, res) => {
  await deleteUserAndAnonymizeData(req.user.id);
  res.json({ message: 'Account deleted' });
});
```

### Data Retention Policies

```
- Order data: 7 years (legal requirement)
- User accounts: Delete after 2 years of inactivity
- Payment records: 7 years for tax
- Audit logs: 5 years
- Session data: 30 days
- Temporary files: 24 hours
```

### PCI DSS Compliance (Payment Cards)

- Never store full card numbers
- Use payment gateway tokenization
- Quarterly security assessments
- Encrypted transmission
- Access control to card data
- Regular penetration testing

## 7. Incident Response Plan

### Detection & Alert

```javascript
// Anomaly detection
const detectAnomalies = (activity) => {
  const thresholds = {
    failedLogins: 5,
    failedPayments: 3,
    unusualIpAccess: true,
    bulkDownloads: 1000
  };
  
  return Object.entries(thresholds).some(([key, threshold]) => {
    return activity[key] > threshold;
  });
};

// Alert on anomaly
if (detectAnomalies(userActivity)) {
  await sendSecurityAlert(user.email);
  await logIncident({ type: 'anomaly', user, activity });
}
```

### Incident Response Procedure

1. **Detect**: Automated alerts and monitoring
2. **Assess**: Determine severity (Critical/High/Medium/Low)
3. **Contain**: Isolate affected systems
4. **Investigate**: Root cause analysis
5. **Remediate**: Fix and patch
6. **Notify**: Inform affected users (72 hours for breaches)
7. **Document**: Incident report

### Escalation Path

```
Security Alert
    ↓
Security Team (< 15 min response)
    ↓
Critical? → Engineering Team
    ↓
Data Breach? → Legal & Compliance
    ↓
External? → Regulatory Notification
```

## 8. Security Testing

### Penetration Testing

- Quarterly internal testing
- Annual external testing by certified firm
- Scope: Infrastructure, APIs, Web app, Mobile app

### Vulnerability Scanning

- OWASP Top 10 checks
- Dependency scanning (npm audit, Snyk)
- Static code analysis (SonarQube)
- Dynamic scanning (DAST)

### Security Checklist

- [ ] All endpoints authenticated
- [ ] All inputs validated
- [ ] All outputs encoded
- [ ] HTTPS everywhere
- [ ] Security headers set
- [ ] Secrets not in code
- [ ] Sensitive data encrypted
- [ ] Audit logs enabled
- [ ] Rate limiting configured
- [ ] Error messages don't leak info
- [ ] Database backups encrypted
- [ ] Logs don't contain PII

## 9. Security Training & Awareness

- Quarterly security awareness training
- Code review with security focus
- Security incident simulations
- OWASP Top 10 training
- Password manager adoption
- Phishing awareness program

## 10. Third-Party Security

### Vendor Assessment

- Security questionnaire
- SOC 2 Type II certification
- Penetration test results
- Data handling procedures
- Incident response plan

### API Security (Third-Party)

- API key rotation every 90 days
- IP whitelisting when available
- Webhook signature verification
- Rate limit monitoring
- Dependency updates

## Security Incident Contact

**Security Team**: security@medicalshop.com
**On-call**: +91-XXXX-XXXX-XXXX
**Escalation**: ciso@medicalshop.com
