# Database Structure — Golden Light School

MySQL, accessed through Prisma. Single-tenant (one school), so there is no `School`/tenant table — a `Settings` table holds the one row of school-wide info instead.

## 1. Entity-Relationship Diagram

```mermaid
erDiagram
    STAFF ||--o{ SMS_LOG : sends
    STAFF ||--o{ POST : authors
    STAFF ||--o{ PAYMENT : records
    STAFF ||--o{ APPLICATION : reviews

    PARENT ||--o{ STUDENT : "has children"
    PARENT ||--o{ PAYMENT_PROOF : uploads
    PARENT ||--o{ SMS_LOG : receives
    PARENT ||--o{ OTP : requests

    CLASS ||--o{ STUDENT : contains
    CLASS ||--o{ FEE_ITEM : "scoped to (optional)"

    APPLICATION }o--|| CLASS : "requested class"

    STUDENT ||--o{ PAYMENT : "billed for"
    STUDENT ||--o{ PAYMENT_PROOF : "proof for"

    FEE_ITEM ||--o{ PAYMENT : "paid against"

    POST {
        int id
        string title
        string slug
        text body
        string coverImageUrl
        enum status
        datetime publishDate
    }

    STAFF {
        int id
        string name
        string email
        string passwordHash
        enum role
        boolean active
    }

    PARENT {
        int id
        string name
        string phone
        string email
        string address
    }

    STUDENT {
        int id
        string name
        date dob
        enum gender
        string photoUrl
        int classId
        int parentId
        enum status
    }

    CLASS {
        int id
        string name
        int capacity
        string academicYear
    }

    APPLICATION {
        int id
        string childName
        date childDob
        int requestedClassId
        string parentName
        string parentPhone
        string parentEmail
        string parentAddress
        enum status
        text reviewNote
        int reviewedByStaffId
    }

    FEE_ITEM {
        int id
        string name
        decimal amount
        int classId
        boolean active
    }

    PAYMENT {
        int id
        int studentId
        int feeItemId
        decimal amount
        date paidOn
        string method
        int recordedByStaffId
        string note
    }

    PAYMENT_PROOF {
        int id
        int studentId
        int parentId
        decimal claimedAmount
        string fileUrl
        enum status
        string reviewNote
        int reviewedByStaffId
    }

    SMS_LOG {
        int id
        int sentByStaffId
        int parentId
        string message
        enum status
        datetime sentAt
    }

    OTP {
        int id
        string phoneOrEmail
        int parentId
        string codeHash
        datetime expiresAt
        boolean used
    }
```

## 2. Table-by-Table Notes

### `Staff`

- `role`: enum `SUPER_ADMIN | ADMIN | ACCOUNTANT`
- `email` unique, `passwordHash` via bcrypt
- `active` flag instead of hard delete (deactivated staff can't log in)

### `Parent`

- `phone` unique (this is the OTP login identifier)
- `email` optional, unique if present
- One parent can have many `Student` children

### `Class`

- Represents the school's nursery year-groups (e.g. "Baby Class", "Middle Class", "Top Class") — kept as data, not an enum, so names/capacity can change without a code deploy
- `academicYear` allows re-using class names across years if needed later

### `Student`

- `status`: enum `ACTIVE | GRADUATED | WITHDRAWN`
- Belongs to one `Class` and one `Parent`
- `parentId` nullable only transiently during data migration; normally required

### `Application`

- Public-facing submission; not yet a `Student`/`Parent` until approved
- `status`: enum `PENDING | APPROVED | REJECTED`
- On approval, the service layer creates/links `Parent` (by phone) and creates `Student`, then stamps `reviewedByStaffId`

### `FeeItem`

- `classId` nullable → null means "applies to all classes"
- `active` toggle lets Admin hide a fee from the public Fee Info page without deleting history

### `Payment`

- Always linked to a `Student` and a `FeeItem`
- `recordedByStaffId` — every payment is attributable to a staff member (direct entry) or to the staff member who confirmed a proof

### `PaymentProof`

- Parent-submitted, pending Admin/Accountant action
- `status`: enum `PENDING | CONFIRMED | REJECTED`
- On `CONFIRMED`, the service layer creates a corresponding `Payment` row

### `SmsLog`

- One row per recipient per send (a "class" or "all" send fans out into many rows)
- `status`: enum `SENT | FAILED`

### `Otp`

- Short-lived, `expiresAt` + `used` flag; a background job (or query filter) purges expired rows
- `codeHash` — the raw code is never stored, only its hash, mirroring the staff password approach

### `Post`

- `status`: enum `DRAFT | PUBLISHED`
- `slug` unique, used in the public `/news/:slug` route

### `Settings` (not shown in the ERD above — single row)

- School name, address, phone, email, social links, current intake dates

## 3. Draft Prisma Schema

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "mysql"
  url      = env("DATABASE_URL")
}

enum StaffRole {
  SUPER_ADMIN
  ADMIN
  ACCOUNTANT
}

enum ApplicationStatus {
  PENDING
  APPROVED
  REJECTED
}

enum StudentStatus {
  ACTIVE
  GRADUATED
  WITHDRAWN
}

enum PaymentProofStatus {
  PENDING
  CONFIRMED
  REJECTED
}

