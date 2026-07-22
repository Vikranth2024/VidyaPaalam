# Product Requirements Document (PRD)
## VidyaPaalam: Teach & Learn Platform

**Version:** 1.0  
**Last Updated:** July 22, 2026  
**Status:** Live (Deployed)

---

## 1. Executive Summary

VidyaPaalam connects educators with learners for real-time skill-sharing via 1-on-1 sessions. Platform enables teachers to showcase expertise, students to discover mentors, book sessions with secure payment, real-time communication, and video calls.

---

## 2. Product Overview

**Vision:** Democratize skill learning by connecting passionate teachers with eager learners globally.

**Mission:** Provide seamless, affordable, secure platform for skill acquisition through verified mentorship.

**Target Market:**
- Students: Professionals seeking new skills, students pursuing academic help, hobbyists
- Teachers: Experts, tutors, freelancers monetizing knowledge
- Geographies: India (primary), Global (future)

---

## 3. User Personas & User Stories

### 3.1 Persona 1: Student (Learner)
**Name:** Priya, 25, Working Professional  
**Goals:** Learn new skills, schedule flexible sessions, verify teacher quality  
**Pain Points:** Expensive tutoring, inconsistent teacher quality, rigid schedules

**User Stories:**
- US-01: Student signs up via email/Google, selects "learner" role
- US-02: Student provides bio, interested skills (multi-select)
- US-03: Student searches teachers by skill, rating, price, availability
- US-04: Student views teacher profile (bio, qualifications, hourly rate, gallery, ratings)
- US-05: Student selects available date/time slot from teacher calendar
- US-06: Student proceeds to secure payment (Razorpay)
- US-07: Student joins video call at session time
- US-08: Student messaging teacher before/after session
- US-09: Student views past sessions, booked upcoming sessions
- US-10: Student adds teachers to favorites for quick access

### 3.2 Persona 2: Teacher
**Name:** Rajesh, 35, Yoga Instructor  
**Goals:** Monetize expertise, manage schedule, build reputation  
**Pain Points:** Finding students, managing bookings, payment delays, inconsistent income

**User Stories:**
- US-11: Teacher signs up via email/Google, selects "teacher" role
- US-12: Teacher completes profile (name, qualifications, skills, experience, hourly rate, media)
- US-13: Teacher uploads avatar, introduction video, gallery photos to Cloudinary
- US-14: Teacher sets weekly availability (time slots per date)
- US-15: Teacher views booked sessions calendar, student details
- US-16: Teacher starts video call at session time
- US-17: Teacher messages students, receives payment notifications
- US-18: Teacher views ratings, past sessions, earnings summary
- US-19: Teacher updates profile details, availability anytime

### 3.3 Persona 3: Admin
**Name:** Admin User  
**Goals:** Monitor platform health, moderate content, handle disputes

**User Stories:**
- US-20: Admin dashboard with user counts, session stats, revenue
- US-21: Admin manages user accounts, teacher profile verification
- US-22: Admin resolves payment disputes

---

## 4. Features

### 4.1 Authentication & Authorization
- Email/password signup with validation (name, email, password strength)
- Google OAuth via Firebase
- JWT tokens (access: 15min, refresh: 7days)
- Token blacklisting on logout
- Role-based access (student/teacher/admin)

### 4.2 User Profiles
- **User Model:** Name, email, phone, bio, picture (Firebase/Cloudinary), role, skills, availability
- **Teacher Profile:** Dedicated profile (userId ref), qualifications, title, experience, hourly rate, avatar, intro video, gallery photos
- **Profile Completion:** Tracked via flags (teacherOnboardingComplete, isProfileComplete)

### 4.3 Onboarding Flows
- **Student:** Role selection → Basic info (name, phone) → Interested skills (multi-select) → Complete
- **Teacher:** Role selection → Basic info → Teaching skills → Availability setup → Profile completion (optional: photo/video upload)

### 4.4 Teacher Discovery & Search
- Browse all teachers (grid/list view)
- Filter by: skill, rating, hourly rate, availability
- Search by name/skill keyword
- Teacher profile page: bio, qualifications, hourly rate, ratings, availability calendar, gallery

### 4.5 Session Booking
- Calendar-based slot selection (date disabled if no available slots)
- Time slot picker (sorted, shows availability status)
- Review & pay flow (session details, fee + platform fee)
- Razorpay payment gateway integration
- Session created on payment webhook (paymentAcknowledged flag)

