# High Level Design (HLD)
## VidyaPaalam: Teach & Learn Platform

**Version:** 1.0  
**Last Updated:** July 22, 2026

---

## 1. System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
│                     (React 19 + Vite)                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Frontend App (netlify.com)                               │   │
│  │ - Pages: Auth, Dashboard, Search, Booking, Chat, Video   │   │
│  │ - Components: Cards, Forms, Modals, Layout               │   │
│  │ - Contexts: Auth, Session, TeacherProfile, Stream*       │   │
│  │ - Storage: Cookies (JWT tokens), localStorage (UI state) │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────┬─────────────────────────────────────────────────┘
                 │ HTTPS
          ┌──────▼──────┐
          │   CDN/SSL   │
          └──────┬──────┘
                 │ HTTPS
┌────────────────▼─────────────────────────────────────────────────┐
│                      API GATEWAY LAYER                           │
│                    (Express.js on Render)                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ API Server (api.vidyapaalam.com)                         │   │
│  │ - CORS: whitelist frontend domain                        │   │
│  │ - Rate Limiting: 5 logins/15min                          │   │
│  │ - Middleware: auth, validation, error handlers           │   │
│  │ - Routes: /auth, /api/teacher-profiles, /api/sessions    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────┬────────────────────┬────────────────────────────┘
                 │                    │
          ┌──────▼──────┐      ┌──────▼──────────┐
          │  Database   │      │ External APIs   │
          │  (MongoDB)  │      │ & Services      │
          └─────────────┘      └─────────────────┘
                                │         │       │
                        ┌───────┘  ┌──────┘  ┌────┘
                        ▼          ▼         ▼
                    Firebase    Razorpay  Cloudinary
                      Auth      Payment    Storage
                        │         │          │
                        ▼         ▼          ▼
                   ┌──────────────────────────────┐
                   │   Stream.io (Chat & Video)   │
                   └──────────────────────────────┘
