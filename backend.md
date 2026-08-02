# Backend Structure — Golden Light School

## 1. Tech Stack

- **Express.js** + **TypeScript**
- **Prisma ORM** → **MySQL**
- **JWT** (`jsonwebtoken`) for staff auth; short-lived OTP + session JWT for parents
- **bcrypt** for staff password hashing
- **Zod** for request validation
- **Multer** for file uploads (proof of payment, post cover images)
- SMS provider SDK (Africa's Talking / Twilio) behind a small internal interface
- **Nodemailer** for optional email OTP / notification fallback
- **node-cron** (optional) for scheduled tasks — e.g. clearing expired OTPs

## 2. Folder Structure (feature-based modules)

```
backend/
├── src/
│   ├── config/
│   │   ├── env.ts
│   │   ├── prisma.ts
│   │   └── sms.ts / mailer.ts
│   ├── middlewares/
│   │   ├── authenticateStaff.ts       # verifies staff JWT
│   │   ├── authenticateParent.ts      # verifies parent session JWT
│   │   ├── requireRole.ts             # role-based access control
│   │   ├── validateRequest.ts         # zod middleware
│   │   ├── rateLimiter.ts             # OTP request throttling
│   │   ├── errorHandler.ts
│   │   └── upload.ts                  # multer config
│   ├── modules/
│   │   ├── auth/            (staff login, parent request-otp, parent verify-otp, refresh, logout)
│   │   ├── applications/    (submit - public, list/detail/approve/reject - admin)
│   │   ├── students/        (CRUD, search/filter, class history)
│   │   ├── classes/         (CRUD)
│   │   ├── parents/         (CRUD, link to students)
│   │   ├── fees/            (fee structure CRUD - feeds public site)
│   │   ├── payments/        (record payment, list/history, proof upload, proof review)
│   │   ├── sms/             (send single/class/all, logs, templates)
│   │   ├── posts/           (public list/detail, admin CRUD)
│   │   ├── settings/        (school info, roles/permissions)
│   │   └── staff/           (Super Admin: manage staff accounts)
│   │       # each module folder contains: routes.ts, controller.ts, service.ts, validator.ts
│   ├── routes/
│   │   └── index.ts          # mounts all module routers under /api
│   ├── utils/
│   │   ├── generateOtp.ts, tokens.ts, pagination.ts, apiResponse.ts
│   ├── jobs/
│   │   └── cleanExpiredOtps.ts
│   ├── app.ts
│   └── server.ts
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── uploads/                  # served statically via Nginx in production
├── .env.example
├── tsconfig.json
└── package.json
```

## 3. API Endpoints

### Auth
| Method | Path | Access | Description |
|---|---|---|---|
| POST | `/api/auth/staff/login` | Public | Email + password → JWT |
| POST | `/api/auth/parent/request-otp` | Public | Phone/email → generates & sends OTP |
| POST | `/api/auth/parent/verify-otp` | Public | Phone/email + code → session JWT |
| POST | `/api/auth/refresh` | Staff/Parent | Refresh access token |
| POST | `/api/auth/logout` | Staff/Parent | Invalidate session |

### Applications
| Method | Path | Access | Description |
|---|---|---|---|
| POST | `/api/applications` | Public | Submit application (child + parent info) |
| GET | `/api/applications` | Admin | Paginated list, filter by status |
| GET | `/api/applications/:id` | Admin | Detail |
| PATCH | `/api/applications/:id/approve` | Admin | Approve → creates Student + Parent, sends SMS |
| PATCH | `/api/applications/:id/reject` | Admin | Reject → sends SMS/email with reason |

### Students / Classes / Parents
| Method | Path | Access | Description |
|---|---|---|---|
| GET/POST | `/api/students` | Admin | List (paginated/search) / Create |
| GET/PATCH/DELETE | `/api/students/:id` | Admin | Detail / Update / Soft-delete |
| GET/POST | `/api/classes` | Admin | List / Create |
| PATCH/DELETE | `/api/classes/:id` | Admin | Update / Delete |
| GET/POST | `/api/parents` | Admin | List / Create |
| GET/PATCH | `/api/parents/:id` | Admin/Parent (own record) | Detail / Update contact info |

### Fees & Payments
| Method | Path | Access | Description |
|---|---|---|---|
| GET | `/api/fees` | Public | Current fee structure (for Fee Info page) |
| POST/PATCH/DELETE | `/api/fees/:id` | Admin | Manage fee items |
| POST | `/api/payments` | Admin/Accountant | Record a payment |
| GET | `/api/payments` | Admin/Accountant | History (filterable) |
| GET | `/api/payments/student/:studentId` | Admin/Parent (own child) | Statement |
| POST | `/api/payments/proof` | Parent | Upload proof of payment |
| GET | `/api/payments/proof` | Admin/Accountant | Review queue |
| PATCH | `/api/payments/proof/:id/confirm` | Admin/Accountant | Confirm → updates balance, notifies parent |
| PATCH | `/api/payments/proof/:id/reject` | Admin/Accountant | Reject → notifies parent |

### SMS
| Method | Path | Access | Description |
|---|---|---|---|
| POST | `/api/sms/send` | Admin | Body: `{ mode: single\|class\|all, targetId?, message }` |
| GET | `/api/sms/logs` | Admin | Paginated delivery logs |
| GET/POST | `/api/sms/templates` | Admin | Manage reusable templates |

### Posts (News & Events)
| Method | Path | Access | Description |
|---|---|---|---|
| GET | `/api/posts` | Public | Published posts, paginated |
| GET | `/api/posts/:slug` | Public | Single post |
| POST/PATCH/DELETE | `/api/posts/:id` | Admin | Manage posts |

### Settings & Staff
| Method | Path | Access | Description |
|---|---|---|---|
| GET/PATCH | `/api/settings/school` | Public read / Super Admin write | School info feeding the public site |
| GET/POST | `/api/staff` | Super Admin | List / Create staff account |
| PATCH/DELETE | `/api/staff/:id` | Super Admin | Update role / Deactivate |

## 4. Roles & Permissions Matrix

| Module | Super Admin | Admin | Accountant | Parent |
|---|---|---|---|---|
| Applications | ✅ | ✅ | ❌ | Submit only (public) |
| Students/Classes | ✅ | ✅ | View | View own child |
| Parents | ✅ | ✅ | View | Edit own profile |
| Fee Structure | ✅ | ✅ | View | View (public page) |
| Payments/Proofs | ✅ | ✅ | ✅ | Upload proof / view own |
| SMS | ✅ | ✅ | ❌ | Receive only |
| Posts | ✅ | ✅ | ❌ | View (public) |
| Staff Accounts | ✅ | ❌ | ❌ | ❌ |
| School Settings | ✅ | View | ❌ | ❌ |

## 5. Process Flows

### 5.1 Parent OTP Login

```mermaid
flowchart TD
    A[Parent enters phone/email] --> B[POST /auth/parent/request-otp]
    B --> C{Phone/email linked to a Parent record?}
    C -- No --> D[Return error: not found]
    C -- Yes --> E[Generate 6-digit OTP, store with expiry ~5 min]
    E --> F[Send OTP via SMS gateway - or email fallback]
    F --> G[Parent enters OTP code]
    G --> H[POST /auth/parent/verify-otp]
    H --> I{Code valid and not expired?}
    I -- No --> J[Return error, allow retry with rate limit]
    I -- Yes --> K[Issue session JWT, mark OTP used]
    K --> L[Redirect to Parent Portal]
```

### 5.2 Application → Approval → Student/Parent Creation

```mermaid
flowchart TD
    A[Public: Apply Now form submitted] --> B[Application saved: status PENDING]
    B --> C[Admin reviews in dashboard]
    C --> D{Decision}
    D -- Reject --> E[Status REJECTED + optional note]
    E --> F[SMS/email sent to parent with decision]
    D -- Approve --> G[Status APPROVED]
    G --> H{Parent phone already exists?}
    H -- Yes --> I[Link new Student to existing Parent]
    H -- No --> J[Create new Parent record]
    I --> K[Create Student record, assign Class]
    J --> K
    K --> L[Send welcome SMS with Parent Portal login instructions]
```

### 5.3 Fee Payment & Proof Verification

```mermaid
flowchart TD
    A[Two entry points] --> B[Admin records payment directly]
    A --> C[Parent uploads proof of payment]
    B --> D[Payment status CONFIRMED, balance updated immediately]
    C --> E[Proof status PENDING in review queue]
    E --> F{Admin/Accountant reviews}
    F -- Confirm --> D
    F -- Reject --> G[Status REJECTED, note added]
    D --> H[SMS sent to parent: payment confirmed, new balance]
    G --> I[SMS sent to parent: proof rejected, reason]
```

### 5.4 SMS Sending

```mermaid
flowchart TD
    A[Admin composes message] --> B{Recipient mode}
    B -- Single Parent --> C[Resolve one parent's phone]
    B -- By Class --> D[Resolve all parents of students in that class]
    B -- All Parents --> E[Resolve all active parents]
    C --> F[Queue SMS job per recipient]
    D --> F
    E --> F
    F --> G[Send via SMS gateway]
    G --> H{Delivery result}
    H -- Success --> I[Log status = SENT]
    H -- Failure --> J[Log status = FAILED, retry once]
```

## 6. Environment Variables

```
PORT=5000
NODE_ENV=production
DATABASE_URL=mysql://user:password@localhost:3306/golden_light_school

JWT_ACCESS_SECRET=
JWT_REFRESH_SECRET=
JWT_ACCESS_EXPIRES=15m
JWT_REFRESH_EXPIRES=7d

OTP_EXPIRY_MINUTES=5
OTP_RATE_LIMIT_PER_HOUR=5

SMS_PROVIDER=africastalking   # or twilio
SMS_API_KEY=
SMS_USERNAME=
SMS_SENDER_ID=

SMTP_HOST=
SMTP_PORT=
SMTP_USER=
SMTP_PASS=

UPLOAD_DIR=./uploads
MAX_UPLOAD_SIZE_MB=5

CORS_ORIGIN=https://goldenlightschool.example
```

## 7. Security Notes

- Staff passwords hashed with bcrypt (cost factor ≥ 10).
- OTP endpoints rate-limited per phone number and per IP to prevent SMS-bombing/abuse.
- File uploads restricted by MIME type (image/pdf) and size (`MAX_UPLOAD_SIZE_MB`).
- All mutating admin endpoints require both `authenticateStaff` and `requireRole([...])`.
- Public endpoints (`apply`, `fees`, `posts`) are read/write-limited and validated with Zod to prevent injection/spam.
- Centralized `errorHandler` avoids leaking stack traces in production responses.

## 8. Deployment Notes

- Build: `npm run build` → `dist/`.
- Run under PM2: `pm2 start dist/server.js --name golden-light-api --env production`.
- Nginx reverse-proxies `/api/*` to the PM2 process and serves `/uploads/*` as static files.
- Run `npx prisma migrate deploy` as part of the deploy step (not `migrate dev`, which is for local development only).
