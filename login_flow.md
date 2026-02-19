# Authentication in Modern Web Applications

Complete guide to how users log in and stay logged in.

---

## 🔐 What is Authentication?

**Authentication** = Proving you are who you say you are

```
User says: "I'm john@example.com"
System asks: "Prove it!"
User provides: Password
System verifies: ✅ "Yes, you are John"
```

---

## 🎯 Main Authentication Methods

### 1. **Session-Based Authentication** (Traditional)
### 2. **Token-Based Authentication** (JWT)
### 3. **OAuth 2.0** (Login with Google/GitHub)
### 4. **Single Sign-On (SSO)**
### 5. **Passwordless** (Magic links, OTP)

---

## 1️⃣ Session-Based Authentication

### How it works:

```
┌─────────┐                           ┌─────────┐
│ Browser │                           │ Server  │
└────┬────┘                           └────┬────┘
     │                                     │
     │  1. POST /login                     │
     │     email: john@ex.com              │
     │     password: secret123             │
     ├────────────────────────────────────>│
     │                                     │
     │                                     │ 2. Check DB
     │                                     │    ✅ Valid!
     │                                     │
     │                                     │ 3. Create session
     │                                     │    sessionId: abc123
     │                                     │    Store in Redis/DB
     │                                     │
     │  4. Response                        │
     │     Set-Cookie: sessionId=abc123    │
     │<────────────────────────────────────┤
     │                                     │
     │  5. GET /profile                    │
     │     Cookie: sessionId=abc123        │
     ├────────────────────────────────────>│
     │                                     │
     │                                     │ 6. Look up session
     │                                     │    abc123 → John's data
     │                                     │
     │  7. Profile data                    │
     │<────────────────────────────────────┤
     │                                     │
```

### Code Example:

```javascript
// Server (Node.js + Express)
const express = require('express');
const session = require('express-session');
const app = express();

// Setup session middleware
app.use(session({
  secret: 'my-secret-key',
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: true,      // HTTPS only
    httpOnly: true,    // Can't access via JavaScript (XSS protection)
    maxAge: 24 * 60 * 60 * 1000  // 24 hours
  }
}));

// Login endpoint
app.post('/login', async (req, res) => {
  const { email, password } = req.body;

  // 1. Check credentials
  const user = await db.findUserByEmail(email);
  if (!user || !await bcrypt.compare(password, user.passwordHash)) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // 2. Create session
  req.session.userId = user.id;
  req.session.email = user.email;

  res.json({ message: 'Logged in successfully' });
});

// Protected endpoint
app.get('/profile', (req, res) => {
  // Check if session exists
  if (!req.session.userId) {
    return res.status(401).json({ error: 'Not authenticated' });
  }

  // User is authenticated
  res.json({
    userId: req.session.userId,
    email: req.session.email
  });
});

// Logout
app.post('/logout', (req, res) => {
  req.session.destroy();
  res.json({ message: 'Logged out' });
});
```

```javascript
// Client (React)
async function login(email, password) {
  const response = await fetch('https://api.example.com/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    credentials: 'include', // Important! Sends cookies
    body: JSON.stringify({ email, password })
  });

  // Cookie is automatically stored by browser
  return response.json();
}

async function getProfile() {
  const response = await fetch('https://api.example.com/profile', {
    credentials: 'include' // Sends session cookie
  });

  return response.json();
}
```

### Pros & Cons:

```
✅ Pros:
- Simple to implement
- Server has full control
- Easy to revoke (delete session)
- Works well with traditional server-side rendering

❌ Cons:
- Doesn't scale well (need shared session store)
- CSRF vulnerability (need CSRF tokens)
- Doesn't work well with mobile apps
- Cross-origin issues
```

---

## 2️⃣ Token-Based Authentication (JWT)

### How it works:

```
┌─────────┐                           ┌─────────┐
│ Browser │                           │ Server  │
└────┬────┘                           └────┬────┘
     │                                     │
     │  1. POST /login                     │
     │     email: john@ex.com              │
     │     password: secret123             │
     ├────────────────────────────────────>│
     │                                     │
     │                                     │ 2. Check DB
     │                                     │    ✅ Valid!
     │                                     │
     │                                     │ 3. Create JWT token
     │                                     │    Sign with secret key
     │                                     │
     │  4. Response                        │
     │     { token: "eyJhbG..." }          │
     │<────────────────────────────────────┤
     │                                     │
     │  Store token in localStorage        │
     │  or memory                          │
     │                                     │
     │  5. GET /profile                    │
     │     Authorization: Bearer eyJhbG... │
     ├────────────────────────────────────>│
     │                                     │
     │                                     │ 6. Verify JWT signature
     │                                     │    Decode user info
     │                                     │
     │  7. Profile data                    │
     │<────────────────────────────────────┤
     │                                     │
```

### What's in a JWT?

```
JWT = Header.Payload.Signature

Example token:
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiIxMjMiLCJlbWFpbCI6ImpvaG5AZXhhbXBsZS5jb20iLCJpYXQiOjE2MTYyMzkwMjJ9.4Adcj_TLlFmH5D3fG7pL8xQz

Decoded:

Header:
{
  "alg": "HS256",     // Algorithm
  "typ": "JWT"        // Type
}

Payload (Claims):
{
  "userId": "123",
  "email": "john@example.com",
  "iat": 1616239022,  // Issued at
  "exp": 1616325422   // Expires at
}

Signature:
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

### Code Example:

```javascript
// Server (Node.js + Express)
const jwt = require('jsonwebtoken');
const SECRET_KEY = 'your-secret-key-keep-it-safe';

// Login endpoint
app.post('/login', async (req, res) => {
  const { email, password } = req.body;

  // 1. Check credentials
  const user = await db.findUserByEmail(email);
  if (!user || !await bcrypt.compare(password, user.passwordHash)) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // 2. Create JWT token
  const token = jwt.sign(
    {
      userId: user.id,
      email: user.email
    },
    SECRET_KEY,
    { expiresIn: '24h' }  // Token expires in 24 hours
  );

  res.json({ token });
});

// Middleware to verify JWT
function authenticateToken(req, res, next) {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1]; // "Bearer TOKEN"

  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }

  jwt.verify(token, SECRET_KEY, (err, decoded) => {
    if (err) {
      return res.status(403).json({ error: 'Invalid token' });
    }

    req.user = decoded; // { userId: '123', email: 'john@...' }
    next();
  });
}

// Protected endpoint
app.get('/profile', authenticateToken, (req, res) => {
  res.json({
    userId: req.user.userId,
    email: req.user.email
  });
});
```

```javascript
// Client (React)
async function login(email, password) {
  const response = await fetch('https://api.example.com/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password })
  });

  const data = await response.json();

  // Store token
  localStorage.setItem('token', data.token);
  // OR better: store in memory (React state)

  return data;
}

async function getProfile() {
  const token = localStorage.getItem('token');

  const response = await fetch('https://api.example.com/profile', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });

  return response.json();
}

// Axios interceptor (automatic token inclusion)
import axios from 'axios';

const api = axios.create({
  baseURL: 'https://api.example.com'
});

// Add token to every request
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Usage
await api.get('/profile'); // Token automatically included
```

### Refresh Tokens Pattern:

```javascript
// Problem: JWT expires after 24 hours, user gets logged out
// Solution: Use refresh tokens

/*
Access Token: Short-lived (15 minutes)
Refresh Token: Long-lived (7 days)
*/

// Login returns both tokens
app.post('/login', async (req, res) => {
  // ... validate credentials

  const accessToken = jwt.sign(
    { userId: user.id },
    ACCESS_SECRET,
    { expiresIn: '15m' }
  );

  const refreshToken = jwt.sign(
    { userId: user.id },
    REFRESH_SECRET,
    { expiresIn: '7d' }
  );

  // Store refresh token in database
  await db.saveRefreshToken(user.id, refreshToken);

  res.json({ accessToken, refreshToken });
});

