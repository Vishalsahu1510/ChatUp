# ChatUp 💬

A real-time, full-stack chat application built with a **microservices architecture**. ChatUp allows users to chat one-on-one, send images, and see live typing indicators and read receipts — all powered by WebSockets, RabbitMQ, and Redis.

---

## 🏗️ Architecture Overview

ChatUp is composed of four independent services:

```
ChatUp/
├── backend/
│   ├── user/      → User Service   (Port 5000)
│   ├── mail/      → Mail Service   (Port 5001)
│   └── chat/      → Chat Service   (Port 5002)
└── frontend/      → Next.js 15 App (Port 3000)
```

```
┌─────────────────┐        ┌─────────────────┐
│   Next.js 15    │◄──────►│  Chat Service   │
│   (Frontend)    │        │  (Express + WS) │
└─────────────────┘        └────────┬────────┘
          │                         │ (HTTP inter-service)
          │                ┌────────▼────────┐
          └───────────────►│  User Service   │
                           │ (Express + Redis│
                           │  + RabbitMQ)    │
                           └────────┬────────┘
                                    │ (RabbitMQ queue)
                           ┌────────▼────────┐
                           │  Mail Service   │
                           │  (Nodemailer)   │
                           └─────────────────┘
```

---

## ✨ Features

- 🔐 **Passwordless OTP Authentication** — Login via email OTP, verified on the backend with rate-limiting
- 💬 **Real-time Messaging** — Powered by Socket.io with room-based chat
- 🖼️ **Image Sharing** — Upload images via Cloudinary
- ✅ **Read Receipts** — Messages are marked as seen when the recipient opens the chat
- ⌨️ **Typing Indicators** — Live "user is typing..." events
- 🟢 **Online Presence** — Track which users are currently online
- 🔔 **Unseen Message Count** — Badge counters for unread messages per chat

---

## 🧩 Services

### 1. User Service (`backend/user`)

Handles authentication and user management.

**Tech:** Express, MongoDB (Mongoose), Redis, RabbitMQ (amqplib), JWT

**Endpoints:**

| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| `POST` | `/api/v1/login` | ❌ | Send OTP to email |
| `POST` | `/api/v1/verify` | ❌ | Verify OTP, get JWT token |
| `GET` | `/api/v1/me` | ✅ | Get logged-in user profile |
| `GET` | `/api/v1/user/all` | ✅ | Get all users |
| `GET` | `/api/v1/user/:id` | ❌ | Get a user by ID |
| `POST` | `/api/v1/update/user` | ✅ | Update display name |

**OTP Flow:**
1. User submits email → OTP generated using `crypto.randomInt` → stored in Redis (TTL: 5 min)
2. Rate limit key set in Redis (TTL: 60 sec) to prevent spam
3. OTP message published to RabbitMQ `send-otp` queue
4. Mail Service consumes the queue and delivers the email
5. User submits OTP → verified against Redis → JWT issued

---

### 2. Chat Service (`backend/chat`)

Handles all messaging, chat rooms, and real-time events.

**Tech:** Express, MongoDB (Mongoose), Socket.io, Cloudinary, Multer, Axios, JWT

**Endpoints:**

| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| `POST` | `/api/v1/chat/new` | ✅ | Create a new 1-on-1 chat |
| `GET` | `/api/v1/chat/all` | ✅ | Get all chats for the current user |
| `POST` | `/api/v1/message` | ✅ | Send a message (text or image) |
| `GET` | `/api/v1/message/:chatId` | ✅ | Get messages for a chat (marks as seen) |

**Socket.io Events:**

| Event | Direction | Description |
|-------|-----------|-------------|
| `connection` | Server ← Client | User connects, added to online map |
| `getOnlineUser` | Server → All | Broadcast updated online user list |
| `joinChat` | Server ← Client | Join a chat room |
| `leaveChat` | Server ← Client | Leave a chat room |
| `typing` | Server ← Client | User started typing |
| `userTyping` | Server → Room | Broadcast typing event to room |
| `stopTyping` | Server ← Client | User stopped typing |
| `userStoppedTyping` | Server → Room | Broadcast stop typing event |
| `newMessage` | Server → Client | Deliver a new message |
| `messagesSeen` | Server → Client | Notify sender that messages were read |
| `disconnect` | Server ← Client | User goes offline, removed from map |

