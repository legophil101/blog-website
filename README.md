# Philip’s Blog 🧛‍♂️💻

A personal blog website built **from scratch** using **Flask**, **Python**, **Bootstrap**, and **PostgreSQL (Supabase – Singapore)** — deployed on **Render**.

![Blog Homepage Screenshot](static/assets/img/preview.png)

---

# 🏗️ Architecture Overview

```
User → Render (Flask + Gunicorn) → Supabase PostgreSQL (Singapore)
```

The application server and database are aligned in the **same cloud region** to minimize latency and prevent connection timeouts.

---

# ✨ Features

* 📝 **Full CRUD** for blog posts (create, read, update, delete)
* 🔑 **User authentication** (register, login, logout)
* 🖋️ **Rich text editing** with Flask-CKEditor
* 💬 **Comment system** linked to blog posts
* 👤 **Gravatar integration** for user profile images
* 📱 **Responsive design** using Bootstrap 5
* 🔒 **Admin-only routes** protected via custom decorators
* 📧 **Resilient contact form** using **Formspree API** to bypass cloud-provider SMTP port restrictions

  * Smooth scroll to the header after successful submission
  * Dynamic success message display
  * Reliable alternative to SMTP on restricted cloud platforms
* 🛠️ **Tech Highlight:** AJAX form submission with `fetch()` for smoother UX

---

# 🛠️ Technologies Used

| Category           | Tools                                              |
| ------------------ | -------------------------------------------------- |
| **Backend**        | Python, Flask, Gunicorn                            |
| **Database**       | PostgreSQL (Supabase – Singapore), SQLAlchemy ORM  |
| **Frontend**       | Bootstrap 5, Jinja2, Flask-CKEditor                |
| **Authentication** | Flask-Login, Werkzeug                              |
| **Deployment**     | Render, GitHub                                     |
| **Security**       | python-dotenv, environment variables, Supabase RLS |

---

# 🚀 Deployment & Live Demo

The application is deployed on **Render** using **Gunicorn** as the production WSGI server.