// Refresh endpoint
app.post('/refresh', async (req, res) => {
  const { refreshToken } = req.body;

  if (!refreshToken) {
    return res.status(401).json({ error: 'No refresh token' });
  }

  // Verify refresh token
  jwt.verify(refreshToken, REFRESH_SECRET, async (err, decoded) => {
    if (err) {
      return res.status(403).json({ error: 'Invalid refresh token' });
    }

    // Check if refresh token exists in DB
    const exists = await db.checkRefreshToken(decoded.userId, refreshToken);
    if (!exists) {
      return res.status(403).json({ error: 'Refresh token revoked' });
    }

    // Create new access token
    const accessToken = jwt.sign(
      { userId: decoded.userId },
      ACCESS_SECRET,
      { expiresIn: '15m' }
    );

    res.json({ accessToken });
  });
});
```

```javascript
// Client: Automatic token refresh
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    // If access token expired
    if (error.response.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      // Get new access token
      const refreshToken = localStorage.getItem('refreshToken');
      const response = await fetch('/refresh', {
        method: 'POST',
        body: JSON.stringify({ refreshToken })
      });
      const { accessToken } = await response.json();

      // Store new token
      localStorage.setItem('token', accessToken);

      // Retry original request
      originalRequest.headers.Authorization = `Bearer ${accessToken}`;
      return api(originalRequest);
    }

    return Promise.reject(error);
  }
);
```

### Pros & Cons:

```
✅ Pros:
- Stateless (no server-side session storage)
- Scales well (no shared state needed)
- Works great with mobile apps
- Cross-origin friendly
- Can contain user info (no DB lookup needed)

❌ Cons:
- Can't revoke tokens easily (until they expire)
- Token can get stolen (XSS vulnerability)
- Larger payload than session ID
- Need refresh token strategy for long sessions
```

---

## 3️⃣ OAuth 2.0 (Login with Google/GitHub)

### How it works:

```
┌─────────┐         ┌──────────────┐         ┌────────────┐
│ Browser │         │  Your App    │         │   Google   │
└────┬────┘         └──────┬───────┘         └─────┬──────┘
     │                     │                       │
     │ 1. Click "Login    │                       │
     │    with Google"    │                       │
     ├───────────────────>│                       │
     │                     │                       │
     │                     │ 2. Redirect to Google │
     │                     │    with client_id     │
     │<────────────────────┤                       │
     │                                             │
     │ 3. Redirect to Google                       │
     │    /authorize?client_id=...                 │
     ├────────────────────────────────────────────>│
     │                                             │
     │                                             │ 4. User logs in
     │                                             │    and approves
     │                                             │
     │ 5. Redirect back with code                  │
     │    yourapp.com/callback?code=AUTH_CODE      │
     │<────────────────────────────────────────────┤
     │                                             │
     │ 6. Send code to backend                     │
     ├───────────────────>│                       │
     │                     │                       │
     │                     │ 7. Exchange code for  │
     │                     │    access token       │
     │                     ├──────────────────────>│
     │                     │                       │
     │                     │ 8. Access token       │
     │                     │<──────────────────────┤
     │                     │                       │
     │                     │ 9. Get user info      │
     │                     ├──────────────────────>│
     │                     │                       │
     │                     │ 10. User data         │
     │                     │<──────────────────────┤
     │                     │                       │
     │                     │ 11. Create session/JWT│
     │                     │                       │
     │ 12. Login success   │                       │
     │<────────────────────┤                       │
     │                                             │
```

### Code Example:

```javascript
// Server (Node.js + Express)
const passport = require('passport');
const GoogleStrategy = require('passport-google-oauth20').Strategy;

// Configure Google OAuth
passport.use(new GoogleStrategy({
    clientID: process.env.GOOGLE_CLIENT_ID,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    callbackURL: "https://yourapp.com/auth/google/callback"
  },
  async (accessToken, refreshToken, profile, done) => {
    // User authenticated with Google
    // Now create or find user in your database

    let user = await db.findUserByGoogleId(profile.id);

    if (!user) {
      // Create new user
      user = await db.createUser({
        googleId: profile.id,
        email: profile.emails[0].value,
        name: profile.displayName,
        avatar: profile.photos[0].value
      });
    }

    return done(null, user);
  }
));

// Routes
app.get('/auth/google',
  passport.authenticate('google', { scope: ['profile', 'email'] })
);

app.get('/auth/google/callback',
  passport.authenticate('google', { failureRedirect: '/login' }),
  (req, res) => {
    // Successful authentication
    // Create JWT or session
    const token = jwt.sign({ userId: req.user.id }, SECRET_KEY);
    res.redirect(`https://yourapp.com/dashboard?token=${token}`);
  }
);
```

```javascript
// Client (React)
function LoginPage() {
  const handleGoogleLogin = () => {
    // Redirect to backend OAuth endpoint
    window.location.href = 'https://api.yourapp.com/auth/google';
  };

  return (
    <button onClick={handleGoogleLogin}>
      Login with Google
    </button>
  );
}