---

### 3. Mail Service (`backend/mail`)

Asynchronous email delivery via RabbitMQ.

**Tech:** amqplib, Nodemailer (Gmail SMTP)

**Behavior:**
- Consumes messages from the `send-otp` RabbitMQ queue (durable)
- Each message contains `{ to, subject, body }`
- Sends the email using Gmail SMTP over port 465 (SSL)
- Acknowledges the message after successful delivery

---

### 4. Frontend (`frontend`)

A modern Next.js 15 application with App Router.

**Tech:** Next.js 15, React 19, TypeScript, Tailwind CSS, Socket.io Client, Axios, js-cookie, react-hot-toast, lucide-react

**Pages / Routes:**

| Route | Description |
|-------|-------------|
| `/` | Landing / redirect page |
| `/login` | Email OTP login |
| `/verify` | OTP verification |
| `/chat` | Main chat interface |
| `/profile` | User profile management |

**Key Components:**

| Component | Description |
|-----------|-------------|
| `ChatSidebar.tsx` | Contact list with unseen count badges |
| `ChatHeader.tsx` | Chat header with user info and online status |
| `ChatMessages.tsx` | Message thread with seen/unseen indicators |
| `MessageInput.tsx` | Text and image message input |
| `VerifyOtp.tsx` | OTP input form |
| `Loading.tsx` | Loading spinner |

**Context Providers:**

| Context | Description |
|---------|-------------|
| `AppContext` | Global user state, selected chat, auth token |
| `SocketContext` | Socket.io connection management |

---

## 🗄️ Data Models

### User (User Service DB)
```typescript
{
  name: string;       // required
  email: string;      // required, unique
  createdAt: Date;
  updatedAt: Date;
}
```

### Chat (Chat Service DB)
```typescript
{
  users: string[];    // array of 2 user IDs
  latestMessage: {
    text: string;
    sender: string;
  };
  createdAt: Date;
  updatedAt: Date;
}
```

### Message (Chat Service DB)
```typescript
{
  chatId: ObjectId;   // ref: Chat
  sender: string;     // user ID
  text?: string;
  image?: {
    url: string;      // Cloudinary URL
    publicId: string; // Cloudinary public ID
  };
  messageType: "text" | "image";
  seen: boolean;
  seenAt?: Date;
  createdAt: Date;
  updatedAt: Date;
}
```

---

## ⚙️ Environment Variables

### `backend/user/.env`
```env
PORT=5000
MONGO_URI=<your_mongodb_connection_string>
JWT_SECRET=<your_jwt_secret>
REDIS_URL=<your_redis_url>
Rabbitmq_Host=<rabbitmq_host>
Rabbitmq_Username=<rabbitmq_username>
Rabbitmq_Password=<rabbitmq_password>
```

### `backend/chat/.env`
```env
PORT=5002
MONGO_URI=<your_mongodb_connection_string>
JWT_SECRET=<same_jwt_secret_as_user_service>
USER_SERVICE=http://localhost:5000
Cloud_Name=<cloudinary_cloud_name>
Api_Key=<cloudinary_api_key>
Api_Secret=<cloudinary_api_secret>
```

### `backend/mail/.env`
```env
PORT=5001
Rabbitmq_Host=<rabbitmq_host>
Rabbitmq_Username=<rabbitmq_username>
Rabbitmq_Password=<rabbitmq_password>
USER=<gmail_address>
PASSWORD=<gmail_app_password>
```

> **Note:** For Gmail, generate an **App Password** from your Google Account security settings (requires 2FA enabled).

---

## 🚀 Getting Started