🔗 **Live Site**
[https://philips-blog-9kuj.onrender.com/](https://philips-blog-9kuj.onrender.com/)

> ⚠️ **Note:** Render free-tier services may take **30–50 seconds** to spin up if idle.

---

# 🗄️ Database Migration & “Resurrection” Story

This project went through **four database stages**:

1. **SQLite (Local Development)**
   Used during initial development.

2. **Render PostgreSQL (Initial Deployment)**
   First cloud database during early deployment.

3. **Supabase PostgreSQL — Mumbai Region**
   Migrated for improved persistence but experienced cross-region latency.

4. **Supabase PostgreSQL — Singapore Region (Final Production Setup)**
   Migrated again to align the database with Render’s Singapore servers.

Aligning both services in the **same region dramatically reduced connection latency and timeout errors.**

---

## Migration Highlights

* Seamless migration by updating only the `DATABASE_URL`
* No application logic changes thanks to SQLAlchemy ORM
* PostgreSQL-compatible schema and data structure
* Supabase **Row Level Security (RLS)** enabled
* Blog successfully restored after months of downtime — inspiring the **“resurrection” theme**

---

# 📧 SMTP Email & 502 Error Fix

After deployment, the contact form caused intermittent **HTTP 502 errors** on Render.

## Root Cause

* Gmail SMTP handshakes occasionally blocked by cloud networking delays
* Environment variables were loaded during startup instead of request time

## Fix Implemented

* Environment variables are now loaded **inside the email function**
* Added:

```
timeout=15
```

to the SMTP connection

* Errors are logged safely without crashing the request

---

# 🏗️ Production Deployment Notes

After deploying on Render, several **production-only issues** appeared that did not occur locally.

---

# The Issue: Gunicorn Timeout Loop

### Symptoms

Render logs showed repeated:

```
CRITICAL WORKER TIMEOUT
SIGKILL
```

The server would restart before the homepage could load.

### Cause

On Render free-tier instances:

* Cold start + database handshake can take **35–40 seconds**
* Default Gunicorn timeout is **30 seconds**

Gunicorn assumed the app was frozen and terminated the worker.

---

# 🛠️ Fixes Implemented

## Gunicorn Worker Optimization

**Command used:**

```
gunicorn -w 1 --threads 4 --timeout 120 main:app
```

### Reasoning

Render free-tier instances have **512MB RAM**.

Using:

* **1 worker**
* **4 threads**

prevents memory exhaustion while still allowing concurrent requests.

This avoids:

* `SIGKILL`
* `Error 520`
* worker crashes

---

## Regional Latency Optimization

### Problem

The application server runs in **Render (Singapore)** while the original Supabase database was located in **Mumbai**.

This cross-region communication caused:

* long request times
* `OperationalError` connection timeouts
* occasional stalled page loads

### Solution

Migrated the entire database to a new **Supabase project in Singapore** and transferred the data **1:1**.

### Result

* Reduced database round-trip latency
* Eliminated most connection timeout errors
* Improved stability during cold starts

---

## Database Query Optimization (N+1 Fix)

The homepage query was optimized using SQLAlchemy **eager loading**:

```
joinedload()
```

This prevents the **N+1 query problem**, allowing posts and their relationships to be fetched in **one database request instead of many**.

---

## Payload Reduction

The homepage now loads only the latest posts:

```
.limit(5)
```

This reduces payload size and improves **Time To First Byte (TTFB)**.

---

## Advanced Database Connection Pooling

Configuration used:

```python
app.config['SQLALCHEMY_ENGINE_OPTIONS'] = {
    "pool_pre_ping": True,
    "pool_recycle": 300,
    "pool_size": 10,
    "max_overflow": 20,
}
```

### Result

* Prevents stale connections
* Automatically reconnects dropped database sessions
* Eliminated `PendingRollbackError` and many `OperationalError` issues

---

## Optimized Startup Logic

The following code was removed during deployment:

```
db.create_all()
```

Tables already exist in Supabase, so running it during every deploy caused unnecessary startup delays.

---

## Verified Database URL Protocol

Ensured the connection string uses:

```
postgresql://
```

instead of the deprecated:

```
postgres://
```

Required for **SQLAlchemy 2.0+ compatibility**.

---

## Heartbeat Route (`/ping`)

A lightweight route was added:

```
/ping
```

Purpose:

* Returns a `200 OK` without querying the database
* Used by **UptimeRobot** to keep the service warm
* Prevents Render’s 15-minute free-tier sleep

---

## Cascade Delete Logic

Updated model relationship:

```python
comments = relationship(
    "Comment",
    back_populates="parent_post",
    cascade="all, delete"
)
```

### Result

Deleting a blog post automatically removes its comments, preventing:

```
IntegrityError
```

and maintaining database integrity.

---

# Current Status

✅ App boots reliably on Render free-tier

✅ Database latency significantly reduced

✅ Posts and comments delete safely

✅ CKEditor warnings suppressed and form binding fixed

✅ Custom **500 error page** added for production errors

---

# 📦 Local Installation

### 1️⃣ Clone repository

```
git clone https://github.com/legophil101/blog-website.git
cd blog-website
```

---

### 2️⃣ Create `.env`

```
SECRET_KEY=your_secret_key
DATABASE_URL=your_supabase_postgres_url
EMAIL_KEY=your_email_address
PASSWORD_KEY=your_gmail_app_password
```

---

### 3️⃣ Install dependencies

```
pip install -r requirements.txt
```

---

### 4️⃣ Run application

```
python main.py
```

---

### 5️⃣ Open browser

```
http://127.0.0.1:5000
```

---

# 📚 Lessons Learned

* Resolving Python dependency conflicts (Flask, Jinja2, MarkupSafe)
* Deploying Flask apps using **Gunicorn + Render**
* Migrating from SQLite to cloud PostgreSQL
* Using `.env` files for environment security
* Debugging deployment-only issues using **Render logs**
* Handling SMTP failures safely in production
* Implementing **database connection pooling**
* Understanding **cloud region latency**
* Aligning application servers and databases for better performance
* Keeping free-tier services alive using **UptimeRobot**
* Reading logs at **4–5 AM ☕**

---

# 🧠 My Journey

This project started as part of my programming course and eventually became my **first fully deployed production web application**.

Challenges along the way included:

* Resolving Python version compatibility issues on Render
* Migrating databases across multiple platforms
* Debugging production failures that did not appear locally
* Learning how infrastructure decisions affect performance

It took **months, persistence, and many late nights**, but seeing the site finally go live made the entire process worth it.

**Better Call Phil for your next Flask site. 😁**
