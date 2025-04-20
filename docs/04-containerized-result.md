Great — let’s level this one up too. You’re pointing out a **critical yet commonly overlooked** detail: how you run your Node.js process in Docker **can make or break** signal handling, graceful shutdowns, and container orchestration behavior.

We’ll make this markdown **educational**, practical, and a little opinionated (in a good way) — just like your style so far.

---

### 📄 `04-containerized-result.md`

```markdown
# 🚦 Containerizing the Result Service — Doing Node.js the Right Way in Docker

The `result` service uses Node.js, so I expected this one to be a breeze — `copy`, `npm install`, `run`. And yes, that works... but working isn't the same as **correct**, especially in a containerized, orchestrated world.

Here’s my initial attempt:

```dockerfile
FROM node

WORKDIR /App

COPY . .

RUN npm install

CMD node server.js
```

It runs. It installs. It starts.  
But it **fails** at an important layer: **signal handling and orchestration compatibility**.

---

## ❌ Common Bad Practices in Node.js Dockerfiles

These patterns **look fine** at first glance but introduce real problems:

```dockerfile
CMD "npm" "start"
CMD ["yarn", "start"]
CMD "node" "server.js"
CMD "start-app.sh"
```

They’re used everywhere — in blogs, tutorials, even production — but let’s dissect **why they’re flawed**.

---

## 🔍 Why These Patterns Fail

### 1. **Improper Signal Forwarding**

Orchestration engines like **Docker**, **Kubernetes**, and **Swarm** need to send signals (e.g., `SIGTERM`, `SIGKILL`) to gracefully stop containers.

But if the process isn’t PID 1 (the first process inside the container), or is wrapped by something else (like a shell or `npm`), signals might not reach your app at all.

For example:

```dockerfile
CMD ["npm", "start"]
```

This runs the Node app **indirectly** through `npm`, and `npm` doesn’t forward signals.  
Let’s prove it.

---

### 🧪 Test: Signal Handling Failure

Add this simple listener to your Node.js app:

```js
function handle(signal) {
   console.log(`Received signal: ${signal}`)
}
process.on('SIGHUP', handle)
```

Run your container, then trigger a signal:

```bash
docker kill --signal=SIGHUP <your_container_name>
```

You’ll see... nothing.  
Because the signal never made it to `node`.  
**That’s a serious issue.**

---

### 2. **Shell Form vs Exec Form in Docker CMD**

There are two ways to write the `CMD` directive:

#### 🚫 Shell Form:

```dockerfile
CMD node server.js
```

This runs inside `/bin/sh -c`, meaning your app isn't PID 1 — and signal handling becomes unreliable.

#### ✅ Exec Form (Recommended):

```dockerfile
CMD ["node", "server.js"]
```

This starts `node` directly as PID 1 — no shell wrapping — so signals go directly to the app.  
**Always use this.**

---

## ✅ Final Dockerfile: Production-Ready and Secure

```dockerfile
FROM node:23.11.0-alpine

ENV NODE_ENV=production

WORKDIR /app

COPY *.json .

COPY . .

RUN npm install

USER node

ENTRYPOINT ["node", "server.js"]
```

### Why this is better:

| Feature           | Before                          | After                                 |
|------------------|----------------------------------|----------------------------------------|
| **Base Image**    | `node` (big, no version pin)     | `node:23.11.0-alpine` (slim + pinned) |
| **Environment**   | ❌ not set                       | ✅ `NODE_ENV=production`              |
| **User**          | root (security risk)             | `node` (non-root, safer)              |
| **Copy Logic**    | All files blindly                | `*.json` first (for Docker caching)   |
| **Run Form**      | Shell form (`CMD node ...`)      | Exec form (`ENTRYPOINT ["node", ...]`) |

---

## 🧠 TL;DR — Key Lessons

1. **Use the exec form** (`["node", "server.js"]`) — no shells, no indirection.
2. **Avoid `npm start` or `yarn start` in containers** — they don’t forward signals.
3. **Always run containers as non-root** unless absolutely necessary.
4. **Pin versions and digests** for consistency and security.
5. **Handle shutdown signals in Node apps** — don’t get caught off guard.

---

### 🧪 Want to dig deeper?

Try sending different signals (`SIGINT`, `SIGTERM`, `SIGUSR1`, etc.) and observe how your container behaves.  
This builds true understanding — and resilience in production.

![Result](/images/images-comparison.png)