enum SmsStatus {
  SENT
  FAILED
}

enum PostStatus {
  DRAFT
  PUBLISHED
}

model Staff {
  id            Int       @id @default(autoincrement())
  name          String
  email         String    @unique
  passwordHash  String
  role          StaffRole
  active        Boolean   @default(true)
  createdAt     DateTime  @default(now())

  applicationsReviewed Application[]  @relation("ReviewedBy")
  paymentsRecorded     Payment[]
  proofsReviewed       PaymentProof[]
  smsSent              SmsLog[]
  posts                Post[]
}

model Parent {
  id        Int      @id @default(autoincrement())
  name      String
  phone     String   @unique
  email     String?  @unique
  address   String
  createdAt DateTime @default(now())

  students        Student[]
  paymentProofs   PaymentProof[]
  smsReceived     SmsLog[]
  otps            Otp[]
}

model Class {
  id           Int      @id @default(autoincrement())
  name         String
  capacity     Int?
  academicYear String

  students     Student[]
  feeItems     FeeItem[]
  applications Application[]
}

model Student {
  id        Int           @id @default(autoincrement())
  name      String
  dob       DateTime
  gender    String
  photoUrl  String?
  status    StudentStatus @default(ACTIVE)

  classId   Int
  class     Class   @relation(fields: [classId], references: [id])
  parentId  Int
  parent    Parent  @relation(fields: [parentId], references: [id])

  payments      Payment[]
  paymentProofs PaymentProof[]
  createdAt     DateTime @default(now())
}

model Application {
  id                Int                @id @default(autoincrement())
  childName         String
  childDob          DateTime
  requestedClassId  Int
  requestedClass    Class              @relation(fields: [requestedClassId], references: [id])
  parentName        String
  parentPhone       String
  parentEmail       String?
  parentAddress     String
  status            ApplicationStatus  @default(PENDING)
  reviewNote         String?
  reviewedByStaffId Int?
  reviewedBy        Staff?             @relation("ReviewedBy", fields: [reviewedByStaffId], references: [id])
  createdAt         DateTime           @default(now())
}

model FeeItem {
  id       Int      @id @default(autoincrement())
  name     String
  amount   Decimal  @db.Decimal(10, 2)
  classId  Int?
  class    Class?   @relation(fields: [classId], references: [id])
  active   Boolean  @default(true)

  payments Payment[]
}

model Payment {
  id                 Int      @id @default(autoincrement())
  studentId          Int
  student            Student  @relation(fields: [studentId], references: [id])
  feeItemId          Int
  feeItem            FeeItem  @relation(fields: [feeItemId], references: [id])
  amount             Decimal  @db.Decimal(10, 2)
  paidOn             DateTime
  method             String
  note               String?
  recordedByStaffId  Int
  recordedBy         Staff    @relation(fields: [recordedByStaffId], references: [id])
  createdAt          DateTime @default(now())
}

model PaymentProof {
  id                Int                 @id @default(autoincrement())
  studentId         Int
  student           Student             @relation(fields: [studentId], references: [id])
  parentId          Int
  parent            Parent              @relation(fields: [parentId], references: [id])
  claimedAmount     Decimal             @db.Decimal(10, 2)
  fileUrl           String
  status            PaymentProofStatus  @default(PENDING)
  reviewNote        String?
  reviewedByStaffId Int?
  reviewedBy        Staff?              @relation(fields: [reviewedByStaffId], references: [id])
  createdAt         DateTime            @default(now())
}

model SmsLog {
  id             Int       @id @default(autoincrement())
  sentByStaffId  Int
  sentBy         Staff     @relation(fields: [sentByStaffId], references: [id])
  parentId       Int
  parent         Parent    @relation(fields: [parentId], references: [id])
  message        String    @db.Text
  status         SmsStatus
  sentAt         DateTime  @default(now())
}

model Otp {
  id            Int      @id @default(autoincrement())
  phoneOrEmail  String
  parentId      Int?
  parent        Parent?  @relation(fields: [parentId], references: [id])
  codeHash      String
  expiresAt     DateTime
  used          Boolean  @default(false)
  createdAt     DateTime @default(now())
}

model Post {
  id            Int        @id @default(autoincrement())
  title         String
  slug          String     @unique
  body          String     @db.Text
  coverImageUrl String?
  status        PostStatus @default(DRAFT)
  publishDate   DateTime?
  authorStaffId Int
  author        Staff      @relation(fields: [authorStaffId], references: [id])
  createdAt     DateTime   @default(now())
}

model Settings {
  id            Int      @id @default(1)
  schoolName    String
  address       String
  phone         String
  email         String
  facebookUrl   String?
  instagramUrl  String?
  nextIntakeDate DateTime?
}
```

## 4. Indexing & Constraints Summary

- `Parent.phone` — unique, used as the OTP login key.
- `Staff.email` — unique.
- `Post.slug` — unique, indexed for public lookups.
- Foreign keys on `Student.classId`, `Student.parentId`, `Payment.studentId`, `Payment.feeItemId` should all be indexed (Prisma/MySQL do this automatically for FKs) since these back the most frequent dashboard queries (student list by class, payment history by student).
- Consider a composite index on `SmsLog(parentId, sentAt)` for fast "notices for this parent" queries in the Parent Portal.
