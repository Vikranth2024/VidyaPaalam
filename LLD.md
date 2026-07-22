# Low Level Design (LLD)
## VidyaPaalam: Teach & Learn Platform

**Version:** 1.0  
**Last Updated:** July 22, 2026

---

## 1. Database Schema

### 1.1 User Collection

```javascript
{
  _id: ObjectId,
  name: String,
  email: String (unique, sparse),
  password: String (selected: false, only on password fields),
  firebaseUid: String (optional, for Google OAuth),
  picture: String (Cloudinary URL),
  bio: String,
  phoneNumber: String,
  role: Enum["student", "teacher", "admin"] (default: null),
  interestedSkills: [String], // for students
  teachingSkills: [String], // for teachers
  availability: [
    {
      date: Date (stored as UTC midnight),
      slots: [
        {
          startTime: String (format: "HH:MM"),
          endTime: String (format: "HH:MM"),
          available: Boolean (default: true)
        }
      ]
    }
  ],
  teacherOnboardingComplete: Boolean (default: false),
  activeToken: String (refresh token, selected: false),
  paymentAcknowledged: Boolean (default: false),
  lastPaymentId: String (Razorpay payment ID),
  createdAt: Date (auto),
  updatedAt: Date (auto)
}

Indexes:
- email (unique)
- firebaseUid
- role
- availability.date
```

**Constraints:**
- Email: Must be valid, unique (sparse allows null)
- Password: Min 8 chars, must contain uppercase, lowercase, digit, special char
- Phone: 10-15 digits
- Role: Only ["student", "teacher", "admin"]

---

### 1.2 TeacherProfile Collection

```javascript
{
  _id: ObjectId,
  userId: ObjectId (ref: User, unique),
  name: String (required),
  email: String (required, unique),
  title: String (e.g., "Yoga Instructor"),
  phone: String,
  aboutMe: String,
  skills: [String], // e.g., ["Yoga", "Meditation"]
  experience: String (e.g., "5+ years"),
  hourlyRate: Number (min: 0, required),
  qualifications: [String], // e.g., ["B.A. in Dance", "Certified Yoga Trainer"]
  avatar: {
    url: String (Cloudinary URL),
    publicId: String (for deletion)
  },
  videoUrl: {
    url: String (Cloudinary URL, intro video),
    publicId: String
  },
  galleryPhotos: [
    {
      url: String (Cloudinary URL),
      publicId: String,
      name: String (optional filename)
    }
  ],
  isProfileComplete: Boolean (auto-calculated on save),
  createdAt: Date (auto),
  updatedAt: Date (auto)
}

Indexes:
- userId (unique)
- email (unique)
- skills
- hourlyRate

isProfileComplete Calculation:
- true if: name && email && hourlyRate > 0 && experience && skills.length > 0
- false otherwise
```