// Callback page (receives token)
function CallbackPage() {
  useEffect(() => {
    const params = new URLSearchParams(window.location.search);
    const token = params.get('token');

    if (token) {
      localStorage.setItem('token', token);
      navigate('/dashboard');
    }
  }, []);

  return <div>Loading...</div>;
}
```

### Pros & Cons:

```
✅ Pros:
- No password to manage
- User trusts Google/GitHub more
- Get user info easily
- Multi-factor auth handled by provider

❌ Cons:
- Depends on third-party service
- Privacy concerns (Google tracks users)
- Need to handle multiple providers
- More complex setup
```

---

## 4️⃣ Single Sign-On (SSO)

Used in enterprises (login once, access all apps).

```
Employee logs in once →
  Can access: Email, Slack, HR Portal, CRM
  Without logging in again!

Common protocols: SAML, OpenID Connect
```

---

## 5️⃣ Passwordless Authentication

### A. Magic Link (Email)

```
1. User enters email
2. Server sends email with unique link
3. User clicks link
4. Server validates link and logs user in
```

```javascript
// Server
app.post('/auth/magic-link', async (req, res) => {
  const { email } = req.body;

  // Generate unique token
  const token = crypto.randomBytes(32).toString('hex');

  // Store token (expires in 15 minutes)
  await redis.set(`magic:${token}`, email, 'EX', 900);

  // Send email
  await sendEmail(email, {
    subject: 'Login to App',
    body: `Click here to login: https://app.com/auth/verify?token=${token}`
  });

  res.json({ message: 'Check your email' });
});

app.get('/auth/verify', async (req, res) => {
  const { token } = req.query;

  // Verify token
  const email = await redis.get(`magic:${token}`);
  if (!email) {
    return res.status(400).json({ error: 'Invalid or expired link' });
  }

  // Delete token (one-time use)
  await redis.del(`magic:${token}`);

  // Create session/JWT
  const user = await db.findUserByEmail(email);
  const jwt = createJWT(user);

  res.redirect(`/dashboard?token=${jwt}`);
});
```

### B. OTP (One-Time Password)

```
1. User enters phone number
2. Server sends SMS with 6-digit code
3. User enters code
4. Server validates code and logs user in
```

---

## 🔒 Advanced Security Methods

### 1. Multi-Factor Authentication (MFA/2FA)

**Most important security enhancement!**

```
Even if password is stolen, hacker can't login without 2nd factor!

Types:
1. TOTP (Time-based One-Time Password)
   - Google Authenticator, Authy
   - Changes every 30 seconds

2. SMS OTP
   - Send code to phone

3. Hardware Keys
   - YubiKey, security keys

4. Biometric
   - Fingerprint, Face ID

5. Push Notifications
   - "Someone trying to login, approve?"
```

**Flow:**
```
1. User enters email + password ✅
2. Server: "Enter code from authenticator app"
3. User enters 6-digit code from phone
4. Server validates code ✅
5. Login successful!

Even if hacker has password, they can't login without the code!
```

**Code Example:**
```javascript
const speakeasy = require('speakeasy');
const QRCode = require('qrcode');

// Setup MFA for user
app.post('/setup-mfa', authenticateToken, async (req, res) => {
  // Generate secret
  const secret = speakeasy.generateSecret({
    name: `YourApp (${req.user.email})`
  });

  // Save secret to user's account
  await db.updateUser(req.user.id, {
    mfaSecret: secret.base32
  });

  // Generate QR code for user to scan
  const qrCode = await QRCode.toDataURL(secret.otpauth_url);

  res.json({ qrCode, secret: secret.base32 });
});

