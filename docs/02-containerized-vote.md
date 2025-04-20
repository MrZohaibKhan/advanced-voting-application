### 📄 `02-containerized-vote.md`

```markdown
# 🐳 Containerizing the Vote Service — From Basic to Battle-Ready

When I began containerizing the `vote` service, I took the typical route first — something that works, nothing fancy:

```dockerfile
FROM python:alpine

COPY . /app

WORKDIR /app

RUN pip install -r requirements.txt

CMD ["python", "app.py"]
```

It ran. Job done? Not quite.

---

## 🧠 Reimagining the Container — Line by Line

I wasn't just after "it works." I wanted something more robust, more secure, and more intentional. So, I started refining — with each improvement grounded in reasoning.

---

### ✅ **Why Alpine over Slim?**

Many default to `python:slim`, and for good reason — it's a great balance between size and usability. But I chose `python:alpine` because:

- **It’s even leaner**, minimizing the image size dramatically.
- **Smaller size = smaller attack surface**, reducing vulnerabilities from unused system components.
- And in this case, the app didn’t need extra OS-level dependencies. So Alpine was a perfect fit.

⚠️ *Caveat:* If the app did need system packages or complex builds (like for C-based Python extensions), `slim` might’ve been a safer bet.

---

### 🧪 **Multi-stage Build? Not Needed.**

I considered using a multi-stage build — a common best practice. But think about it:  
When you're already using a slim base like Alpine **for both building and running**, there’s no duplication to eliminate.  
So why overengineer it? Keep it simple. Keep it clean.

---

### 🐍 **Python Version — Why Latest?**

The original repo didn’t lock in a Python version. No `.python-version`, no version pinning in the code or `requirements.txt`.

So I tested it.

Using the latest Python image worked like a charm. At the time of writing, that’s `3.13.3`. I declared it as an `ARG`:

```dockerfile
ARG PYTHON_VERSION=3.13.3
FROM python:${PYTHON_VERSION}-alpine
```

Did I *need* the ARG? No.  
But it adds clarity — and makes future updates effortless.

---

### 🗂️ **Working Directory — /app**

Simple, standard, and explicit:

```dockerfile
WORKDIR /app
```

This sets a clear context and keeps the container organized.

---

### 👤 **Non-root User — Meet `appuser`**

Security matters. Containers should never run as root unless absolutely necessary.  
So I added a non-root user with:

```dockerfile
RUN adduser \
    --disabled-password \
    --gecos "" \
    --home "/nonexistent" \
    --shell "/sbin/nologin" \
    --no-create-home \
    --uid "${UID}"  \
    appuser
```

This user (`appuser`) can’t log in, doesn’t have a home dir, and has minimal permissions.  
**Defense in depth**, baked into the container.

---

### 🧾 **Installing Dependencies — With Cache in Mind**

```dockerfile
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
```

Why copy the requirements separately? Two reasons:

1. **Docker layer caching.** Unless dependencies change, this layer won’t rebuild — speeding things up.
2. **`--no-cache-dir`** ensures pip doesn’t stash anything unnecessary, helping keep the image small.

---

### 📦 **Copying Code**

```dockerfile
COPY . .
```

Once the dependencies are handled, I bring in the rest of the source code.

---

### 👇 **Switching to Non-root Before Runtime**

```dockerfile
USER appuser
```

Once setup is complete, I drop to the least-privileged user before the app starts — a best practice for container hardening.

---

### 🚀 **Entrypoint vs CMD — Flexibility First**

```dockerfile
ENTRYPOINT ["python3"]
CMD ["app.py"]
```

By splitting `ENTRYPOINT` and `CMD`, I keep the command flexible:

- `ENTRYPOINT` is fixed to Python.
- `CMD` defines the default script — but can be overridden at runtime if needed.

This combo makes the container **clean, composable, and developer-friendly**.

---

## ✅ Final Dockerfile

Here’s what it all came together as:

```dockerfile
ARG PYTHON_VERSION=3.13.3
FROM python:${PYTHON_VERSION}-alpine

WORKDIR /app

ARG UID=10001
RUN adduser \
    --disabled-password \
    --gecos "" \
    --home "/nonexistent" \
    --shell "/sbin/nologin" \
    --no-create-home \
    --uid "${UID}"  \
    appuser

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

USER appuser

ENTRYPOINT ["python3"]
CMD ["app.py"]
```

---

### 📌 In Short

This isn’t just a Dockerfile — it’s a statement:  
🧱 **Built small.**  
🛡️ **Locked down.**  
⚙️ **Practical.**  
And every choice made *with intention*.

---
```
![alt text](images/images-comparison.png)
