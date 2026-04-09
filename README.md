# AgriAdvisor — System Architecture

## Overview

AgriAdvisor is a full-stack agricultural advisory web app built with **React + TypeScript + Vite** on the frontend, **Express.js** as the backend API layer, **Prisma ORM** with a local **SQLite** database for persistence, and the **Google Gemini AI API** for plant disease detection.

---

## High-Level Architecture

```mermaid
graph TB
    subgraph Client["🌐 Browser (React + Vite)"]
        UI["UI Components\nAuth / Dashboard / Settings"]
        LS["localStorage\n(session token only)"]
        UI -->|"read/write session"| LS
    end

    subgraph Vite_Proxy["⚡ Vite Dev Server (port 3000)"]
        PROXY["Proxy: /api → localhost:4000"]
    end

    subgraph Backend["🖥️ Express Server (port 4000)"]
        RATE["Rate Limiter\nexpress-rate-limit"]
        AUTH_REG["POST /api/auth/register"]
        AUTH_LOG["POST /api/auth/login"]
        AUTH_UPD["PUT /api/auth/update"]
        HASH["SHA-256 Password Hash\n(Node crypto)"]
        RATE --> AUTH_REG
        RATE --> AUTH_LOG
        RATE --> AUTH_UPD
        AUTH_REG --> HASH
        AUTH_LOG --> HASH
    end

    subgraph ORM["🔷 Prisma ORM"]
        PRISMA["PrismaClient\nuser.create / findUnique / update"]
    end

    subgraph DB["🗄️ SQLite Database"]
        USERS["users table\nid · name · email · phone\npassword · fieldType\nlandArea · profilePic"]
    end

    subgraph Gemini["🤖 Google Gemini AI"]
        GM["gemini-flash-latest\ngenerateContent"]
        PROMPT["Structured JSON Prompt\n(Disease Detection Logic)"]
        GM --> PROMPT
    end

    UI -->|"HTTP fetch /api/*"| VITE_PROXY_NODE
    VITE_PROXY_NODE -->|"proxied to"| Backend
    AUTH_REG --> PRISMA
    AUTH_LOG --> PRISMA
    AUTH_UPD --> PRISMA
    PRISMA --> USERS

    UI -->|"@google/genai SDK\n(Direct from browser)"| Gemini
    Gemini -->|"JSON response\ndisease analysis"| UI

    subgraph Vercel["☁️ Vercel Deployment"]
        BUILD["npm run build\nvite build → /dist"]
        FUNC["Serverless Function\n/api/index.ts"]
        REWRITE["Rewrite: /api/(.*) → /api/index.ts"]
        BUILD --> REWRITE
        REWRITE --> FUNC
    end

    VITE_PROXY_NODE["Vite Proxy\n/api → :4000"]
```

---

## Authentication Flow

```mermaid
sequenceDiagram
    participant U as 👤 User
    participant FE as React (Auth.tsx)
    participant BE as Express Server
    participant DB as SQLite (Prisma)

    Note over U,DB: 📝 Registration
    U->>FE: Fill signup form & submit
    FE->>FE: Validate all fields locally
    FE->>BE: POST /api/auth/register
    BE->>BE: Hash password (SHA-256)
    BE->>DB: prisma.user.create(...)
    DB-->>BE: User row created
    BE-->>FE: 201 { message: "success" }
    FE-->>U: Show "Account created! Please login."

    Note over U,DB: 🔐 Login
    U->>FE: Enter email + password
    FE->>BE: POST /api/auth/login
    BE->>BE: Hash input password (SHA-256)
    BE->>DB: prisma.user.findUnique({ email })
    DB-->>BE: User record
    BE->>BE: Compare hashed passwords
    BE-->>FE: 200 { id, name, email, ... } (no password)
    FE->>FE: localStorage.setItem("agri_current_user")
    FE-->>U: Redirect to Dashboard
```

---

## Plant Disease Detection Flow

```mermaid
sequenceDiagram
    participant U as 👤 Farmer
    participant D as Dashboard.tsx
    participant G as Gemini AI API

    U->>D: Upload leaf image(s)
    D->>D: Show thumbnails with ❌ remove
    U->>D: Click "Analyze Leaf"
    D->>D: Convert images to base64 inlineData
    D->>G: generateContent(model: gemini-flash-latest, images + prompt)

    alt Image is NOT a plant
        G-->>D: { status: "not_plant", message: "..." }
        D-->>U: 🔴 Red error banner
    else Image needs more angles
        G-->>D: { status: "need_more_photos", message: "..." }
        D-->>U: 🟡 Amber warning banner
    else Plant is Healthy
        G-->>D: { status: "healthy", disease: "Healthy Plant", recommendations: [...] }
        D-->>U: 🟢 Healthy result card
    else Disease Detected
        G-->>D: { status: "disease", disease: "...", pestName: "...", quantity: "...", ... }
        D-->>U: 🔴 Disease result card with treatment plan
    end
```

---

## Settings / Profile Update Flow

```mermaid
sequenceDiagram
    participant U as 👤 User
    participant S as Settings.tsx
    participant BE as Express Server
    participant DB as SQLite (Prisma)

    U->>S: Click "Edit Profile"
    U->>S: Modify name / phone / field type / land area
    U->>S: Click "Save Changes"
    S->>BE: PUT /api/auth/update { email, name, phone, ... }
    BE->>DB: prisma.user.update({ where: { email }, data: { ... } })
    DB-->>BE: Updated user row
    BE-->>S: 200 { id, name, email, ... } (no password)
    S->>S: setUser(data) + localStorage.setItem("agri_current_user")
    S-->>U: ✅ "Profile updated successfully!"
```

---

## Database Schema

```mermaid
erDiagram
    USERS {
        Int     id          PK "Auto-increment"
        String  name        "Full name"
        String  email       UK "Unique login identifier"
        String  phone       "10-digit phone number"
        String  password    "SHA-256 hashed"
        String  fieldType   "Dry Land / Wet Land / Mixed"
        String  landArea    "in acres"
        String  profilePic  "(optional) base64 or URL"
    }
```

---

## Tech Stack Summary

| Layer | Technology |
|-------|-----------|
| **Frontend Framework** | React 19 + TypeScript |
| **Build Tool** | Vite 6 |
| **Styling** | Tailwind CSS v4 |
| **Animations** | motion/react |
| **Icons** | lucide-react |
| **AI** | Google Gemini (`gemini-flash-latest`) |
| **Backend** | Express.js |
| **ORM** | Prisma |
| **Database** | SQLite (`agriadvisor.db`) |
| **Password Hashing** | Node.js `crypto` SHA-256 |
| **Rate Limiting** | express-rate-limit |
| **Dev Runner** | concurrently + tsx |
| **Deployment** | Vercel (Serverless Functions) |
| **Multilingual** | English · Hindi · Telugu · Tamil |