// Login with MFA
app.post('/login', async (req, res) => {
  const { email, password, mfaCode } = req.body;

  // 1. Verify password
  const user = await db.findUserByEmail(email);
  if (!user || !await bcrypt.compare(password, user.passwordHash)) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // 2. If user has MFA enabled, verify code
  if (user.mfaSecret) {
    if (!mfaCode) {
      return res.status(403).json({
        error: 'MFA code required',
        requiresMFA: true
      });
    }

    const verified = speakeasy.totp.verify({
      secret: user.mfaSecret,
      encoding: 'base32',
      token: mfaCode,
      window: 2 // Allow 2 steps before/after (60 second window)
    });

    if (!verified) {
      return res.status(401).json({ error: 'Invalid MFA code' });
    }
  }

  // 3. Create token
  const token = jwt.sign({ userId: user.id }, SECRET_KEY);
  res.json({ token });
});
```

---

### 2. OAuth 2.0 with PKCE (Proof Key for Code Exchange)

More secure version of OAuth for SPAs and mobile apps.

```
Problem with regular OAuth:
- Authorization code can be intercepted
- Attacker can exchange it for access token

PKCE solution:
- Generate random "code verifier"
- Send hash of it during auth
- Send original during token exchange
- Attacker can't forge this without the verifier
```

**Flow:**
```javascript
// Client generates random string
const codeVerifier = base64url(randomBytes(32));
const codeChallenge = base64url(sha256(codeVerifier));

// 1. Redirect to OAuth provider with challenge
window.location = `https://oauth.com/authorize?
  client_id=YOUR_CLIENT_ID&
  redirect_uri=https://yourapp.com/callback&
  code_challenge=${codeChallenge}&
  code_challenge_method=S256`;

// 2. User approves, redirected back with code
// 3. Exchange code for token with verifier
const response = await fetch('https://oauth.com/token', {
  method: 'POST',
  body: JSON.stringify({
    code: authorizationCode,
    code_verifier: codeVerifier, // Must match!
    redirect_uri: 'https://yourapp.com/callback'
  })
});
```

---

### 3. Certificate-Based Authentication (mTLS)

Used in enterprise/banking applications.

```
Instead of password, use digital certificate

How it works:
1. Organization issues certificate to user
2. User installs certificate on their device
3. During login, certificate is verified
4. No password needed!