**Constraints:**
- userId: Must reference existing User with role="teacher"
- Email: Unique across collection
- HourlyRate: Non-negative number
- Skills: At least 1 required for profile completion
- Media: Cloudinary URLs (https://)

---

### 1.3 Session Collection

```javascript
{
  _id: ObjectId,
  teacherName: String (required),
  teacherInitials: String (e.g., "JD" for John Doe),
  skill: String (subject being taught),
  dateTime: Date (e.g., 2025-04-15T00:00:00Z, stored as UTC),
  studentId: ObjectId (ref: User, required),
  teacherId: ObjectId (ref: User, required),
  paymentId: String (Razorpay payment ID, required),
  startTime: String (format: "HH:MM", e.g., "14:30"),
  endTime: String (format: "HH:MM", e.g., "15:30"),
  timeRange: String (computed on save, format: "2:30 PM - 3:30 PM"),
  createdAt: Date (auto),
  updatedAt: Date (auto)
}

Indexes:
- studentId
- teacherId
- dateTime
- paymentId (unique)

Pre-save Hook:
- Compute timeRange from startTime & endTime
- Format times to 12-hour format (e.g., "2:30 PM")
```

**Constraints:**
- StudentId: Must reference existing User with role="student"
- TeacherId: Must reference existing User with role="teacher"
- PaymentId: Razorpay ID, unique per session
- StartTime/EndTime: Valid 24-hour format (validation on create)

---

### 1.4 BlacklistedToken Collection

```javascript
{
  _id: ObjectId,
  token: String (required, unique),
  expiresAt: Date (default: now + 24 hours, required)
}

Indexes:
- token (unique)
- expiresAt (TTL: auto-delete after expiry)
```

**Purpose:** Store logout tokens (refresh tokens) to prevent reuse.

---

## 2. API Endpoints Specification

### 2.1 Authentication Endpoints

#### POST /auth/register
**Purpose:** Create new user account

**Request:**
```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "SecurePass123!"
}
```

**Validation:**
- name: 3-50 chars, letters + spaces only
- email: Valid email format, unique in DB
- password: Min 8 chars, 1 upper, 1 lower, 1 digit, 1 special char

**Response (201):**
```json
{
  "message": "User registered successfully",
  "user": {
    "id": "65a1b2c3...",
    "name": "John Doe",
    "email": "john@example.com",
    "role": null,
    "isVerified": false,
    "paymentAcknowledged": false
  }
}
```

**Cookies Set:**
- accessToken (15min, httpOnly)
- refreshToken (7days, httpOnly)

**Errors:**
- 400: Missing fields, validation failed
- 400: User already exists

---

#### POST /auth/login
**Purpose:** User login

**Request:**
```json
{
  "email": "john@example.com",
  "password": "SecurePass123!"
}
```

**Response (200):**
```json
{
  "message": "Logged in successfully",
  "user": { /* same as register */ }
}
```

**Errors:**
- 400: Invalid credentials

---

#### POST /auth/firebase-google
**Purpose:** Google OAuth login/signup via Firebase

**Request:**
```json
{
  "idToken": "eyJhbGci..."
}
```

**Backend:**
1. Verify idToken with Firebase Admin SDK
2. Extract firebaseUid, email, name, picture
3. Find or create User
4. Issue JWT tokens

**Response (200):**
```json
{
  "message": "Authentication successful with Firebase Google.",
  "user": {
    "id": "65a1b2c3...",
    "firebaseUid": "abc123...",
    "email": "john@google.com",
    "name": "John Doe",
    "picture": "https://...",
    "role": null,
    "isVerified": true,
    "paymentAcknowledged": false
  }
}
```

**Errors:**
- 400: Missing idToken
- 401: Invalid/expired idToken

---

#### POST /auth/logout
**Purpose:** Logout user (blacklist refresh token)

**Request:** (No body, cookies auto-attached)

**Backend:**
1. Extract refreshToken from cookies
2. Add to BlacklistedToken collection
3. Clear cookies

**Response (200):**
```json
{
  "message": "Logged out successfully"
}
```

---

#### POST /auth/refresh-token
**Purpose:** Get new access token using refresh token

**Request:** (No body, cookies auto-attached)

**Backend:**
1. Extract refreshToken from cookies
2. Check if blacklisted
3. Verify JWT signature
4. Issue new access + refresh tokens
5. Update user.activeToken

**Response (200):**
```json
{
  "message": "Access token refreshed successfully"
}
```

**Errors:**
- 401: No refresh token provided
- 401: Refresh token expired
- 401: Refresh token invalid
- 401: Token blacklisted

---

#### GET /auth/profile
**Protection:** @protect middleware (JWT required)

**Purpose:** Fetch current user profile

**Response (200):**
```json
{
  "id": "65a1b2c3...",
  "name": "John Doe",
  "email": "john@example.com",
  "role": "teacher",
  "phoneNumber": "+1234567890",
  "bio": "Certified yoga instructor",
  "interestedSkills": ["Yoga"],
  "teachingSkills": ["Yoga", "Meditation"],
  "teacherOnboardingComplete": true,
  "isVerified": true,
  "availability": [ /* array */ ],
  "firebaseUid": "abc123...",
  "picture": "https://...",
  "paymentAcknowledged": true
}
```

**Errors:**
- 401: Not authorized (no token)
- 404: User not found

---

#### PATCH /auth/profile/role
**Protection:** @protect

**Purpose:** Set user role (student or teacher)

**Request:**
```json
{
  "role": "teacher"
}
```

**Validation:**
- role: Must be "student" or "teacher"

**Response (200):**
```json
{
  "message": "Role updated successfully",
  "user": {
    "id": "65a1b2c3...",
    "role": "teacher",
    "paymentAcknowledged": false
  }
}
```

**Errors:**
- 400: Invalid role
- 401: Authentication required

---

#### PATCH /auth/profile
**Protection:** @protect

**Purpose:** Update basic profile info

**Request:**
```json
{
  "name": "Jane Doe",
  "phoneNumber": "+1234567890",
  "bio": "New bio",
  "picture": "https://cloudinary.com/..."
}
```

**Response (200):**
```json
{
  "message": "Profile updated successfully",
  "user": { /* full user object */ }
}
```

---

#### PATCH /auth/profile/interested-skills
**Protection:** @protect

**Purpose:** Update student's interested skills

**Request:**
```json
{
  "interestedSkills": ["Yoga", "Programming", "Cooking"]
}
```

**Validation:**
- interestedSkills: Must be array

**Response (200):**
```json
{
  "message": "Interested skills updated successfully",
  "user": { /* full user */ }
}
```

---

#### PATCH /auth/profile/teaching-skills
**Protection:** @protect

**Purpose:** Update teacher's teaching skills

**Request:**
```json
{
  "teachingSkills": ["Yoga", "Meditation"]
}
```

**Response (200):**
```json
{
  "message": "Teaching skills updated successfully",
  "user": { /* full user */ }
}
```

---

#### PATCH /auth/profile/availability
**Protection:** @protect

**Purpose:** Update teacher's availability slots

**Request:**
```json
[
  {
    "date": "2025-04-15",
    "slots": [
      {
        "startTime": "09:00",
        "endTime": "10:00",
        "available": true
      },
      {
        "startTime": "10:30",
        "endTime": "11:30",
        "available": true
      }
    ]
  },
  {
    "date": "2025-04-16",
    "slots": [ /* ... */ ]
  }
]
```

**Backend:**
1. Validate dates (no past dates)
2. Validate slots (endTime > startTime)
3. Update User.availability array
4. Merge or replace existing dates

**Response (200):**
```json
{
  "message": "Availability updated successfully",
  "user": {
    "id": "65a1b2c3...",
    "availability": [ /* updated array */ ]
  }
}
```

**Errors:**
- 400: Invalid date/slot format
- 401: Authentication required

---

### 2.2 Teacher Profile Endpoints

#### GET /api/teacher-profiles
**Protection:** None (public)

**Purpose:** List all teacher profiles

**Response (200):**
```json
[
  {
    "_id": "65a1b2c3...",
    "userId": "65a1b2c3...",
    "name": "Rajesh Kumar",
    "email": "rajesh@example.com",
    "title": "Certified Yoga Instructor",
    "avatar": {
      "url": "https://cloudinary.com/.../avatar.jpg",
      "publicId": "vidyapaalam/avatars/..."
    },
    "skills": ["Yoga", "Meditation"],
    "hourlyRate": 500,
    "experience": "5+ years",
    "isProfileComplete": true
  },
  /* ... more profiles ... */
]
```

---

#### GET /api/teacher-profiles/:id
**Protection:** None (public)

**Purpose:** Fetch single teacher profile with availability

**Response (200):**
```json
{
  "_id": "65a1b2c3...",
  "name": "Rajesh Kumar",
  "avatarUrl": "https://cloudinary.com/.../avatar.jpg",
  "teachingSkills": ["Yoga", "Meditation"],
  "rating": 4.8,
  "hourlyRate": 500,
  "bio": "Certified yoga instructor with 5+ years experience",
  "availability": [
    {
      "date": "2025-04-15T00:00:00.000Z",
      "slots": [
        {
          "startTime": "09:00",
          "endTime": "10:00",
          "available": true
        }
      ]
    }
  ],
  "videoUrl": "https://cloudinary.com/.../intro.mp4",
  "galleryPhotos": [
    {
      "url": "https://cloudinary.com/.../photo1.jpg",
      "publicId": "vidyapaalam/gallery/...",
      "name": "Class session"
    }
  ]
}
```

---

#### GET /api/teacher-profiles/teacher-profiles-for-booking/:id
**Protection:** None

**Purpose:** Fetch teacher profile with formatted availability for booking flow

**Response (200):** (Same as GET /:id but optimized for booking)

---

#### POST /api/teacher-profiles
**Protection:** @protect, @authorizeRoles('teacher')

**Purpose:** Create teacher profile (with file uploads)

**Request:** multipart/form-data
- avatar (file)
- video (file, intro video)
- galleryPhotos (files, multiple)
- name (text)
- email (text)
- title (text)
- phone (text)
- aboutMe (text)
- skills (JSON array)
- experience (text)
- hourlyRate (number)
- qualifications (JSON array)

**File Upload:**
1. Receive multipart data via multer
2. Upload avatar to Cloudinary/avatars
3. Upload video to Cloudinary/videos
4. Upload galleryPhotos to Cloudinary/gallery
5. Store URLs + publicIds in TeacherProfile

**Response (201):**
```json
{
  "_id": "65a1b2c3...",
  "userId": "65a1b2c3...",
  "name": "Rajesh Kumar",
  "email": "rajesh@example.com",
  "title": "Certified Yoga Instructor",
  "avatar": {
    "url": "https://cloudinary.com/.../avatar.jpg",
    "publicId": "vidyapaalam/avatars/abc123"
  },
  "skills": ["Yoga", "Meditation"],
  "hourlyRate": 500,
  "experience": "5+ years",
  "qualifications": ["B.A. in Dance", "Certified Yoga Trainer"],
  "videoUrl": { /* ... */ },
  "galleryPhotos": [ /* ... */ ],
  "isProfileComplete": true
}
```

**Errors:**
- 400: Validation failed
- 409: Profile already exists
- 500: Cloudinary upload failed

---

#### PUT /api/teacher-profiles/:id
**Protection:** @protect, @authorizeRoles('teacher', 'admin')

**Purpose:** Update teacher profile (with file replacements)

**Request:** multipart/form-data (same fields as POST)

**File Handling:**
1. If avatar provided: Upload new, delete old from Cloudinary
2. If video provided: Upload new, delete old
3. If galleryPhotos provided: Replace or append
4. Update document

**Response (200):** (Updated profile object)

**Errors:**
- 404: Profile not found
- 403: Not authorized
- 400: Validation failed

---

#### DELETE /api/teacher-profiles/:id
**Protection:** @protect, @authorizeRoles('teacher', 'admin')

**Purpose:** Delete teacher profile (with media cleanup)

**Backend:**
1. Find profile
2. Delete avatar, video, gallery photos from Cloudinary
3. Delete profile document
4. Return success

**Response (200):**
```json
{
  "message": "Teacher profile removed successfully."
}
```

**Errors:**
- 404: Profile not found
- 403: Not authorized

---

### 2.3 Session Endpoints

#### GET /api/student/sessions
**Protection:** @protect

**Purpose:** Fetch student's sessions (upcoming + past)

**Response (200):**
```json
{
  "upcoming": [
    {
      "_id": "65a1b2c3...",
      "teacherName": "Rajesh Kumar",
      "teacherInitials": "RK",
      "skill": "Yoga",
      "dateTime": "2025-04-20T00:00:00.000Z",
      "startTime": "14:30",
      "endTime": "15:30",
      "timeRange": "2:30 PM - 3:30 PM",
      "paymentId": "pay_abc123...",
      "teacherId": {
        "_id": "65a1b2c3...",
        "name": "Rajesh Kumar"
      },
      "studentId": "65a1b2c3...",
      "createdAt": "2025-04-15T10:00:00.000Z"
    }
  ],
  "past": [
    /* ... older sessions ... */
  ]
}
```

**Logic:**
- Fetch Session docs where studentId = req.user.id
- Populate teacherId (name only)
- Sort by dateTime + startTime
- Separate: if(dateTime + endTime > now) → upcoming, else → past

---

#### POST /api/student/sessions
**Protection:** @protect

**Purpose:** Create session (called by webhook, not by frontend directly)

**Request:**
```json
{
  "teacherName": "Rajesh Kumar",
  "teacherInitials": "RK",
  "skill": "Yoga",
  "dateTime": "2025-04-20T00:00:00.000Z",
  "startTime": "14:30",
  "endTime": "15:30",
  "paymentId": "pay_abc123...",
  "teacherId": "65a1b2c3..."
}
```

**Response (201):**
```json
{
  "_id": "65a1b2c3...",
  "teacherName": "Rajesh Kumar",
  /* ... full session object ... */
}
```

---

#### GET /api/teacher/sessions
**Protection:** @protect

**Purpose:** Fetch teacher's sessions (upcoming + past)

**Response (200):**
```json
{
  "upcomingSessions": [
    {
      "_id": "65a1b2c3...",
      "teacherName": "Rajesh Kumar",
      "skill": "Yoga",
      "dateTime": "2025-04-20T00:00:00.000Z",
      "startTime": "14:30",
      "endTime": "15:30",
      "timeRange": "2:30 PM - 3:30 PM",
      "paymentId": "pay_abc123...",
      "studentId": {
        "_id": "65a1b2c3...",
        "name": "Priya",
        "email": "priya@example.com"
      },
      "teacherId": "65a1b2c3..."
    }
  ],
  "pastSessions": [ /* ... */ ]
}
```

**Logic:** (Same as student, but for teacherId filter)

---

### 2.4 Payment Endpoints

#### POST /api/create-payment-order
**Protection:** @protect

**Purpose:** Create Razorpay order for session booking

**Request:**
```json
{
  "amount": 500,
  "currency": "INR",
  "teacherData": {
    "name": "Rajesh Kumar",
    "dateTime": "2025-04-20",
    "startTime": "14:30",
    "endTime": "15:30",
    "teacherId": "65a1b2c3...",
    "skill": "Yoga"
  }
}
```

**Backend:**
1. Validate amount, teacherData
2. Generate receipt ID (userIdShort + timestampShort)
3. Create Razorpay order via SDK
4. Store teacherData in order notes (JSON string)

**Response (200):**
```json
{
  "orderId": "order_abc123...",
  "amount": 500,
  "currency": "INR",
  "teacherData": {
    "name": "Rajesh Kumar",
    "dateTime": "2025-04-20",
    /* ... */
  }
}
```

**Errors:**
- 400: Missing fields
- 500: Razorpay API error

---

#### POST /api/razorpay-webhook
**Protection:** None (but signature verified)

**Purpose:** Handle Razorpay payment events (async)

**Webhook Event:** payment.captured, order.paid

**Backend:**
1. Extract raw body (Buffer)
2. Calculate HMAC-SHA256 signature
3. Verify against x-razorpay-signature header
4. Parse webhook payload (JSON)
5. Extract paymentEntity, orderEntity
6. Extract userId, teacherData from order.notes
7. Find Student User, update paymentAcknowledged = true
8. Create Session doc (POST Session)
9. Find Teacher User, mark booked slot as unavailable
10. Return success response

**Response (200):**
```json
{
  "status": "success"
}
```

**Errors:**
- 400: Invalid signature
- 400: Missing data in webhook
- 404: User not found
- 500: Session creation failed

**Security:**
- Signature must match (prevents spoofing)
- Idempotency: Check if paymentId already processed (lastPaymentId)

---

### 2.5 Stream Endpoints

#### GET /api/stream/token
**Protection:** @protect

**Purpose:** Generate Stream Chat token

**Backend:**
1. Fetch User (id, name, picture)
2. Create StreamChat server client
3. Upsert user on Stream servers
4. Generate token (JWT signed with Stream secret)

**Response (200):**
```json
{
  "token": "eyJhbGci...",
  "apiKey": "abc123...",
  "userId": "65a1b2c3..."
}
```

**Errors:**
- 401: Not authorized
- 404: User not found
- 500: Stream API error

---

#### GET /api/stream/video-token
**Protection:** @protect

**Purpose:** Generate Stream Video token

**Response (200):**
```json
{
  "token": "eyJhbGci...",
  "apiKey": "abc123...",
  "userId": "65a1b2c3..."
}
```

---

## 3. Frontend State Management

### 3.1 AuthContext

**State:**
```javascript
{
  user: { id, name, email, role, picture, ... },
  loading: boolean,
  isAuthenticated: boolean
}
```

**Methods:**
- `login(email, password)` → POST /auth/login
- `signup(name, email, password)` → POST /auth/register
- `googleSignIn()` → Firebase popup → POST /auth/firebase-google
- `logout()` → POST /auth/logout
- `fetchUser()` → GET /auth/profile
- `updateRole(role)` → PATCH /auth/profile/role
- `updateGeneralProfile(data)` → PATCH /auth/profile
- `updateInterestedSkills(skills)` → PATCH /auth/profile/interested-skills
- `updateTeachingSkills(skills)` → PATCH /auth/profile/teaching-skills
- `updateAvailability(data)` → PATCH /auth/profile/availability

**Interceptors:** Auto-refresh tokens on 401

---

### 3.2 SessionContext

**State:**
```javascript
{
  studentUpcomingSessions: [],
  studentPastSessions: [],
  teacherUpcomingSessions: [],
  teacherPastSessions: [],
  loadingSessions: boolean,
  sessionError: string
}
```

**Methods:**
- `fetchStudentSessions()` → GET /api/student/sessions
- `fetchTeacherSessions()` → GET /api/teacher/sessions
- `handlePaymentSuccess(teacherData)` → POST /api/create-payment-order → Razorpay modal

**Auto-refresh:** Every 60s

---

### 3.3 TeacherProfileContext

**State:**
```javascript
{
  teacherProfile: { _id, userId, name, skills, ... },
  isLoading: boolean,
  isSaving: boolean,
  avatarFile: File,
  videoFile: File,
  galleryFiles: [File]
}
```

**Methods:**
- `createTeacherProfile(data)` → POST /api/teacher-profiles (multipart)
- `updateTeacherProfile(data)` → PUT /api/teacher-profiles/:id (multipart)

---

### 3.4 StreamChatContext

**State:**
```javascript
{
  chatClient: StreamChatClient,
  isClientReady: boolean
}
```

**Lifecycle:**
1. Fetch Stream token from backend
2. Initialize StreamChat instance
3. Connect user
4. Cleanup on unmount

---

### 3.5 StreamVideoContext

**State:**
```javascript
{
  videoClient: StreamVideoClient,
  isClientReady: boolean
}
```

**Lifecycle:** (Similar to StreamChatContext)

---

## 4. Key Algorithms & Logic

### 4.1 Session Categorization (Upcoming vs Past)

```javascript
const sessionEndDateTime = new Date(
  `${session.dateTime.split('T')[0]}T${session.endTime}:00`
);
const now = new Date();

if (sessionEndDateTime > now) {
  upcoming.push(session);
} else {
  past.push(session);
}
```

**Why:** DateTime stored as date only, time stored separately (HH:MM string)

---

### 4.2 Profile Completion Check

**Student:**
- Complete if: bio.length > 0 && phoneNumber.length > 0 && interestedSkills.length > 0

**Teacher:**
- Complete if: name && email && hourlyRate > 0 && experience && skills.length > 0
- Set by pre-save hook

---

### 4.3 Availability Slot Marking (Post-Payment)

```javascript
const bookedDateString = new Date(teacherData.dateTime).toISOString().split('T')[0];
const availabilityEntry = teacherUser.availability.find(a => 
  a.date.toISOString().split('T')[0] === bookedDateString
);

if (availabilityEntry) {
  const slotToUpdate = availabilityEntry.slots.find(slot =>
    slot.startTime === bookedStartTime &&
    slot.endTime === bookedEndTime &&
    slot.available === true
  );
  if (slotToUpdate) {
    slotToUpdate.available = false;
  }
}
await teacherUser.save();
```

---

### 4.4 Token Refresh Queue (Axios Interceptor)

```javascript
let isRefreshing = false;
let failedQueue = [];

const processQueue = (error, token = null) => {
  failedQueue.forEach(prom => {
    if (error) prom.reject(error);
    else prom.resolve(token);
  });
  failedQueue = [];
};

// On 401 error:
if (!isRefreshing) {
  isRefreshing = true;
  try {
    await axios.post('/auth/refresh-token');
    isRefreshing = false;
    processQueue(null, token);
    return api(originalRequest); // Retry
  } catch (err) {
    isRefreshing = false;
    processQueue(err);
    // Redirect to login
  }
} else {
  // Queue request, wait for refresh
  return new Promise((resolve, reject) => {
    failedQueue.push({ resolve, reject });
  });
}
```

**Why:** Prevent multiple simultaneous refresh requests

---

## 5. Error Codes & Messages

| Code | Error | Message | Handling |
|------|-------|---------|----------|
| 400 | BAD_REQUEST | Validation failed, missing fields | Show field errors |
| 400 | USER_EXISTS | User already registered | Redirect to login |
| 400 | INVALID_CREDENTIALS | Email or password incorrect | Show error toast |
| 401 | UNAUTHORIZED | No token or token expired | Refresh → retry / redirect login |
| 403 | FORBIDDEN | User role not authorized for action | Redirect to /not-authorized |
| 404 | NOT_FOUND | Resource doesn't exist | Show 404 page |
| 409 | CONFLICT | Duplicate email or profile | Show error toast |
| 500 | SERVER_ERROR | Internal server error | Show generic error, log to Sentry |

---

## 6. Testing Strategy

### 6.1 Unit Tests (Backend)
- Auth controller (register, login, logout)
- Validators (email, password, phone)
- Payment webhook signature verification

### 6.2 Integration Tests
- API endpoints (auth, sessions, payments)
- Database operations
- Cloudinary upload/delete

### 6.3 E2E Tests (Frontend)
- Signup flow
- Teacher profile creation
- Session booking
- Video call join

### 6.4 Manual Testing
- Cross-browser compatibility
- Mobile responsiveness
- Payment flow (test mode)
- Webhook delivery

---

## 7. Performance Optimization

**Database:**
- Indexes on frequently queried fields
- Lean queries (select needed fields only)
- Pagination (future)

**Frontend:**
- Code splitting (Vite dynamic imports)
- Image lazy loading
- Memoization (useMemo, useCallback)
- React.memo for cards

**API:**
- Gzip compression
- Cache headers (Cache-Control, ETag)
- CDN for static assets (Netlify)
- Response compression

---

## 8. Deployment Checklist

- [x] Environment variables configured (DATABASE_URL, JWT_SECRET, etc.)
- [x] Database indexes created
- [x] Cloudinary account setup
- [x] Firebase project created
- [x] Razorpay merchant account (test mode)
- [x] Stream.io API keys configured
- [x] Backend deployed (Render)
- [x] Frontend deployed (Netlify)
- [x] SSL/HTTPS enforced
- [x] CORS whitelist configured
- [x] Error logging setup (future: Sentry)
- [x] Health check endpoint working

---

**End of LLD**