### 4.6 Real-Time Communication
- Stream Chat integration (messaging before/after sessions)
- Direct DM between student-teacher
- User profile fetch in chat header

### 4.7 Video Calls
- Stream Video SDK integration
- Unique callId = sorted(userId1, userId2)
- Speaker layout (desktop), grid layout (mobile)
- Call controls (mute, screen share, end call)
- Participant list sidebar
- Join during session window

### 4.8 Session Management
- **Student View:** Upcoming sessions (chronological), past sessions, session cards with join/message/details
- **Teacher View:** Upcoming sessions (student names, times), past sessions, session stats
- Automatic timezone-aware date/time display
- Session-end-time calculation to separate past/upcoming

### 4.9 Availability Management (Teacher)
- Calendar UI to select dates
- Add time slots per date (start/end time)
- Mark slots as available/booked
- Remove slots
- Save to User.availability array (MongoDB)
- Availability updates after payment (slot marked unavailable)

### 4.10 Payment & Billing
- Razorpay payment orders created with teacher data in notes
- Webhook verification (SHA256 signature)
- Payment events processed (payment.captured, order.paid)
- Session auto-created on successful webhook
- Availability auto-updated (slot marked unavailable)
- User flag paymentAcknowledged tracks first payment

### 4.11 Ratings & Reviews (Planned - MVP v1.1)
- 1-5 star rating system
- Review text submission
- Display on teacher profile (avg rating, individual reviews)
- Mock ratings currently shown (TeacherRatings component)

### 4.12 Favorites (Planned - MVP v1.1)
- Add/remove teacher from favorites
- Favorites list view (Student)
- Quick access to top teachers

---

## 5. Functional Requirements

### 5.1 Auth API (POST /auth/*)
- POST /auth/register → Create user (email unique)
- POST /auth/login → Issue JWT tokens
- POST /auth/firebase-google → Link/create user from Firebase
- GET /auth/profile → Fetch current user
- POST /auth/logout → Blacklist refresh token
- POST /auth/refresh-token → Issue new access token
- PATCH /auth/profile → Update basic info
- PATCH /auth/profile/role → Set role (student/teacher)
- PATCH /auth/profile/interested-skills → Update student skills
- PATCH /auth/profile/teaching-skills → Update teacher skills
- PATCH /auth/profile/availability → Update teacher availability

### 5.2 Teacher Profile API (GET/POST/PUT /api/teacher-profiles/*)
- GET /api/teacher-profiles → List all (public)
- GET /api/teacher-profiles/:id → Fetch single (public)
- GET /api/teacher-profiles/user/:userId → By userId
- GET /api/teacher-profiles/me → Authenticated user's profile (teacher only)
- POST /api/teacher-profiles → Create (multipart, teacher only)
- PUT /api/teacher-profiles/:id → Update (multipart, owner/admin)
- DELETE /api/teacher-profiles/:id → Delete (owner/admin)
- GET /api/teacher-profiles/:id/availability → Fetch availability
- GET /api/teacher-profiles/teacher-profiles-for-booking/:id → For booking flow

### 5.3 Session API (GET/POST /api/*sessions)
- GET /api/student/sessions → Fetch student's sessions (paginated, upcoming+past)
- POST /api/student/sessions → Create session (protection: payment webhook only)
- GET /api/teacher/sessions → Fetch teacher's sessions (protected)

### 5.4 Payment API (POST /api/*)
- POST /api/create-payment-order → Create Razorpay order
- POST /api/razorpay-webhook → Webhook handler (raw body, signature verify)

### 5.5 Stream Chat API (GET /api/stream/*)
- GET /api/stream/token → Generate chat token (protected)

### 5.6 Stream Video API (GET /api/stream/*)
- GET /api/stream/video-token → Generate video token (protected)

---

## 6. Non-Functional Requirements

### 6.1 Performance
- Page load < 3s (optimized images, lazy loading)
- API response < 500ms
- Database indexing on frequently queried fields (userId, email, availability.date)

### 6.2 Security
- HTTPS enforced
- JWT token secure storage (httpOnly cookies)
- CORS whitelist (frontend domain only)
- Password strength validation (8+ chars, uppercase, lowercase, digit, special char)
- Rate limiting on login (5 attempts/15min)
- Input validation (express-validator)
- Signature verification on webhooks