Benefits:
- Can't be phished
- Can't be brute-forced
- Device-specific (can't share)

Used by:
- Banks (for high-value transactions)
- Government systems
- Corporate VPNs
```

---

### 4. Biometric Authentication

```javascript
// WebAuthn API (Browser standard)
// Supports: Face ID, Touch ID, Windows Hello

// Register biometric
async function registerBiometric() {
  const publicKey = {
    challenge: new Uint8Array(32), // Server-generated
    rp: { name: "Your App" },
    user: {
      id: Uint8Array.from(userId, c => c.charCodeAt(0)),
      name: "john@example.com",
      displayName: "John Doe"
    },
    pubKeyCredParams: [{ alg: -7, type: "public-key" }],
    authenticatorSelection: {
      authenticatorAttachment: "platform", // Device-specific
      userVerification: "required" // Require biometric
    }
  };

  const credential = await navigator.credentials.create({ publicKey });

  // Send credential to server to save
  await fetch('/register-biometric', {
    method: 'POST',
    body: JSON.stringify({
      id: credential.id,
      publicKey: credential.response.publicKey
    })
  });
}

// Login with biometric
async function loginWithBiometric() {
  const publicKey = {
    challenge: new Uint8Array(32),
    allowCredentials: [{
      id: savedCredentialId,
      type: 'public-key'
    }]
  };

  const assertion = await navigator.credentials.get({ publicKey });

  // Send to server for verification
  const response = await fetch('/login-biometric', {
    method: 'POST',
    body: JSON.stringify({
      credentialId: assertion.id,
      signature: assertion.response.signature
    })
  });

  const { token } = await response.json();
  return token;
}
```

---

### 5. Risk-Based Authentication (Adaptive Auth)

Analyzes login context and adjusts security accordingly.

```javascript
// Server-side risk assessment
app.post('/login', async (req, res) => {
  const { email, password } = req.body;

  // 1. Verify credentials
  const user = await db.findUserByEmail(email);
  if (!user || !await bcrypt.compare(password, user.passwordHash)) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // 2. Calculate risk score
  const riskFactors = {
    // New device?
    newDevice: !await isKnownDevice(user.id, req.fingerprint),

    // New location?
    newLocation: !await isKnownLocation(user.id, req.ip),

    // Unusual time?
    unusualTime: isUnusualLoginTime(user.loginHistory, new Date()),

    // Multiple failed attempts?
    recentFailures: await getRecentFailedAttempts(email) > 3,

    // IP on blocklist?
    suspiciousIP: await isIPBlocked(req.ip),

    // Impossible travel?
    impossibleTravel: await detectImpossibleTravel(user.id, req.ip)
  };

  const riskScore = calculateRiskScore(riskFactors);

  // 3. Apply appropriate security level
  if (riskScore < 30) {
    // Low risk: Allow login
    const token = jwt.sign({ userId: user.id }, SECRET_KEY);
    return res.json({ token });

  } else if (riskScore < 70) {
    // Medium risk: Require MFA
    return res.status(403).json({
      requiresMFA: true,
      message: 'Additional verification required'
    });

  } else {
    // High risk: Block + notify user
    await sendSecurityAlert(user.email, {
      message: 'Suspicious login attempt blocked',
      location: req.location,
      ip: req.ip,
      time: new Date()
    });

    return res.status(403).json({
      error: 'Login blocked for security reasons. Check your email.'
    });
  }
});

function calculateRiskScore(factors) {
  let score = 0;
  if (factors.newDevice) score += 20;
  if (factors.newLocation) score += 20;
  if (factors.unusualTime) score += 10;
  if (factors.recentFailures) score += 30;
  if (factors.suspiciousIP) score += 40;
  if (factors.impossibleTravel) score += 50;
  return score;
}
```

**Real-world example:**
```
Scenario 1: Normal login
- Same device you always use
- Same location (home/office)
- Normal time (9am-5pm)
→ Risk score: 0
→ Action: Allow immediately

Scenario 2: Suspicious login
- New device
- Different country
- 3am login
- 5 failed attempts in last hour
→ Risk score: 80
→ Action: Block + Email alert

Scenario 3: Medium risk
- New device (bought new phone)
- Same location
→ Risk score: 40
→ Action: Require MFA
```

---

### 6. Device Fingerprinting

Track devices to detect unauthorized access.

```javascript
// Client: Generate device fingerprint
import FingerprintJS from '@fingerprintjs/fingerprintjs';

const fp = await FingerprintJS.load();
const result = await fp.get();
const deviceId = result.visitorId;

// Send with login request
await fetch('/login', {
  method: 'POST',
  body: JSON.stringify({
    email,
    password,
    deviceId,
    deviceInfo: {
      userAgent: navigator.userAgent,
      screenResolution: `${screen.width}x${screen.height}`,
      timezone: Intl.DateTimeFormat().resolvedOptions().timeZone,
      language: navigator.language
    }
  })
});
```

```javascript
// Server: Track devices
app.post('/login', async (req, res) => {
  const { email, password, deviceId, deviceInfo } = req.body;

  // Verify credentials
  const user = await db.findUserByEmail(email);
  // ... password check

  // Check if device is recognized
  const knownDevice = await db.findDevice(user.id, deviceId);

  if (!knownDevice) {
    // New device detected!

    // Send email notification
    await sendEmail(user.email, {
      subject: 'New device login',
      body: `Someone logged in from a new device:
        Device: ${deviceInfo.userAgent}
        Location: ${req.location}
        Time: ${new Date()}

        If this wasn't you, secure your account immediately.`
    });

    // Require additional verification
    return res.status(403).json({
      requiresVerification: true,
      message: 'New device detected. Check your email.'
    });
  }

  // Known device, allow login
  await db.updateDeviceLastUsed(deviceId);
  const token = jwt.sign({ userId: user.id }, SECRET_KEY);
  res.json({ token });
});
```

---

### 7. IP Allowlisting/Blocklisting

Restrict access based on IP address.

```javascript
// Corporate use case: Only allow office IPs
const ALLOWED_IPS = [
  '203.0.113.0/24',  // Office network
  '198.51.100.0/24'  // VPN
];

app.post('/login', async (req, res) => {
  const clientIP = req.ip;

  // Check if IP is allowed
  if (!isIPAllowed(clientIP, ALLOWED_IPS)) {
    return res.status(403).json({
      error: 'Access denied from this location'
    });
  }

  // Continue with login...
});

// Or: Block known malicious IPs
const BLOCKED_IPS = await fetch('https://blocklist-service.com/ips')
  .then(r => r.json());

if (BLOCKED_IPS.includes(req.ip)) {
  return res.status(403).json({ error: 'Access denied' });
}
```

---

### 8. Time-Based Access Control

Restrict login during certain hours.

```javascript
app.post('/login', async (req, res) => {
  const user = await db.findUserByEmail(req.body.email);

  // Check user's allowed time window
  const currentHour = new Date().getHours();
  const allowedStart = user.accessHours?.start || 0;
  const allowedEnd = user.accessHours?.end || 24;

  if (currentHour < allowedStart || currentHour > allowedEnd) {
    return res.status(403).json({
      error: `Access only allowed between ${allowedStart}:00 - ${allowedEnd}:00`
    });
  }

  // Continue with login...
});
```

---

### 9. Behavioral Biometrics

Analyze typing patterns, mouse movements.

```javascript
// Client: Track typing patterns
let typingPattern = [];

input.addEventListener('keydown', (e) => {
  typingPattern.push({
    key: e.key,
    timestamp: Date.now(),
    duration: 0
  });
});

input.addEventListener('keyup', (e) => {
  const lastKey = typingPattern[typingPattern.length - 1];
  lastKey.duration = Date.now() - lastKey.timestamp;
});

// Send pattern with login
await fetch('/login', {
  method: 'POST',
  body: JSON.stringify({
    email,
    password,
    typingPattern // Server analyzes if it matches user's pattern
  })
});
```

---

### 10. Hardware Security Keys

Physical device required to login.

```javascript
// YubiKey, Titan Key, etc.

// User must physically plug in USB key and press button

// WebAuthn implementation
const publicKey = {
  challenge: serverChallenge,
  allowCredentials: [{
    id: userKeyId,
    type: 'public-key',
    transports: ['usb', 'nfc']
  }]
};

const assertion = await navigator.credentials.get({ publicKey });

// User must physically touch the key
// Protects against:
// - Phishing (key verifies domain)
// - Password theft (no password needed)
// - Remote attacks (must have physical key)
```

---

## 🔒 Security Best Practices

### 1. Password Storage

```javascript
// ❌ NEVER store plain passwords
password: "secret123"

// ❌ NEVER use simple hashing
password: md5("secret123")

// ✅ Use bcrypt with salt
const bcrypt = require('bcrypt');

// Hash password
const hash = await bcrypt.hash('secret123', 10);
// Result: $2b$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy

// Verify password
const isValid = await bcrypt.compare('secret123', hash);
```

### 2. Token Storage (Client)

```javascript
// ❌ DON'T use localStorage (XSS vulnerability)
localStorage.setItem('token', token);

// ⚠️  Better: sessionStorage (cleared on tab close)
sessionStorage.setItem('token', token);

// ✅ BEST: Memory only (React state)
const [token, setToken] = useState(null);

// ✅ OR: HttpOnly cookie (can't access via JS)
// Server sets:
res.cookie('token', jwt, {
  httpOnly: true,   // Can't access via JavaScript
  secure: true,     // HTTPS only
  sameSite: 'strict' // CSRF protection
});
```

### 3. HTTPS Only

```
❌ http://example.com  → Passwords sent in plain text!
✅ https://example.com → Encrypted connection
```

### 4. Rate Limiting

```javascript
// Prevent brute-force attacks
const rateLimit = require('express-rate-limit');

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // 5 attempts
  message: 'Too many login attempts, try again later'
});

