### 📄 `03-containerized-worker.md`

```markdown
# ⚙️ Containerizing the Worker Service — Building it Right, the Microsoft Way

At first glance, the Worker service seemed simple enough to containerize. My first draft Dockerfile was clean, direct, and worked:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:7.0

WORKDIR /App

COPY . ./

RUN dotnet restore

ENTRYPOINT ["dotnet", "run"]
```

It got the job done.

But "it works" doesn’t mean "it’s ready."

---

## 🧭 Rethinking the Base Image — Why MCR?

You might ask — why not grab a .NET image from Docker Hub?

Because:
- **Microsoft Container Registry (MCR)** is the official source for .NET images — updated, secure, and reliable.
- It includes **SHA digests**, letting me pin the exact image I want.  
  No surprises. No silent updates. Just certainty.

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:7.0.409-bookworm-slim@sha256:...
```

Locking the SHA boosts **reproducibility** and **security** — both are gold in production environments.

---

## 🧱 Multi-stage Build — This Time, It’s Necessary

Unlike the `vote` service, the Worker app needs to be compiled before it can run.  
So, I embraced the power of multi-stage builds:

- **Stage 1 (`build`)**: Use the full SDK image to compile and publish the app.
- **Stage 2 (`runtime`)**: Use a lightweight runtime-only image to run the final executable.

💡 *This pattern separates the "build muscle" from the "run elegance".*

---

## 🔎 Inspecting the Codebase — Targeting .NET 7

To find the right base image, I went to the source — the `Worker.csproj` file.

There it was: `.NET 7`.  
So I locked into version `7.0.409` and continued from there.

---

### 🔧 Build Stage

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:7.0.409-bookworm-slim@sha256:... AS build

WORKDIR /app

COPY Worker.csproj .

RUN dotnet restore

COPY . .

RUN dotnet publish -c Release -o out
```

Let’s break it down:

- **Restore early**: Copying `Worker.csproj` separately ensures Docker caching is used efficiently.  
  Unless the `.csproj` changes, `dotnet restore` won’t re-run.
- **Publish to `-o out`**: Packages the compiled app and its dependencies into a clean output folder.

This keeps the final image **light, fast, and free of dev-time bloat**.

---

### 📦 Runtime Stage

```dockerfile
FROM mcr.microsoft.com/dotnet/runtime:7.0
```

This is the lean, execution-only layer — no compilers, no SDK, just what’s needed to run the Worker.

---

### 👤 Creating a Secure Non-root User

```dockerfile
ARG UID=10001
RUN adduser \
    --disabled-password \
    --gecos "" \
    --home "/nonexistent" \
    --shell "/sbin/nologin" \
    --no-create-home \
    --uid "${UID}" \
    dotnetuser
```

Again, security takes priority:

- No shell access.
- No home directory.
- No login.
- A specific UID, to avoid clashing with existing system users.

---

### 📂 Copy + Permissions + Run

```dockerfile
WORKDIR /app

COPY --from=build /app/out .

USER dotnetuser

ENTRYPOINT ["dotnet", "Worker.dll"]
```

This part brings it all together:

- The final executable is pulled straight from the `build` stage.
- We switch to our locked-down `dotnetuser`.
- The app launches securely and efficiently.

---

## 🧠 Final Dockerfile

```dockerfile
FROM  mcr.microsoft.com/dotnet/sdk:7.0.409-bookworm-slim@sha256:08009349c7f0ad79898f5de5bcf70a3bddcca82dcf736052bd9c02066f042200 AS build

WORKDIR /app

COPY Worker.csproj .

RUN dotnet restore

COPY . .

RUN dotnet publish -c Release -o out 

FROM  mcr.microsoft.com/dotnet/runtime:7.0

ARG UID=10001
RUN adduser \
    --disabled-password \
    --gecos "" \
    --home "/nonexistent" \
    --shell "/sbin/nologin" \
    --no-create-home \
    --uid "${UID}" \
    dotnetuser

WORKDIR /app

COPY --from=build /app/out .

USER dotnetuser

ENTRYPOINT ["dotnet", "Worker.dll"]
```

---

### 🔍 Comparison: Before vs After

|                 | 🚧 Initial Draft                              | 🚀 Final Dockerfile                          |
|-----------------|-----------------------------------------------|----------------------------------------------|
| **Base Image**  | SDK only, single-stage                        | SDK for build, Runtime for execution         |
| **Registry**    | Implicit source (no SHA)                     | MCR + SHA pinned                             |
| **Security**    | Runs as root                                  | Custom non-root user                         |
| **Efficiency**  | Copies everything at once                     | Leverages Docker caching (copy csproj first) |
| **Size**        | Heavy — includes SDK at runtime               | Lean — only runtime essentials               |

---

### 🏁 In Summary

This isn't just about running a .NET worker in a container — it’s about **doing it the right way**:

🔐 **Secure**  
📉 **Slim**  
⚙️ **Built for production**  
📌 **Version-locked and reproducible**

![Result](/images/images-comparison.png)