### 6.3 Scalability
- Stateless API (no server sessions)
- Database indexes for queries
- Cloudinary CDN for media
- Horizontal scaling via Render/Heroku

### 6.4 Availability
- 99.5% uptime target
- Graceful error handling
- User-friendly error messages

### 6.5 Accessibility
- Mobile-responsive design
- Keyboard navigation
- ARIA labels (planned v1.1)

---

## 7. Data Model (Summary)

| Model | Key Fields | Relations |
|-------|-----------|-----------|
| User | id, name, email, password, role, picture, availability[], interestedSkills[], teachingSkills[], activeToken | 1:1 TeacherProfile |
| TeacherProfile | id, userId, name, email, avatar, skills[], experience, hourlyRate, qualifications[], videoUrl, galleryPhotos[], isProfileComplete | 1:1 User |
| Session | id, teacherId, studentId, teacherName, skill, dateTime, startTime, endTime, paymentId | ref User |
| BlacklistedToken | id, token, expiresAt | TTL index |

---

## 8. User Flows

### 8.1 Student Discovery → Booking → Payment
```
1. Student logs in / signs up (Google/email)
2. Role selection → "I want to learn"
3. Complete onboarding (name, phone, interested skills)
4. Dashboard shows upcoming sessions / empty
5. Click "Find Teachers" → Search/filter page
6. Select teacher → View profile + availability
7. Click "Book Session" → Calendar picks date/slot
8. Review & pay (Razorpay modal)
9. Payment success → Session created → Dashboard updated
10. Join call at session time → Video call
11. Post-session → Chat or rate (v1.1)
```

### 8.2 Teacher Setup → Availability → Sessions
```
1. Teacher logs in / signs up (Google/email)
2. Role selection → "I want to teach"
3. Complete onboarding (name, phone, teaching skills)
4. Profile setup (avatar, bio, qualifications, hourly rate, video/gallery)
5. Set availability (calendar + time slots)
6. Dashboard shows upcoming sessions (when students book)
7. Join call at session time
8. View ratings/reviews (when students rate)
```

---

## 9. Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| User Acquisition | 1000 users/month | Analytics |
| Teacher Onboarding Completion | 80%+ | Onboarding flag check |
| Session Completion Rate | 90%+ | Session timestamp comparison |
| Payment Success Rate | 95%+ | Razorpay webhook logs |
| Average Session Rating | 4.5+ | Reviews collection (v1.1) |
| Platform Uptime | 99.5% | Monitoring service |

---

## 10. Future Roadmap (Post-MVP)

### Phase 2 (v1.1)
- Ratings & reviews system
- Favorites functionality
- Admin dashboard
- Email notifications

### Phase 3 (v1.2)
- Group sessions
- Recorded sessions (replay)
- Certificate generation
- In-app wallet/credits

### Phase 4 (v2.0)
- Mobile app (iOS/Android)
- AI-powered teacher recommendations
- Multi-language support
- Corporate training partnerships

---

## 11. Constraints & Assumptions

**Constraints:**
- Initial market: India (timezone IST)
- Payment currency: INR only (Razorpay)
- Max file size: 100MB (Cloudinary)
- Max users per free tier: 1000

**Assumptions:**
- Users have stable internet (for video calls)
- Teachers provide accurate availability
- Students book sessions in advance (min 24hrs before)
- Payment processing successful (webhook triggered)

---

## 12. Acceptance Criteria

✅ User can sign up, select role, complete onboarding  
✅ Teacher can create profile, set availability  
✅ Student can search, filter teachers, view profiles  
✅ Student can book session via calendar + payment  
✅ Payment webhook creates session + updates availability  
✅ Student & teacher can join video call  
✅ Real-time messaging functional before/after session  
✅ All pages responsive (mobile, tablet, desktop)  
✅ Deployed to production (Netlify + Render)  
✅ No console errors in production  

---

## 13. Glossary

- **Slot:** Time interval (start/end time) on availability date
- **Session:** Booked meeting between student & teacher
- **Onboarding:** Multi-step setup process post-signup
- **Payment Acknowledgment:** Server confirms payment success
- **Webhook:** Server-to-server POST from Razorpay on payment event
- **Stream:** Third-party SDK (Chat/Video) by GetStream.io
- **Cloudinary:** CDN for file uploads (avatars, videos, photos)

---

**End of PRD**