app.post('/login', loginLimiter, async (req, res) => {
  // ... login logic
});
```

---

## 🔒 Complete Security Checklist

```
✅ Password Security:
   - bcrypt hashing (10+ rounds)
   - Min 8 chars, complexity requirements
   - Check against breach databases (Have I Been Pwned)

✅ Token Security:
   - Short-lived access tokens (15 min)
   - Refresh token rotation
   - Store in httpOnly cookies or memory
   - Sign with strong secret (256+ bits)

✅ Transport Security:
   - HTTPS only (TLS 1.3)
   - HSTS headers
   - Certificate pinning for mobile

✅ Session Security:
   - Secure, httpOnly, sameSite cookies
   - CSRF tokens
   - Session timeout on inactivity

✅ Rate Limiting:
   - Login attempts (5 per 15 min)
   - API requests (100 per min)
   - Account lockout after 10 failures

✅ Multi-Factor Auth:
   - TOTP (Google Authenticator)
   - SMS backup
   - Recovery codes

✅ Monitoring:
   - Failed login attempts
   - Unusual access patterns
   - New device/location alerts
   - Account takeover detection

✅ Additional:
   - Email verification
   - Password reset security
   - Security questions (deprecated, use MFA)
   - Account recovery process
```

---

## 📊 Comparison Table

| Method | Best For | Complexity | Scalability | Mobile | Security |
|--------|----------|------------|-------------|--------|----------|
| Session | Simple apps, SSR | Low | Medium | ❌ | Good |
| JWT | APIs, mobile apps | Medium | High | ✅ | Good* |
| OAuth | Social login | High | High | ✅ | Very Good |
| SSO | Enterprise | Very High | High | ✅ | Very Good |
| Passwordless | Modern UX | Medium | High | ✅ | Good |
| MFA | Any critical app | Medium | High | ✅ | Excellent |
| mTLS | Banking, Enterprise | Very High | High | ⚠️ | Excellent |
| Biometric | Mobile, Modern | High | High | ✅ | Very Good |

*JWT security depends on proper implementation (refresh tokens, secure storage)

---

## 🎯 Interview Answer Template

### When Asked: "How does authentication work?"

**Good Answer:**
```
"Authentication verifies user identity through multiple approaches:

1. Foundation: Session or JWT
   - Session: Server stores user state, client gets cookie
   - JWT: Stateless token with user info, signed by server

2. Security Layers:
   - MFA: Even with stolen password, need 2nd factor
   - Risk-based: Analyze device, location, behavior
   - Device tracking: Alert on new device logins
   - Rate limiting: Prevent brute-force attacks

3. For production systems, I'd use:
   - JWT with refresh tokens (scalability)
   - TOTP-based MFA (security)
   - Risk-based checks (UX + security balance)
   - Secure storage (httpOnly cookies or memory)
   - HTTPS only, bcrypt passwords, rate limiting

The key is defense in depth - multiple layers so if one fails,
others still protect the system."
```

---

## 💡 Interview Tips

When dealing with challenging interviewers:

1. **Stay calm and confident**
   - They might be stress-testing you

2. **Ask clarifying questions**
   - "What specific attack vector are you concerned about?"
   - "Are you asking about authentication or authorization?"
   - "What security level - consumer app or banking?"

3. **Show breadth of knowledge**
   - "There are multiple approaches depending on requirements..."
   - Then list 3-5 methods

4. **Acknowledge trade-offs**
   - "MFA is more secure but affects UX"
   - "Hardware keys are safest but expensive to deploy"

5. **If they interrupt, politely persist**
   - "I understand your concern about X. Additionally, we could..."

6. **Red flag check**
   - If interviewer is consistently rude/dismissive
   - Consider if you want to work there
   - Culture starts from the top

---

## 🎯 Recommendation for Different Apps

### Consumer App (Social Media, E-commerce):
```
Use: JWT with Refresh Tokens + Optional OAuth

Why?
- Stateless (scales well)
- Works with mobile apps
- Optional social login (better UX)

Add:
- Email verification
- Password reset flow
- Basic rate limiting
```

### Financial App (Banking, Payments):
```
Use: Session + MFA + Device Tracking + Risk-Based Auth

Why?
- Highest security needs
- Can justify friction for security
- Regulatory requirements

Add:
- Transaction verification
- Suspicious activity alerts
- Account freeze capabilities
- Hardware keys for high-value accounts
```

### Enterprise App (Internal Tools):
```
Use: SSO + MFA + IP Allowlisting

Why?
- Single login for all tools
- Corporate security policies
- Audit trail requirements

Add:
- Role-based access control (RBAC)
- Activity logging
- Compliance reporting
```

### Jira Filter System (Interview Context):
```
Use: JWT with Refresh Tokens

Why?
- Stateless (scales well for 1000+ users)
- Works with React SPA
- Easy to implement
- Industry standard

Implementation:
1. Login → Get access token (15 min) + refresh token (7 days)
2. Store tokens securely (memory or httpOnly cookie)
3. Include token in all API requests
4. Auto-refresh when access token expires
5. Logout → Delete tokens
```

---

Good luck with your interviews! Remember: **Anyone who dismisses session + JWT as "easy to hack" without acknowledging proper implementation is showing their own lack of depth.** Those are literally the two most common authentication methods in production systems worldwide! 🚀