```

---

## 2. Component Architecture

### 2.1 Frontend (React)

**Pages:**
- `pages/index.jsx` → Landing/home
- `pages/onboarding/OnboardingPage.jsx` → Multi-step setup
- `pages/onboarding/RoleSelection.jsx` → Role picker
- `pages/onboarding/StudentOnboarding.jsx` → Student onboarding
- `pages/onboarding/TeacherOnboarding.jsx` → Teacher onboarding + availability
- `pages/student/FindTeacher.jsx` → Search + filter
- `pages/student/TeacherProfile.jsx` → Teacher detail view
- `pages/student/BookSession.jsx` → Calendar + payment
- `pages/student/StudentOverview.jsx` → Dashboard
- `pages/student/Favorites.jsx` → Favorites list (v1.1)
- `pages/teacher/TeacherOverview.jsx` → Teacher dashboard
- `pages/teacher/TeacherProfileEdit.jsx` → Profile management + media upload
- `pages/teacher/TeacherAvailability.jsx` → Slot management
- `pages/teacher/TeacherRatings.jsx` → Reviews (v1.1)
- `pages/ChatPage.jsx` → Stream Chat UI
- `pages/VideoCallPage.jsx` → Stream Video UI

**Contexts (State Management):**
- `AuthContext.jsx` → User auth, login/signup, profile updates, token refresh
- `SessionContext.jsx` → Upcoming/past sessions, payment handler, session fetching
- `TeacherProfileContext.jsx` → Teacher profile CRUD, media file handling
- `StreamChatContext.jsx` → Stream Chat client init & lifecycle
- `StreamVideoContext.jsx` → Stream Video client init & lifecycle

**Components:**
- `layout/Navbar.jsx` → Navigation
- `layout/Footer.jsx` → Footer
- `layout/AppLayout.jsx` → Wrapper with nav/footer
- `layouts/StudentLayout.jsx` → Student sidebar + outlet
- `layouts/TeacherLayout.jsx` → Teacher sidebar + outlet
- `auth/AuthModals.jsx` → Login/signup modal
- `auth/SignInForm.jsx` → Login form
- `auth/SignUpForm.jsx` → Signup form
- `ui/PrivateRoute.jsx` → Route guard (role-based)
- `ui/calendar.jsx` → Date picker (shadcn)
- `ui/card.jsx` → Card wrapper
- `skills/SkillGrid.jsx` → Skill cards
- `skills/SkillCard.jsx` → Individual skill
- `skills/SkillCategoryFilter.jsx` → Filter by category

**Utilities & Hooks:**
- `api/axios.js` → Axios instance + interceptors (auto-refresh tokens)
- `contexts/AuthContext.jsx` → useAuth() hook
- `contexts/SessionContext.jsx` → useSession() hook
- `hooks/useRazorpayScript.js` → Load Razorpay script
- `hooks/useTeacherProfile.js` → Teacher profile operations
- `firebase.js` → Firebase init

---

### 2.2 Backend (Node.js + Express)

**Controllers:**
- `authController.js` → Register, login, logout, profile CRUD, role management, Firebase auth
- `teacherController.js` → Teacher profile CRUD, media upload via Cloudinary
- `sessionController.js` → Session CRUD, fetch sessions (student/teacher)
- `paymentController.js` → Payment order creation, Razorpay webhook
- `streamController.js` → Generate Stream Chat/Video tokens

**Routes:**
- `/auth/*` → Authentication endpoints
- `/api/teacher-profiles/*` → Teacher profile endpoints
- `/api/student/sessions`, `/api/teacher/sessions` → Session endpoints
- `/api/create-payment-order` → Payment order creation
- `/api/razorpay-webhook` → Webhook handler (raw body, signature verify)
- `/api/stream/*` → Stream token generation

**Middleware:**
- `authMiddleware.js` → JWT verification (protect), role authorization
- `validation.js` → Input validation rules (signup/login)
- `uploadMiddleware.js` → Multer file upload (avatar, video, gallery photos)

**Services:**
- `firebaseAdmin.js` → Firebase Admin SDK initialization
- `cloudinary.js` → Upload/delete files from Cloudinary CDN
- `rateLimiter.js` → Express rate limit config

**Models (MongoDB):**
- `User.js` → User document (name, email, password hash, role, skills, availability, Firebase UID)
- `Teacher.js` (TeacherProfile) → Teacher document (userId ref, qualifications, avatar, video, gallery)
- `Session.js` → Session document (studentId, teacherId, dateTime, paymentId)
- `BlackListedToken.js` → Blacklisted JWT tokens (TTL index for auto-cleanup)

**Config:**
- `db.js` → MongoDB connection
- `rateLimiter.js` → Rate limiting rules

---

### 2.3 External Services

**Firebase (Authentication)**
- OAuth provider (Google sign-in)
- Email verification (planned)
- User session tokens
- Service account for backend verification

**Razorpay (Payment Gateway)**
- Order creation API
- Webhook signature verification
- Payment events (payment.captured, order.paid)
- Test mode for development

**Cloudinary (Media Storage)**
- Image/video upload (base64 encoding)
- Secure URLs returned
- Public/private folders
- Automatic cleanup on profile delete

**Stream.io (Chat & Video)**
- User creation/upsert on Stream servers
- Token generation (JWT-based)
- Real-time messaging
- Video call infrastructure
- Participant management

---

## 3. Data Flow Diagrams

### 3.1 Authentication Flow
```
User Sign-up
├─ Email/Password
│  ├─ Validate input (length, format, strength)
│  ├─ Hash password (bcrypt 12 rounds)
│  ├─ Save to MongoDB User
│  └─ Issue JWT tokens (access + refresh)
│
└─ Google OAuth (Firebase)
   ├─ Firebase popup sign-in
   ├─ Verify idToken on backend
   ├─ Upsert User (Firebase UID)
   └─ Issue JWT tokens

Token Refresh
├─ Check refresh token in cookies
├─ Verify JWT signature
├─ Check blacklist (for logout)
└─ Issue new access token

Logout
├─ Add refresh token to blacklist
└─ Clear cookies
```

### 3.2 Teacher Profile Setup Flow
```
Teacher Onboarding
├─ Step 1: Role Selection
│  └─ Patch /auth/profile/role → role = "teacher"
│
├─ Step 2: Basic Info
│  └─ Patch /auth/profile → name, phone, bio
│
├─ Step 3: Teaching Skills
│  └─ Patch /auth/profile/teaching-skills → []
│
└─ Step 4: Availability
   └─ Patch /auth/profile/availability → [{date, slots[]}]

Teacher Profile Creation
├─ POST /api/teacher-profiles (multipart/form-data)
├─ Files uploaded to Cloudinary (avatar, video, gallery)
├─ URLs stored in TeacherProfile
└─ isProfileComplete flag = true

Teacher Profile Edit
├─ PUT /api/teacher-profiles/:id (multipart)
├─ Delete old media from Cloudinary
├─ Upload new media
└─ Update document
```

### 3.3 Session Booking → Payment → Session Creation Flow
```
Student Booking Flow
├─ GET /api/teacher-profiles/teacher-profiles-for-booking/:id
├─ Parse availability (dates, slots)
│
├─ Calendar Date Selection
│  └─ Filter available dates (has available slots)
│
├─ Time Slot Selection
│  └─ Display slots for selected date
│
└─ Review & Pay
   ├─ POST /api/create-payment-order
   │  ├─ Amount, currency, teacherData (name, dateTime, startTime, endTime, teacherId)
   │  ├─ Create Razorpay order
   │  └─ Return orderId, key, teacherData
   │
   ├─ Razorpay Modal
   │  ├─ User enters card/UPI details
   │  └─ Razorpay processes payment
   │
   └─ Razorpay Webhook (async)
      ├─ POST /api/razorpay-webhook (raw body)
      ├─ Verify signature (SHA256)
      ├─ Extract userId, teacherData from order notes
      │
      ├─ Create Session
      │  ├─ POST Session doc (studentId, teacherId, paymentId, dateTime, startTime, endTime)
      │  └─ Set paymentAcknowledged = true
      │
      └─ Update Teacher Availability
         ├─ Find availability entry for booked date
         ├─ Mark slot (startTime-endTime) as available = false
         └─ Save User.availability
```

### 3.4 Video Call Flow
```
Video Call Initiation
├─ Student/Teacher clicks "Join Call"
├─ Frontend fetches callId from URL (sorted user IDs)
│
├─ Stream Video Token Generation
│  ├─ GET /api/stream/video-token (protected)
│  ├─ Backend creates token (JWT, signed with Stream secret)
│  └─ Return token, apiKey, userId
│
├─ Initialize StreamVideoClient
│  ├─ StreamVideoClient(apiKey, user, token)
│  ├─ User = {id, name, image}
│  └─ Connect to Stream servers
│
├─ Join Call
│  ├─ Create call object (videoClient.call('default', callId))
│  ├─ await call.join({create: true})
│  └─ WebRTC connection established
│
├─ Render Video Grid
│  ├─ SpeakerLayout (desktop) or grid (mobile)
│  ├─ useCallStateHooks().useParticipants()
│  └─ ParticipantView for each participant
│
└─ Call Controls
   ├─ Mute/unmute audio
   ├─ Turn on/off video
   ├─ Screen share
   └─ End call → call.leave() → redirect to dashboard
```

---

## 4. Security Architecture

**Authentication:**
- Passwords hashed with bcrypt (12 rounds salt)
- JWT tokens (access: 15min, refresh: 7days) stored in httpOnly cookies
- CORS restricted to frontend domain
- Refresh token blacklist (MongoDB TTL index)

**Authorization:**
- Middleware checks role (student/teacher/admin)
- Routes protected with `@protect` middleware
- User can only update own profile/sessions

**API Security:**
- Input validation (express-validator rules)
- Rate limiting on login (5 attempts/15min)
- HTTPS enforced (SSL/TLS)
- Webhook signature verification (Razorpay SHA256)
- No sensitive data in logs

**File Upload:**
- Multer memory storage (no disk write)
- File type validation (image/video only)
- Size limits (100MB max)
- Uploaded to Cloudinary (third-party CDN)

---

## 5. Scalability & Performance

**Database Optimization:**
- Indexes on: userId, email, availability.date, paymentId
- Connection pooling (Mongoose)
- Query optimization (populate selectively)

**Frontend Optimization:**
- Code splitting (Vite lazy loading)
- Image lazy loading (Cloudinary responsive images)
- Component memoization (React.memo, useMemo)
- Framer Motion for smooth animations

**API Optimization:**
- Stateless design (no server sessions)
- Pagination (future enhancement)
- Caching headers (Cache-Control)
- Compression (gzip via Express)

**Deployment:**
- Frontend: Netlify (CDN, auto-deploy on push)
- Backend: Render (auto-scaling, zero-downtime deploys)
- Database: MongoDB Atlas (managed, auto-backup)
- Media: Cloudinary (CDN, auto-optimization)

---

## 6. Error Handling & Logging

**Frontend:**
- Try-catch in async functions
- Toast notifications for user feedback
- Error boundaries (future)
- Console logs (dev), no errors in prod

**Backend:**
- Error middleware catches unhandled errors
- HTTP status codes (400, 401, 403, 404, 500)
- Error messages (generic for users, detailed in logs)
- Winston logging (future enhancement)

**Monitoring:**
- Sentry (future) for error tracking
- Render logs for server monitoring
- MongoDB Atlas alerts for DB issues

---

## 7. API Integration Points

**Client ↔ API:**
- Axios interceptor for token refresh
- CORS credentials included
- Content-Type: application/json or multipart/form-data

**API ↔ Database:**
- Mongoose ODM
- Connection pooling
- Async/await pattern

**API ↔ External APIs:**
- Firebase Admin SDK (verify tokens)
- Razorpay SDK (create orders, fetch details)
- Cloudinary SDK (upload, delete files)
- Stream SDK (create users, tokens)

---

## 8. Deployment Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    GitHub (Source)                       │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ Repository: kalviumcommunity/vidyapaalam             │ │
│  │ Branches: main, dev, feature/*                       │ │
│  │ CI/CD: GitHub Actions (future)                       │ │
│  └──────────────────────────────────────────────────────┘ │
└────────────────────┬─────────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        ▼                         ▼
   ┌─────────────┐         ┌──────────────┐
   │   Netlify   │         │    Render    │
   ├─────────────┤         ├──────────────┤
   │ Frontend    │         │ Backend      │
   │ Build: vite │         │ Start: node  │
   │ Deploy: FE  │         │ Deploy: API  │
   │ URL: .app   │         │ URL: .com    │
   └──────┬──────┘         └───────┬──────┘
          │                        │
          ▼                        ▼
   ┌─────────────┐         ┌──────────────┐
   │  Cloudinary │         │ MongoDB      │
   │  (CDN)      │         │ (Atlas)      │
   └─────────────┘         └──────────────┘
```

---

## 9. Integration with Third-Party Services

| Service | Purpose | Integration | Status |
|---------|---------|-------------|--------|
| Firebase | OAuth, user verification | Admin SDK | ✅ Live |
| Razorpay | Payment processing | SDK + Webhook | ✅ Live |
| Cloudinary | Media storage | SDK, multer | ✅ Live |
| Stream.io | Chat & video | SDKs, token generation | ✅ Live |
| MongoDB | Database | Mongoose | ✅ Live |
| Netlify | Frontend hosting | Git push deploy | ✅ Live |
| Render | Backend hosting | Git push deploy | ✅ Live |

---

## 10. Sequence Diagrams

### 10.1 Payment & Session Creation Sequence
```
Student         Frontend      Backend          Razorpay       DB
  │                │            │                 │            │
  ├─ Select date ──┤            │                 │            │
  │                │            │                 │            │
  ├─ Pick slot ────┤            │                 │            │
  │                │            │                 │            │
  ├─ Click Pay ────┼─ POST /create-payment-order  │            │
  │                │            ├─ Create order   │            │
  │                │            ├──────────────────┤ Order ID  │
  │                │◄───────────┤ Return orderId  │            │
  │                │            │                 │            │
  │  Razorpay ◄────┼─ Load Modal│                 │            │
  │  Modal          │            │                 │            │
  │                 │            │                 │            │
  ├─ Pay with card ├────────────────────────────────┤           │
  │                 │            │    Webhook      │           │
  │                 │            │◄─ payment.captured          │
  │                 │            ├─ Verify signature           │
  │                 │            │                 │           │
  │                 │            ├─ Create Session────────────┤
  │                 │            │    (POST)       │    Save   │
  │                 │            │                 │           │
  │                 │            ├─ Update availability       │
  │                 │            │    (Mark slot used)        │
  │                 │            │                 │◄─ Update │
  │                 │            │                 │           │
  │◄────────────────┼───────────┤ Success response│           │
  │ (Session ready) │            │                 │           │
```

---

## 11. Monitoring & Maintenance

**Health Checks:**
- Backend: `/` endpoint returns "Welcome" (HTTP 200)
- Frontend: Deployed via Netlify (auto-health check)
- Database: Mongoose connection logs

**Alerts (Future):**
- Payment webhook failures
- Database connection errors
- API response time > 1s
- Cloudinary upload failures

---

## 12. Key Technical Decisions

| Decision | Rationale | Alternative |
|----------|-----------|-------------|
| React 19 + Vite | Fast dev, modern JS, ESM | Vue, Svelte |
| Express.js | Lightweight, Node.js standard | Fastify, Koa |
| MongoDB | Flexible schema, JSON docs | PostgreSQL, DynamoDB |
| JWT + Cookies | Stateless, secure, CSRF protection | Sessions, OAuth tokens |
| Razorpay | India-first, easy integration | Stripe, PayPal |
| Cloudinary | CDN, auto-optimization | AWS S3, Firebase Storage |
| Stream.io | Reliable, managed infrastructure | Agora, Twilio |

---

**End of HLD**