### Prerequisites

- **Node.js** v18+
- **MongoDB** (local or Atlas)
- **Redis** (local or Upstash)
- **RabbitMQ** (local or CloudAMQP)
- **Cloudinary** account
- **Gmail** account with App Password enabled

### Installation & Running

#### 1. Clone the repository
```bash
git clone <repo-url>
cd ChatUp
```

#### 2. Start the User Service
```bash
cd backend/user
npm install
# Create .env with the variables listed above
npm run dev
```

#### 3. Start the Mail Service
```bash
cd backend/mail
npm install
# Create .env with the variables listed above
npm run dev
```

#### 4. Start the Chat Service
```bash
cd backend/chat
npm install
# Create .env with the variables listed above
npm run dev
```

#### 5. Start the Frontend
```bash
cd frontend
npm install
npm run dev
```

The app will be available at **http://localhost:3000**

### Development Scripts

All backend services share the same scripts:

| Script | Command | Description |
|--------|---------|-------------|
| `dev` | `npm run dev` | Run with TypeScript watch + nodemon |
| `build` | `npm run build` | Compile TypeScript to `dist/` |
| `start` | `npm start` | Run compiled JavaScript |

---

## 🔒 Authentication

ChatUp uses **JWT Bearer token** authentication:

1. After OTP verification, the server issues a signed JWT containing the user object.
2. The frontend stores the token in a cookie (via `js-cookie`).
3. Every authenticated API request includes the header:
   ```
   Authorization: Bearer <token>
   ```
4. Both the **User Service** and **Chat Service** validate tokens independently using the same `JWT_SECRET`.

---

## 🛠️ Tech Stack Summary

| Layer | Technology |
|-------|------------|
| Frontend | Next.js 15, React 19, TypeScript, Tailwind CSS |
| User Service | Node.js, Express 5, TypeScript, MongoDB, Redis, JWT |
| Chat Service | Node.js, Express 5, TypeScript, MongoDB, Socket.io, Cloudinary |
| Mail Service | Node.js, Express 5, TypeScript, Nodemailer |
| Message Broker | RabbitMQ (via amqplib) |
| Real-time | Socket.io 4 |
| Image Storage | Cloudinary |
| Cache / Rate Limit | Redis (Upstash compatible) |
| Database | MongoDB Atlas |

---

## 📁 Project Structure

```
ChatUp/
├── backend/
│   ├── chat/
│   │   └── src/
│   │       ├── config/         # DB, socket, cloudinary, TryCatch
│   │       ├── controllers/    # chat.ts (all chat logic)
│   │       ├── middlewares/    # isAuth.ts, multer.ts
│   │       ├── models/         # Chat.ts, Messages.ts
│   │       └── routes/         # chat.ts
│   ├── mail/
│   │   └── src/
│   │       ├── consumer.ts     # RabbitMQ consumer + Nodemailer
│   │       └── index.ts        # App entry point
│   └── user/
│       └── src/
│           ├── config/         # DB, generateToken, rabbitmq, TryCatch
│           ├── controllers/    # user.ts (auth + user CRUD)
│           ├── middleware/     # isAuth.ts
│           ├── model/          # User.ts
│           └── routes/         # user.ts
└── frontend/
    └── src/
        ├── app/
        │   ├── chat/           # Chat page
        │   ├── login/          # Login page
        │   ├── verify/         # OTP verify page
        │   ├── profile/        # Profile page
        │   ├── layout.tsx      # Root layout
        │   └── page.tsx        # Root redirect
        ├── components/         # ChatSidebar, ChatMessages, etc.
        └── context/            # AppContext, SocketContext
```

---

## 🤝 Contributing

1. Fork the repository
2. Create your feature branch: `git checkout -b feature/your-feature`
3. Commit your changes: `git commit -m 'Add some feature'`
4. Push to the branch: `git push origin feature/your-feature`
5. Open a Pull Request

---

## 📄 License

This project is licensed under the **ISC License**.
