# Golden Light School — Web Application

A complete web presence + management system for **Golden Light School**, a Nursery school (3-year program). The project has three user-facing surfaces built on one shared backend:

1. **Public Website** — multi-page marketing/information site (Home, About, Admissions, Academics, News, Contact, Apply Now, Fee Info).
2. **Admin Dashboard** — internal tool for Super Admin / Admin / Accountant staff to manage applications, students, classes, fees, payments, SMS communication, and website content (news/posts).
3. **Parent Portal** — passwordless (OTP-based) portal for parents to view their child's info, check fee balance, upload proof of payment, and receive school communication.

### Documentation Index

| File                                                    | Contents                                                                                                                                        |
| ------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| [`project-structure.md`](./project-structure.md)       | Repo layout, system architecture diagram, functional & non-functional requirements, deployment plan                                             |
| [`frontend.md`](./frontend.md)                         | React app structure, all public pages, dashboard sidebar & pages, parent portal pages, all forms, folder tree, design references                |
| [`backend.md`](./backend.md)                           | Express + TypeScript + Prisma module structure, API endpoints, auth flows (staff JWT + parent OTP), business-process flowcharts, security notes |
| [`database.md`](./database.md)                         | Entity-relationship diagram, table-by-table field list, draft Prisma schema                                                                     |
| [`landing-page-content.md`](./landing-page-content.md) | Actual written copy for every public page, tailored to Golden Light School as a Nursery school                                                  |

## Tech Stack

| Layer        | Choice                                                                                       |
| ------------ | -------------------------------------------------------------------------------------------- |
| Frontend     | React (Vite) + React Router + TailwindCSS + React Hook Form + Zod + TanStack Query           |
| Backend      | Express.js + TypeScript + Prisma ORM                                                         |
| Database     | MySQL                                                                                        |
| Auth         | JWT (staff) + Phone/Email OTP (parents)                                                      |
| SMS          | Africa's Talking / Twilio (pluggable SMS provider)                                           |
| File uploads | Multer → local disk (or S3/Cloudinary later)                                                |
| Deployment   | VPS, PM2 (backend process manager), Nginx (reverse proxy + static hosting + SSL via Certbot) |

## Core Roles

- **Super Admin** — full access, manages staff accounts, school settings, fee structure.
- **Admin** — day-to-day operations: applications, students, classes, SMS, posts.
- **Accountant** *(optional role)* — fee/payment management only.
- **Parent** — OTP login, view-only + proof-of-payment upload, no self-registration (accounts are created automatically when an application is approved).

## Key Feature Summary

- Multi-page public website with dynamically-updatable fee figures (pulled from the same fee data the dashboard edits).
- Online application form → Admin review → Approve/Reject → auto-creates Student + Parent records on approval.
- Student & class management (3 nursery year-groups).
- Fee structure management (registration fee, tuition, etc.) that reflects instantly on the public Fee page.
- Payment recording by staff, and proof-of-payment upload by parents with an admin verification step.
- SMS communication: single parent, whole class, or all parents — with delivery logs.
- News/Newsletter posts manageable from the dashboard, shown publicly.
- Parent Portal with phone-number + OTP login (no passwords, no self-signup).

## Getting Started (high level)

```bash
# Backend
cd backend
cp .env.example .env        # fill DB, JWT, SMS, mail credentials
npm install
npx prisma migrate dev
npm run dev

# Frontend
cd frontend
cp .env.example .env        # set VITE_API_URL
npm install
npm run dev
```

Full setup, environment variables, and deployment steps are detailed in `backend.md` and `project-structure.md`.

# school_management_backend
