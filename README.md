# Node.js Authentication Tutorial

A complete guide to building authentication with Node.js, Express, Passport.js, PostgreSQL, and bcrypt.

## 📋 Table of Contents
- [Prerequisites](#prerequisites)
- [Project Setup](#project-setup)
- [Database Setup](#database-setup)
- [Install Dependencies](#install-dependencies)
- [Project Structure](#project-structure)
- [Step-by-Step Implementation](#step-by-step-implementation)
- [Running the Application](#running-the-application)
- [Testing](#testing)
- [Key Concepts](#key-concepts)

---

## Prerequisites

- Node.js (v14 or higher)
- PostgreSQL installed and running
- Basic knowledge of JavaScript, Express, and SQL

---

## Project Setup

### 1. Initialize the Project

```bash
mkdir odin-auth
cd odin-auth
npm init -y
```

### 2. Update `package.json`

Add these fields to your `package.json`:

```json
{
  "name": "odin-auth",
  "version": "1.0.0",
  "description": "learning node js auth",
  "type": "module",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js"
  }
}
```

**Important:** The `"type": "module"` enables ES6 import/export syntax.

---

## Database Setup

### 1. Create PostgreSQL Database

```sql
-- Connect to PostgreSQL
psql -U postgres

-- Create database
CREATE DATABASE odin_auth;

-- Connect to the database
\c odin_auth

-- Create users table
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  username VARCHAR(255) UNIQUE NOT NULL,
  password VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Verify table creation
\dt
\d users
```

### 2. Create `.env` File

Create a `.env` file in the root directory:

```env
DB_HOST=localhost
DB_USER=postgres
DB_DATABASE=odin_auth
DB_PASSWORD=your_password_here
DB_PORT=5432
```

**Important:** Add `.env` to your `.gitignore` to keep credentials secure!

```bash
echo ".env" >> .gitignore
echo "node_modules/" >> .gitignore
```

---

## Install Dependencies

```bash
npm install express ejs express-session passport passport-local pg bcryptjs dotenv
npm install --save-dev nodemon
```

### What Each Package Does:

- **express**: Web framework for Node.js
- **ejs**: Templating engine for rendering HTML
- **express-session**: Session middleware for Express
- **passport**: Authentication middleware
- **passport-local**: Username/password authentication strategy
- **pg**: PostgreSQL client for Node.js
- **bcryptjs**: Password hashing library
- **dotenv**: Load environment variables from .env file
- **nodemon**: Auto-restart server on file changes (dev only)

---

## Project Structure

```
odin-auth/
├── views/
│   ├── index.ejs          # Home/login page
│   └── sign-up-form.ejs   # Registration page
├── .env                   # Environment variables (don't commit!)
├── .gitignore
├── index.js               # Main server file
├── package.json
└── README.md
```

---

## Step-by-Step Implementation

### Step 1: Create `index.js` - Basic Setup

```javascript
import { join, dirname } from "node:path";
import { fileURLToPath } from "node:url";
import express, { urlencoded } from "express";
import "dotenv/config";

// ES6 module workaround for __dirname
const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

const app = express();

// View engine setup
app.set("views", join(__dirname, "views"));
app.set("view engine", "ejs");

// Middleware to parse form data
app.use(urlencoded({ extended: false }));

// Test route
app.get("/", (req, res) => {
  res.send("Server is running!");
});

app.listen(3000, () => {
  console.log("app listening on port 3000!");
});
```

**Test:** Run `npm run dev` and visit `http://localhost:3000`

---

### Step 2: Add Database Connection

Add PostgreSQL pool configuration:

```javascript
import { Pool } from "pg";

const pool = new Pool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  database: process.env.DB_DATABASE,
  password: process.env.DB_PASSWORD,
  port: process.env.DB_PORT,
});
```

**Test database connection:**

```javascript
// Add temporary test
pool.query("SELECT NOW()", (err, res) => {
  console.log(err, res);
  pool.end();
});
```

---

### Step 3: Add Session Middleware

```javascript
import session from "express-session";

app.use(session({ 
  secret: "cats",              // Change this to a random string in production
  resave: false,               // Don't save session if unmodified
  saveUninitialized: false     // Don't create session until something stored
}));
```

**Session options explained:**
- `secret`: Used to sign the session ID cookie (use strong secret in production)
- `resave`: Forces session to be saved even if not modified
- `saveUninitialized`: Forces uninitialized session to be saved

---

### Step 4: Configure Passport.js

#### Initialize Passport

```javascript
import passport from "passport";
import { Strategy as LocalStrategy } from "passport-local";

app.use(passport.initialize());
app.use(passport.session());
```

#### Configure Local Strategy (Login Logic)

```javascript
passport.use(
  new LocalStrategy(async (username, password, done) => {
    try {
      // Find user in database
      const { rows } = await pool.query(
        "SELECT * FROM users WHERE username = $1", 
        [username]
      );
      const user = rows[0];

      // Check if user exists
      if (!user) {
        return done(null, false, { message: "Incorrect username" });
      }

      // Compare password with hashed password
      const match = await bcrypt.compare(password, user.password);
      if (!match) {
        return done(null, false, { message: "Incorrect password" });
      }

      // Success!
      return done(null, user);
    } catch(err) {
      return done(err);
    }
  })
);
```

**How LocalStrategy works:**
1. User submits login form
2. Passport calls this function with username & password
3. We query database for user
4. Compare plain password with hashed password using bcrypt
5. Return user object if successful, or false if failed

#### Serialize/Deserialize User

```javascript
// Store user ID in session
passport.serializeUser((user, done) => {
  done(null, user.id);
});

// Retrieve user from database using ID stored in session
passport.deserializeUser(async (id, done) => {
  try {
    const { rows } = await pool.query(
      "SELECT * FROM users WHERE id = $1", 
      [id]
    );
    const user = rows[0];
    done(null, user);
  } catch(err) {
    done(err);
  }
});
```

**Why serialize/deserialize?**
- Sessions only store user ID (not entire user object) to save memory
- On each request, Passport uses the ID to fetch full user data
- `req.user` becomes available in all routes after authentication

---

### Step 5: Add bcryptjs for Password Hashing

```javascript
import bcrypt from "bcryptjs";
```

**Never store plain passwords!** bcrypt:
- Hashes passwords with salt
- Makes brute-force attacks nearly impossible
- One-way encryption (can't reverse hash to get password)

---

### Step 6: Create Routes

#### Home/Login Route

```javascript
app.get("/", (req, res) => {
  res.render("index", { user: req.user });
});
```

**Note:** `req.user` is automatically populated by Passport if logged in.

#### Sign-up Routes

```javascript
// Display sign-up form
app.get("/sign-up", (req, res) => res.render("sign-up-form"));

// Handle sign-up form submission
app.post("/sign-up", async (req, res, next) => {
  try {
    // Hash password with bcrypt (10 salt rounds)
    const hashedPassword = await bcrypt.hash(req.body.password, 10);
    
    // Insert user into database
    await pool.query(
      "INSERT INTO users (username, password) VALUES ($1, $2)", 
      [req.body.username, hashedPassword]
    );
    
    res.redirect("/");
  } catch (error) {
    console.error(error);
    next(error);
  }
});
```

**bcrypt.hash(password, 10):**
- First arg: plain password
- Second arg: salt rounds (10 is good balance of security/speed)
- Higher number = more secure but slower

#### Login Route

```javascript
app.post(
  "/log-in",
  passport.authenticate("local", {
    successRedirect: "/",
    failureRedirect: "/"
  })
);
```

**Passport handles:**
1. Parsing form data
2. Calling LocalStrategy
3. Setting up session
4. Redirecting based on success/failure

#### Logout Route

```javascript
app.get("/log-out", (req, res, next) => {
  req.logout((err) => {
    if (err) {
      return next(err);
    }
    res.redirect("/");
  });
});
```

---

### Step 7: Create Views

#### `views/index.ejs` - Home/Login Page

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Log In</title>
</head>
<body>
  <% if (locals.user) {%>
    <h1>WELCOME BACK <%= user.username %></h1>
    <a href="/log-out">LOG OUT</a>
  <% } else { %>
    <h1>please log in</h1>
    <form action="/log-in" method="POST">
      <label for="username">Username</label>
      <input id="username" name="username" placeholder="username" type="text" />
      <label for="password">Password</label>
      <input id="password" name="password" type="password" />
      <button type="submit">Log In</button>
    </form>
  <%}%>
</body>
</html>
```

**Key points:**
- `locals.user` checks if user exists (logged in)
- Shows welcome message if logged in, login form if not
- Form posts to `/log-in` route

#### `views/sign-up-form.ejs` - Registration Page

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Sign Up</title>
</head>
<body>
  <h1>Sign Up</h1>
  <form action="/sign-up" method="POST">
    <label for="username">Username</label>
    <input id="username" name="username" placeholder="username" type="text" />
    <label for="password">Password</label>
    <input id="password" name="password" type="password" />
    <button type="submit">Sign Up</button>
  </form>
</body>
</html>
```

---

## Running the Application

### Development Mode (auto-restart)

```bash
npm run dev
```

### Production Mode

```bash
npm start
```

Visit: `http://localhost:3000`

---

## Testing

### 1. Sign Up a New User
- Go to `http://localhost:3000/sign-up`
- Enter username and password
- Click "Sign Up"
- Should redirect to home page

### 2. Verify User in Database

```sql
psql -U postgres -d odin_auth
SELECT * FROM users;
```

You should see:
- User with hashed password (not plain text!)
- Password starts with `$2a$10$` (bcrypt hash format)

### 3. Log In
- Go to `http://localhost:3000`
- Enter credentials
- Should see welcome message

### 4. Log Out
- Click "LOG OUT" link
- Should show login form again

---

## Key Concepts

### 🔐 Authentication Flow

1. **Sign Up:**
   - User submits form → password hashed with bcrypt → stored in database

2. **Login:**
   - User submits form → Passport LocalStrategy triggered
   - Query database for user → compare passwords with bcrypt
   - If match: serialize user ID → store in session → set cookie

3. **Authenticated Requests:**
   - Browser sends session cookie with each request
   - Passport deserializes user ID → fetches user from DB
   - `req.user` populated with user data

4. **Logout:**
   - Destroy session → clear cookie → redirect

---

### 🛡️ Security Best Practices

✅ **Implemented:**
- Passwords hashed with bcrypt
- Environment variables for DB credentials
- Session secrets
- SQL parameterized queries (prevents SQL injection)

⚠️ **Add for Production:**
- HTTPS only
- Strong session secret (use random generator)
- Session store (Redis/PostgreSQL instead of memory)
- CSRF protection
- Rate limiting
- Input validation & sanitization
- Password strength requirements

---

### 📦 Middleware Order Matters!

```javascript
// 1. Session must come before Passport
app.use(session({...}));

// 2. Initialize Passport
app.use(passport.initialize());

// 3. Connect Passport to session
app.use(passport.session());

// 4. Body parser for form data
app.use(urlencoded({ extended: false }));
```

---

### 🔑 Important Files to Remember

| File | Purpose |
|------|---------|
| `.env` | Database credentials (don't commit!) |
| `index.js` | Server setup, routes, Passport config |
| `views/index.ejs` | Home/login page |
| `views/sign-up-form.ejs` | Registration page |

---

### 🚀 Quick Start Commands (for next time)

```bash
# Start PostgreSQL
sudo service postgresql start

# Run the app
npm run dev

# Check database
psql -U postgres -d odin_auth
SELECT * FROM users;
```

---

### 🐛 Common Issues & Solutions

**Problem:** Can't connect to database  
**Solution:** Check PostgreSQL is running and `.env` credentials are correct

**Problem:** "Cannot use import statement"  
**Solution:** Add `"type": "module"` to `package.json`

**Problem:** Session not persisting  
**Solution:** Check cookie settings and make sure session middleware is before Passport

**Problem:** Password validation failing  
**Solution:** Ensure you're using `bcrypt.compare()` not `===`

---

## 📚 Documentation References

- [Passport.js](http://www.passportjs.org/)
- [Express.js](https://expressjs.com/)
- [bcryptjs](https://github.com/dcodeIO/bcrypt.js)
- [node-postgres](https://node-postgres.com/)
- [EJS](https://ejs.co/)

---

**Author:** Younes Hebaiche  
**License:** ISC
