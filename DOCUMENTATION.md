# D'Blossom Model Private Schools — Complete Technical Documentation

> This document is the complete, authoritative specification of the D'Blossom Model Private Schools web application. It is detailed enough that another AI or a professional software engineer can recreate this application **exactly** without ever seeing the original project. Nothing is summarized or omitted.

---

## Table of Contents

1. Project Overview
2. Complete Software Architecture
3. Database Schema
4. Authentication
5. Every Page
6. Dashboard
7. Student Portal
8. Teacher Portal
9. Parent Portal
10. Admin Portal
11. Result Management
12. Attendance System
13. Fees System
14. File Upload System
15. Notification System
16. Business Rules
17. UI Design System
18. Components
19. API Documentation
20. Folder Structure
21. Source Code Explanation
22. Deployment
23. Migration Guide (React + GitHub + Netlify + Supabase)
24. Recreation Master Prompt

---

## 1. Project Overview

### Purpose
D'Blossom Model Private Schools is a comprehensive school management web platform for a real Nigerian private school (Divine Blossom Model School, Abeokuta, Ogun State). It serves as the school's public website **and** its internal management system — handling student records, staff records, academic results, fee payment notifications, events/news, photo gallery, and parent complaints/contact messages.

### Type of Application
A single-page application (SPA) built with React + Vite, using a Backend-as-a-Service (Base44) for the database, file storage, and email. It is a responsive web app that works on mobile and desktop and is publishable to iOS/Android from the same codebase.

### Target Users
- **Public visitors / prospective parents** — browse the public website (Home, About, Academics, Gallery, Events, Payment info, Contact).
- **Students (JSS1–SS3)** — log in to the Student Portal to view their academic results and download report cards as PDF.
- **Teachers / Staff** — log in to the Teacher Portal. Class Teachers can enter and edit student results; general Teaching Staff get view-only access.
- **Administrators** — log in to the Admin Dashboard to manage students, teachers, results, payments, events, gallery, complaints, subjects, and settings.

### Functional Overview
- Public marketing website with hero, about, academics programs, gallery, events, testimonials.
- Student portal: login by admission number + password, view results by term/session, download PDF report card.
- Teacher portal: login by staff ID + password, role-based (Class Teacher = full result entry; Teaching Staff = view only), enter CA/exam scores per subject, auto-calculate totals/grades/average, add overall percentage, write teacher comment, save/update results.
- Admin dashboard: full CRUD for students, teachers, results, events, gallery images, payments, complaints; subject list management; admin password change.
- Payment notification system: parents fill a form, a Payment record is created, admin confirms/rejects, parent sends proof via WhatsApp.
- Complaint/contact system: form creates a Complaint record and sends an email to the school.
- File uploads for event images and gallery images via the platform's UploadFile integration.
- PDF report card generation with multi-page support.

---

## 2. Complete Software Architecture

### Frontend Architecture
- **Framework:** React 18 (functional components, hooks).
- **Build tool:** Vite 6 (ESM, `import.meta.env`).
- **Routing:** react-router-dom v6 (`BrowserRouter`, `Routes`, `Route`, `Link`, `Outlet`, `useNavigate`, `useLocation`).
- **Styling:** Tailwind CSS 3.4 with a custom design-token system (CSS variables in `src/index.css` mapped in `tailwind.config.js`). shadcn/ui component primitives (Radix UI based) in `src/components/ui/`.
- **Icons:** lucide-react.
- **Animations:** framer-motion (page/section entrance animations, testimonial carousel, modal transitions).
- **Data fetching / cache:** @tanstack/react-query v5 for admin lists (students, teachers, results, payments, complaints, events, gallery). Direct `base44.entities.*` SDK calls for portal logins and result saving.
- **Toasts:** sonner (`toast.success/error/info`).
- **PDF:** jspdf + html2canvas (dynamically imported on demand).
- **Date formatting:** moment.

### Backend Architecture
- **BaaS:** Base44 platform. The backend is **not** custom code — it is a managed service providing:
  - A document database (MongoDB-style) where each "entity" is a collection with a JSON schema.
  - Auto-generated REST endpoints per entity (list, filter, get, create, update, delete, bulk operations).
  - Row-Level Security (RLS) per entity per operation (create/read/update/delete).
  - Built-in integrations: `UploadFile`, `SendEmail`, `InvokeLLM`, `GenerateImage`, etc.
  - Auth (token-based) — though this app uses a **custom localStorage-based** auth for admin/teacher/student, NOT the platform auth, because the live site is public.
- The frontend talks to the backend through a pre-initialized SDK client (`@/api/base44Client` → `@base44/sdk`'s `createClient`).

### APIs
There are no hand-written REST endpoints. All data access is via the Base44 SDK entity methods:
- `base44.entities.<Entity>.list(sort, limit)`
- `base44.entities.<Entity>.filter(query, sort, limit)`
- `base44.entities.<Entity>.get(id)`
- `base44.entities.<Entity>.create(data)`
- `base44.entities.<Entity>.update(id, data)`
- `base44.entities.<Entity>.delete(id)`
- `base44.integrations.Core.UploadFile({ file })` → `{ file_url }`
- `base44.integrations.Core.SendEmail({ to, subject, body })`
- `base44.auth.me()`, `base44.auth.logout()`, `base44.auth.redirectToLogin()`

### Authentication Flow
There are **two parallel auth systems**:

1. **Platform auth (AuthContext)** — `src/lib/AuthContext.jsx` checks app public settings and the current platform user via `base44.auth.me()`. This is scaffold code from the platform. On this app, `requiresAuth: false` (public app), so unauthenticated visitors can browse. This auth is **not** used for admin/teacher/student logins.

2. **Custom app auth (localStorage + entity lookup)** — the real auth used by the three portals:
   - **Admin:** username + password checked against constants (`ADMIN_USERNAME = "DivineBlossom"`, default password `"DBMS"`). The password can be changed and is stored in `localStorage` under `admin_password`. Login state stored in `localStorage` as `admin_logged_in = "true"`.
   - **Teacher:** staff_id + password looked up in the `Teacher` entity (`status: 'Active'`), password compared as plain text. Session held in React state only (no persistence — refresh logs out).
   - **Student:** admission_number + password looked up in the `Student` entity (`status: 'Active'`), finds the matching record by password among all matches with that admission number. Session held in React state only.

### Data Flow
- Public pages fetch entities via react-query (gallery, events) or directly (none).
- Admin pages fetch via react-query with query keys like `['admin-students']`, invalidate after mutations.
- Portals fetch directly with `base44.entities.*.filter()` on login and on action.
- Mutations call `create/update/delete` then `qc.invalidateQueries` to refresh lists.
- File uploads call `UploadFile` to get a `file_url`, stored in the entity's `image_url` field.

### State Management
- No global state library (no Redux/Zustand).
- Local component state via `useState`.
- Server state via @tanstack/react-query (caching, refetch, invalidation).
- Auth context via React Context (`AuthContext`) — only for platform auth scaffolding.
- Persistent admin login + admin password + custom subjects list via `localStorage`.

### Services
- `base44` SDK client (`@/api/base44Client.js`) — the single service for all data + integrations.
- `getSubjects()` helper in `ManageSubjects.jsx` — reads the custom subject list from `localStorage` (key `dbms_subjects`), falling back to the default `SUBJECTS` constant.

### Business Logic
- Grading: `calcGrade(total)` → A (≥70), B (≥60), C (≥50), D (≥40), F (<40).
- Total per subject = `ca_score + exam_score` (CA max 30, Exam max 70, total max 100).
- Result total_score = sum of all subject totals.
- Result average = `Math.round((totalScore / subjects.length) * 100) / 100`.
- Overall percentage is a manually entered field (0–100), stored as `overall_percentage`.
- Position and principal_comment are stored fields but not auto-calculated in the current UI.

### Security Model
- The app is **public** (no platform auth required to load).
- RLS (Row-Level Security) per entity:
  - **Student, Teacher, Result:** all operations open (`{}`) — public/unauthenticated access — because the custom login system is not a platform user. This is a deliberate trade-off for the live public site.
  - **Complaint:** create open; read/update/delete require `role: "admin"` (platform admin).
  - **GalleryImage, Event:** read open; create/update/delete require `role: "admin"`.
  - **Payment:** create open; read/update/delete require `role: "admin"`.
- **Important:** Because Student/Teacher/Result are open, anyone with the SDK can read/write them. The custom login only gates the UI, not the data. (This matches the original project's decision to support the public localStorage admin login.)
- Passwords are stored in plain text in the Student and Teacher entities (not hashed). This is a known limitation of the original design.

### Folder Structure
See Section 20.

### Libraries / Frameworks / Dependencies
See `package.json` (Section 2 below lists the key ones):
- Runtime: react, react-dom, react-router-dom, @tanstack/react-query, framer-motion, lucide-react, moment, sonner, jspdf, html2canvas, @base44/sdk, class-variance-authority, clsx, tailwind-merge.
- UI primitives: @radix-ui/* (dialog, select, label, etc.), shadcn/ui components.
- Build: vite, @vitejs/plugin-react, tailwindcss, postcss, autoprefixer, eslint, typescript.

### Environment Variables
- `VITE_BASE44_APP_ID` — the Base44 app id.
- `VITE_BASE44_APP_BASE_URL` — the backend base URL.
- `VITE_BASE44_FUNCTIONS_VERSION` — functions version (optional).
These are read in `src/lib/app-params.js` (with URL-param and localStorage overrides for the platform's preview/embed flow).

### Build Configuration
- `vite.config.js` — Vite + React plugin + `@base44/vite-plugin`.
- `tailwind.config.js` — content paths, dark mode `class`, theme extend (colors from CSS vars, fonts, radius, keyframes/animations).
- `postcss.config.js` — tailwindcss + autoprefixer.
- `jsconfig.json` — `@/` path alias to `src/`.
- `index.html` — root mount point, fonts, meta.

---

## 3. Database Schema

The database has **7 custom entities** (collections) plus the built-in **User** entity. There is **no** separate Attendance, Sessions, Terms, Notifications, Classes, Subjects, or Permissions table — those are constants in `src/lib/constants.js` (CLASSES, TERMS, SUBJECTS) or localStorage (`dbms_subjects`). Below is every entity with every field.

> Built-in fields on EVERY record (never declared in schema): `id` (string, primary key), `created_date` (ISO date), `updated_date` (ISO date), `created_by_id` (string, the platform user id or null).

### 3.1 Student

**Purpose:** Stores every enrolled student and their login credentials.

| Field | Type | Required | Constraints / Notes |
|---|---|---|---|
| full_name | string | yes | Student's full name |
| admission_number | string | yes | Unique-ish login ID (e.g. "04926B"). Not enforced unique — duplicates possible (handled in login by matching password). |
| class_name | string (enum) | yes | One of: JSS1, JSS2, JSS3, SS1, SS2, SS3 |
| gender | string (enum) | no | Male, Female |
| date_of_birth | string (date) | no | ISO date |
| parent_name | string | no | Parent/guardian name |
| parent_phone | string | no | |
| parent_email | string | no | |
| boarding_status | string (enum) | no | Day, Boarding |
| password | string | yes | Plain-text login password (default convention: surname with first letter capitalised) |
| status | string (enum) | no | Active (default), Inactive |

**Primary key:** `id`. **Relationships:** Referenced by `Result.student_id` and `Result.student_name`. **RLS:** all open.

### 3.2 Teacher

**Purpose:** Stores staff records and teacher portal login credentials.

| Field | Type | Required | Constraints / Notes |
|---|---|---|---|
| full_name | string | yes | |
| staff_id | string | yes | Login ID for teacher portal |
| email | string | no | |
| phone | string | no | |
| subject | string | no | Subject taught |
| role | string (enum) | no | "Class Teacher" (can enter results) or "Teaching Staff" (view only) |
| assigned_class | string (enum) | no | One of: JSS1–SS3, or "" (empty = none) |
| password | string | yes | Plain-text login password |
| status | string (enum) | no | Active (default), Inactive |

**Primary key:** `id`. **Relationships:** `assigned_class` filters which students a class teacher manages. **RLS:** all open.

### 3.3 Result

**Purpose:** Stores a student's academic result for one term + session.

| Field | Type | Required | Constraints / Notes |
|---|---|---|---|
| student_id | string | yes | FK → Student.id |
| student_name | string | yes | Denormalised student full name |
| class_name | string (enum) | yes | JSS1–SS3 |
| term | string (enum) | yes | First Term, Second Term, Third Term |
| session | string | yes | e.g. "2024/2025" |
| subjects | array | no | Array of subject score objects (see below) |
| total_score | number | no | Sum of subject totals |
| average | number | no | Mean of subject totals (2 dp) |
| overall_percentage | number | no | Manually entered 0–100 |
| position | string | no | Class position (e.g. "1st") |
| teacher_comment | string | no | |
| principal_comment | string | no | |

**subjects[] item:**
| Field | Type | Notes |
|---|---|---|
| name | string | Subject name |
| ca_score | number | Continuous Assessment, max 30 |
| exam_score | number | Exam score, max 70 |
| total | number | ca_score + exam_score (max 100) |
| grade | string | A/B/C/D/F |

**Primary key:** `id`. **Composite uniqueness (logical, not enforced):** `student_id + term + session`. **RLS:** all open (so teachers/students can save/view without platform auth).

### 3.4 Payment

**Purpose:** Fee payment notifications from parents, confirmed/rejected by admin.

| Field | Type | Required | Constraints / Notes |
|---|---|---|---|
| student_name | string | yes | |
| class_name | string (enum) | yes | JSS1–SS3 |
| amount | number | yes | Amount in Naira |
| status | string (enum) | no | Pending (default), Confirmed, Rejected |
| payment_date | string (date) | no | Set to today on creation |

**Primary key:** `id`. **RLS:** create open; read/update/delete require platform `role: "admin"`.

### 3.5 Event

**Purpose:** School news/events shown on the public Events page and home preview.

| Field | Type | Required | Constraints / Notes |
|---|---|---|---|
| title | string | yes | |
| description | string | yes | |
| date | string (date) | yes | Event date |
| image_url | string | no | Uploaded file URL |
| category | string (enum) | no | Announcement, Activity, Exam, Holiday, Other |
| status | string (enum) | no | Published (default), Draft |

**Primary key:** `id`. **RLS:** read open; create/update/delete require `role: "admin"`.

### 3.6 GalleryImage

**Purpose:** Photos for the public gallery.

| Field | Type | Required | Constraints / Notes |
|---|---|---|---|
| title | string | yes | |
| image_url | string | yes | Uploaded file URL |
| category | string (enum) | no | Campus, Events, Sports, Academics, Activities, Other |

**Primary key:** `id`. **RLS:** read open; create/update/delete require `role: "admin"`.

### 3.7 Complaint

**Purpose:** Contact/complaint messages from parents/visitors.

| Field | Type | Required | Constraints / Notes |
|---|---|---|---|
| sender_name | string | yes | |
| student_name | string | yes | |
| email | string | yes | Sender email |
| message | string | yes | |
| status | string (enum) | no | New (default), Read, Resolved |

**Primary key:** `id`. **RLS:** create open; read/update/delete require `role: "admin"`.

### 3.8 User (built-in)

**Purpose:** Platform users (admins who can manage restricted entities). Read-only fields: `id`, `created_date`, `full_name`, `email`. Editable: `role` ("admin"/"user"). Not used by the custom login system. Users join via platform invites, not created directly.

### Tables that DO NOT exist
There is no separate: Admins table (admin is a constant + localStorage), Parents table (parent info is fields on Student), Classes table (constant array), Subjects table (constant + localStorage), Sessions/Terms tables (constants), Attendance (not implemented), Notifications (not implemented), File uploads table (files are URLs stored in entity fields), Permissions table (RLS config per entity).

---

## 4. Authentication

### Login Process

**Admin login** (`/admin-login`):
1. User enters username + password.
2. Validate both non-empty (red error text otherwise).
3. Read saved password: `localStorage.getItem('admin_password') || ADMIN_DEFAULT_PASSWORD` ("DBMS").
4. If username !== "DivineBlossom" → "Username not recognised".
5. Else if password !== saved → "Incorrect password".
6. Else: set `localStorage.admin_logged_in = "true"`, toast "Welcome, Admin!", navigate to `/admin-dashboard`.

**Teacher login** (`/teacher-portal`):
1. Enter staff_id + password.
2. Validate non-empty.
3. `base44.entities.Teacher.filter({ staff_id, status: 'Active' })`.
4. If none → "Staff ID not found".
5. Else if `byId[0].password !== password` → "Incorrect password".
6. Else: set teacher in state, `loggedIn = true`, toast welcome.

**Student login** (`/student-portal`):
1. Enter admission_number + password.
2. Validate non-empty.
3. `base44.entities.Student.filter({ admission_number, status: 'Active' })`.
4. `match = byAdm.find(s => s.password === password)` — finds the right student even if admission numbers are duplicated.
5. If none → "Admission number not found".
6. Else if no match → "Incorrect password".
7. Else: set student in state, `loggedIn = true`, toast welcome.

### Registration Process
- **Students and Teachers** are NOT self-registered. They are created by the **Admin** in the dashboard (Manage Students / Manage Teachers). Default password convention: student's surname with first letter capitalised.
- **Admin** is not registered — credentials are hardcoded constants, changeable via Settings.
- **Platform Users** (for RLS admin operations) join via Base44 invites — not part of this app's UI.

### Password Handling
- Plain text, stored in the entity (`Student.password`, `Teacher.password`) and in `localStorage` for admin.
- Admin password change (Settings): validates current password, new ≥ 4 chars, new === confirm, then writes `localStorage.admin_password`.
- No hashing, no reset flow, no email-based reset.

### Sessions
- Admin: persisted in `localStorage` (`admin_logged_in`). Survives refresh. Logout removes the key.
- Teacher & Student: in-memory React state only. Refresh = logged out. Logout resets state.

### Tokens
- The platform auth token (if any) is managed by the SDK/AuthContext and stored under `base44_access_token`. Not used by the custom portals.

### Authorization
- **Admin dashboard** is gated by `localStorage.admin_logged_in === 'true'`. The dashboard's `useEffect` redirects to `/admin-login` if not set.
- **Teacher portal** role check: `teacher.role === 'Class Teacher'` → full result entry; else → view-only message.
- **Student portal** has no sub-roles.

### User Roles / Permission System
- App-level roles (custom): Admin (constant), Class Teacher, Teaching Staff (Teacher.role enum), Student (implicit).
- Platform-level roles: `admin` / `user` on the built-in User entity — only affects RLS for Complaint/Event/Gallery/Payment admin operations.

---

## 5. Every Page

Routes (from `src/App.jsx`):
- `/` → Home (layout)
- `/about` → About (layout)
- `/academics` → Academics (layout)
- `/gallery` → Gallery (layout)
- `/events` → Events (layout)
- `/payment` → Payment (layout)
- `/complaint` → Complaint (layout)
- `/student-portal` → StudentPortal (layout)
- `/teacher-portal` → TeacherPortal (layout)
- `/admin-login` → AdminLogin (no layout)
- `/admin-dashboard` → AdminDashboard (no layout)
- `*` → PageNotFound

### 5.1 Home (`/`)
**Purpose:** Landing page. **Layout:** Hero + About preview + Why Choose Us + Events preview + Testimonials. **Components:** `HeroSection`, `AboutPreview`, `WhyChooseUs`, `EventsPreview`, `TestimonialsSection`. **API calls:** `EventsPreview` fetches `Event.filter({ status: 'Published' }, '-date', 3)` via react-query key `events-preview`. **Navigation:** hero buttons to /student-portal, /teacher-portal, /payment, /complaint; "Learn More" to /about; "View All" to /events.

### 5.2 About (`/about`)
**Purpose:** School story, mission/vision, proprietor. **Layout:** header banner, two-column image+text, mission & vision cards, proprietor section. **Components:** motion divs, lucide Target/Eye/BookOpen. **API calls:** none. **Data:** constants (SCHOOL_NAME, MOTTO, ESTABLISHED, images).

### 5.3 Academics (`/academics`)
**Purpose:** Academic programs. **Layout:** header + 4 level cards (Preschool, Basic, JSS, SS) each with icon, age range, description, feature chips. **Data:** hardcoded `LEVELS` array. **API calls:** none.

### 5.4 Gallery (`/gallery`)
**Purpose:** Photo gallery with category filter + lightbox. **Layout:** header, category pills, responsive grid, click → fullscreen lightbox. **State:** `category` (default "All"), `selected` (lightbox). **API:** `GalleryImage.list('-created_date', 50)` via react-query key `gallery`. **Filter:** client-side by category. **Empty state:** "No images in this category yet."

### 5.5 Events (`/events`)
**Purpose:** Public events/news list. **Layout:** header + vertical list of event cards (image left, content right). **API:** `Event.filter({ status: 'Published' }, '-date', 50)` key `events`. **Loading:** spinner. **Empty:** "No events posted yet." **Date format:** moment `MMMM D, YYYY`.

### 5.6 Payment (`/payment`)
**Purpose:** Fee payment notification. **Layout:** header, two columns: bank details card (with copy-to-clipboard) + payment form. **Form fields:** student_name, class_name (select CLASSES), amount. **Validation:** all fields required (toast error). **Submit:** creates `Payment` record with `payment_date = today`, shows success screen with WhatsApp link (`wa.me/${WHATSAPP}` with prefilled text). **API:** `Payment.create`. **Copy:** `navigator.clipboard.writeText(ACCOUNT_NUMBER)`.

### 5.7 Complaint (`/complaint`)
**Purpose:** Contact/complaint form + contact info. **Layout:** header, two columns: contact info + form. **Form fields:** sender_name, student_name, email, message. **Validation:** all required. **Submit:** `Complaint.create(form)` then `SendEmail({ to: COMPLAINT_EMAIL, subject, body })`. Success screen "Message Sent!". **API:** Complaint.create + Core.SendEmail.

### 5.8 StudentPortal (`/student-portal`)
**Purpose:** Student login + result viewing. **Layout:** if not logged in → login card (admission_number + password). If logged in → welcome header + result search (term select + session input + View button) + result cards. **API:** login `Student.filter({ admission_number, status: 'Active' })`; results `Result.filter({ student_id, term, session })`. **Components:** `ResultView` (with PDF download). **Validation:** red errors, toasts. **Logout:** resets state.

### 5.9 TeacherPortal (`/teacher-portal`)
**Purpose:** Teacher login + result entry. **Layout:** login card (staff_id + password) OR dashboard. **Role logic:** Class Teacher → `ClassTeacherDashboard`; Teaching Staff → view-only message. **ClassTeacherDashboard:** student select (filtered by `teacher.assigned_class`), term select, session input, Load button → loads existing result or seeds blank subjects from `getSubjects()`. Subject score table (name, CA 0–30, Exam 0–70, auto total, auto grade), add/remove subject, overall percentage input, teacher comment textarea, Save/Update button. **API:** `Student.filter({ class_name, status: 'Active' })`, `Result.filter`, `Result.update`/`Result.create`. **Grading:** calcGrade. **Toasts:** success/error. **Logout:** resets state.

### 5.10 AdminLogin (`/admin-login`)
**Purpose:** Admin login. **Layout:** centered card on primary background, "Back to Home" link. **Fields:** username, password. **Logic:** see Section 4. **Navigation:** success → `/admin-dashboard`.

### 5.11 AdminDashboard (`/admin-dashboard`)
**Purpose:** Admin control panel. **Layout:** top bar (logo + logout), left sidebar (desktop) / bottom bar (mobile) with 9 tabs, content area. **Tabs:** Students, Teachers, Results, Payments, Events, Gallery, Complaints, Subjects, Settings. **Auth guard:** redirects to `/admin-login` if not logged in. **Logout:** clears `admin_logged_in`, navigates to login.

### 5.12 PageNotFound (`*`)
**Purpose:** 404 fallback.

---

## 6. Dashboard

The Admin Dashboard is the only "dashboard". It has **no charts/widgets/statistics cards** — it is a tabbed CRUD interface.

**Widgets / Tabs:**
1. **Students** (`ManageStudents`) — table with search (name/admission no) + class filter, Add/Edit dialog (full_name, admission_number, class, gender, DOB, parent name/phone/email, boarding status, status, password), delete with confirm.
2. **Teachers** (`ManageTeachers`) — table, Add/Edit dialog (full_name, staff_id, subject, email, phone, role, assigned_class, password, status), delete.
3. **Results** (`ManageResults`) — table filtered by class + term, view detail dialog, delete.
4. **Payments** (`ManagePayments`) — table, Confirm/Reject actions for Pending payments.
5. **Events** (`ManageEvents`) — list, Add/Edit dialog with image upload, delete.
6. **Gallery** (`ManageGallery`) — grid, Add dialog with image upload, delete.
7. **Complaints** (`ManageComplaints`) — list, Mark Read / Resolve actions.
8. **Subjects** (`ManageSubjects`) — add/delete/reset subjects (localStorage).
9. **Settings** (`AdminSettings`) — change admin password.

**Data sources:** each tab uses its own react-query key (`admin-students`, `admin-teachers`, `admin-results`, `admin-payments`, `admin-events`, `admin-gallery`, `admin-complaints`). **Refresh logic:** `qc.invalidateQueries` after every mutation.

---

## 7. Student Portal

**Features:**
- Login by admission number + password (handles duplicate admission numbers by matching password).
- View welcome banner with name, class, admission number.
- Select term (First/Second/Third) and type session (e.g. 2024/2025).
- "View Result" fetches all results matching student_id + term + session.
- Each result renders as a `ResultView` card: header (school logo, name, address, motto), student info, subjects table (CA, Exam, Total, Grade), summary (Total, Average, Position), teacher/principal comments.
- Download result as PDF (multi-page A4).
- Logout.

**Business rules:** only `status: 'Active'` students can log in. No self-registration. No password change for students (admin must change it).

---

## 8. Teacher Portal

**Features:**
- Login by staff_id + password (Active only).
- Role-based UI:
  - **Class Teacher:** full result entry dashboard.
  - **Teaching Staff:** view-only informational message.
- Class Teacher dashboard:
  - Student dropdown auto-filtered to `teacher.assigned_class` Active students.
  - Select student + term + session → "Load" fetches existing result or seeds blank subject rows from the custom subject list.
  - Editable subject table: name (text), CA (number 0–30), Exam (number 0–70); total and grade auto-calculated on change.
  - Add Subject / Remove Subject (trash icon).
  - Overall Percentage input (0–100).
  - Teacher's Comment textarea.
  - Save Result (create) or Update Result (if existing) — button label changes accordingly.
- Toasts for success/error; saving state disables button.

**Business rules:** only Class Teachers can save. `assigned_class` drives which students are visible. Results are keyed by student_id + term + session (one result per student per term per session).

---

## 9. Parent Portal

There is **no dedicated parent portal**. Parents interact via:
- The public **Payment** page (submit payment notification + WhatsApp proof).
- The public **Complaint/Contact** page (send message + email).
- Parent contact details are stored as fields on the **Student** entity (parent_name, parent_phone, parent_email).

---

## 10. Admin Portal

The Admin Portal = Admin Dashboard (Section 6). Full CRUD across all entities, subject management, and password settings. Auth via localStorage. All mutations invalidate react-query caches. Image uploads via `UploadFile`. Complaints also trigger emails on the public side (not admin side).

---

## 11. Result Management

### Result Entry
- Class Teacher selects student + term + session → Load.
- If a result exists, it loads subjects, comment, overall_percentage for editing.
- If not, seeds one row per subject from `getSubjects()` with zeroed scores.
- Teacher edits CA (0–30) and Exam (0–70) per subject; total and grade auto-update.

### Editing
- Same form as entry; if `existingResult` is set, Save calls `Result.update(existingResult.id, data)` and the button reads "Update Result".

### Approval / Publishing
- There is **no formal approval workflow**. Saving a result makes it immediately visible to the student (Student Portal reads the same Result entity). Admin can delete results from Manage Results.

### Printing / Report Cards
- Student Portal renders each result as a `ResultView` card with a "Download Result (PDF)" button.
- PDF generation: `html2canvas` captures the card at scale 2, then `jsPDF` paginates the image across multiple A4 pages (so tall results aren't cut off). File name: `${student.full_name}_${result.term}_${result.session}.pdf`.

### Grading
```
total = ca_score + exam_score
grade:
  total >= 70 → A
  total >= 60 → B
  total >= 50 → C
  total >= 40 → D
  else       → F
```

### GPA / Average Calculations
- `total_score = sum(subject.total for all subjects)`
- `average = Math.round((total_score / subjects.length) * 100) / 100` (2 decimal places)
- `overall_percentage` = manually entered 0–100 (not auto-derived).
- There is **no GPA**; the system uses percentage average.

### Position Calculation
- `position` is a stored string field but is **not auto-calculated** by the current UI. It can be set via the database/admin manually. Displayed as "—" when empty.

### Remarks
- `teacher_comment` — entered by the class teacher in the result form.
- `principal_comment` — stored field, displayed on the report card if present; not editable in the current UI.

### Promotion Logic
- **Not implemented.** There is no automated class promotion flow. Admin can edit a student's `class_name` manually.

---

## 12. Attendance System

**Not implemented.** There is no Attendance entity, page, or workflow in this application.

---

## 13. Fees System

**Workflow:**
1. Parent opens `/payment`, sees bank details (Wema Bank, D'Blossom Private School, acct 0250842056) with copy button.
2. Parent fills student_name, class, amount and clicks "I Have Paid ✓".
3. A `Payment` record is created with `status: 'Pending'` and `payment_date = today`.
4. Success screen shows a WhatsApp link (prefilled message) to send proof of payment.
5. Admin opens Dashboard → Payments, sees pending payments, clicks **Confirm** (→ status "Confirmed") or **Reject** (→ "Rejected").
6. Status badges: Pending (yellow), Confirmed (green), Rejected (red).

**Notes:** No online card payment is processed — it is a notification + manual confirmation flow. (Stripe packages are installed but not used.)

---

## 14. File Upload System

**Storage:** Files are uploaded via `base44.integrations.Core.UploadFile({ file })`, which returns `{ file_url }`. The URL is stored in the entity field (`Event.image_url`, `GalleryImage.image_url`). Files live on the Base44 file storage.

**Accepted file types:** images only (`accept="image/*"`) for both events and gallery.

**Limits:** Imposed by the platform's UploadFile integration (not customised). No client-side size validation in the current code.

**Usage:**
- `ManageEvents` — optional event image; preview shown in form and list.
- `ManageGallery` — required image; preview shown in form; grid display with hover overlay.

---

## 15. Notification System

**In-app notifications:** Not implemented (no notifications entity or bell UI).

**Email notifications:** The Complaint form calls `SendEmail` to `COMPLAINT_EMAIL` ("Sabbatah77@gmail.com") on submission. `SendEmail` only delivers to **registered app users** on the platform — if the complaint email is not a registered user, delivery may fail silently. (This is a platform limitation.)

**WhatsApp:** Payment success and Footer link to `https://wa.me/${WHATSAPP}`.

---

## 16. Business Rules

1. The app is **public** — no platform login required to browse.
2. Admin credentials: username `DivineBlossom`, default password `DBMS` (changeable, stored in `localStorage.admin_password`).
3. Admin login state persists in `localStorage.admin_logged_in`; teacher/student logins are in-memory only.
4. Only `status: 'Active'` students and teachers can log in.
5. Student login matches by admission_number **and** password (handles duplicate admission numbers).
6. Teacher passwords and student passwords are plain text.
7. Default student password convention: surname, first letter capitalised.
8. Only `role === 'Class Teacher'` teachers can enter/edit results; `Teaching Staff` get view-only.
9. A class teacher only sees students in their `assigned_class`.
10. One result per student per term per session (enforced by load/save logic, not by a unique DB index).
11. CA max 30, Exam max 70, Total max 100 (enforced via input `min`/`max` attributes, not server-side).
12. Grade bands: A≥70, B≥60, C≥50, D≥40, F<40.
13. Average = round(mean of subject totals × 100)/100.
14. Overall percentage is manually entered (0–100).
15. Position is a free-text field, not auto-calculated.
16. Subjects list is customisable per browser via `localStorage.dbms_subjects`, defaulting to the `SUBJECTS` constant; duplicates not allowed.
17. Payments start as Pending; admin confirms or rejects.
18. Complaints start as New; admin marks Read then Resolved.
19. Events/Gallery with `status: 'Published'` (events) appear publicly; Drafts are hidden.
20. RLS: Student/Teacher/Result fully open; Complaint/Event/Gallery/Payment admin-only for read/write (platform role "admin").
21. PDF report cards paginate across multiple A4 pages.
22. All forms show red error text for invalid/empty required fields.
23. Every portal/auth page has a "Back to Home" link.
24. Toasts (sonner) confirm every CRUD success/failure.
25. Admin mutations invalidate react-query caches so lists refresh.

---

## 17. UI Design System

### Colors (CSS variables in `src/index.css`, mapped in `tailwind.config.js`)
Light (`:root`):
- `--background` 0 0% 100% (white)
- `--foreground` 220 30% 15% (dark navy)
- `--primary` 220 70% 25% (deep navy blue) — main brand
- `--primary-foreground` 0 0% 100%
- `--accent` 0 75% 50% (red) — used for "Payment"/contact CTAs
- `--gold` 43 85% 55% (gold) — secondary accent, motto, highlights
- `--gold-foreground` 43 85% 15%
- `--secondary` 220 15% 95%
- `--muted` 220 15% 96%; `--muted-foreground` 220 10% 45%
- `--border`/`--input` 220 15% 90%
- `--destructive` 0 84% 60%
- `--card`/`--popover` white
- `--radius` 0.625rem
- Sidebar tokens (navy bg, gold primary).

Dark (`.dark`): navy backgrounds, lighter primary, same gold.

Tailwind classes used: `bg-primary`, `text-primary-foreground`, `bg-gold`, `text-gold`, `bg-accent`, `bg-card`, `bg-secondary`, `text-muted-foreground`, `border-border`, etc.

### Fonts
- `--font-heading`: 'Playfair Display', serif (headings).
- `--font-body`: 'Inter', sans-serif (body).
- Imported via Google Fonts at top of `src/index.css`.
- Tailwind: `font-heading`, `font-body`.

### Spacing
- Tailwind default spacing scale. Sections use `py-16 md:py-24` / `py-20 md:py-28`. Cards `p-6 md:p-8/p-10`. Container `max-w-7xl mx-auto px-4 sm:px-6 lg:px-8`.

### Icons
- lucide-react only. Key icons: GraduationCap, Users, CreditCard, MessageSquare, Shield, Home, LogOut, FileText, Download, Plus, Pencil, Trash2, Search, Eye, Save, CheckCircle, X, Calendar, MapPin, Mail, Phone, Clock, Send, Copy, Award, BookOpen, Target, Eye, Heart, Trophy, Lock, Image, BookMarked, Loader2, ChevronLeft/Right, Quote, ArrowRight, Menu, X, Baby.

### Borders / Cards
- `border border-border`, `rounded-xl`/`rounded-2xl`, `shadow-sm`/`shadow-lg`, `bg-card`.

### Tables
- `w-full text-sm`, `border-collapse`, header `bg-secondary/50`, rows `border-t border-border/50 hover:bg-secondary/30`, status pills `px-2 py-0.5 rounded-full text-xs`.

### Buttons
- shadcn `Button` variants: default (primary), outline, ghost, destructive, secondary, link. Sizes: default, sm, lg, icon. Gold CTA: `bg-gold text-gold-foreground hover:bg-gold/90`.

### Animations
- framer-motion: `initial/animate/whileInView` fade+slide (opacity 0→1, y 20→0), testimonial carousel `AnimatePresence` slide, payment success `scale: 0→1`.
- Tailwind keyframes: `fade-in-up`, `fade-in`, accordion.
- Hover: `hover:shadow-lg`, `hover:-translate-y-0.5`, `group-hover:scale-105`, `group-hover:translate-x-1`, color transitions.

### Responsive Behavior
- Mobile-first. Navbar collapses to hamburger under `lg`. Admin sidebar → bottom tab bar under `md`. Grids: `grid-cols-1 md:grid-cols-2 lg:grid-cols-3/4`. Padding/font sizes scale with `md:`/`lg:` breakpoints.

### Dark/Light Mode
- Tokens defined for both; `darkMode: ["class"]`. The app does not include a theme toggle UI in the current code, so light mode is default.

### Other Details
- Custom thin scrollbar (`::-webkit-scrollbar` 6px, muted thumb).
- Backdrop blur navbar (`bg-white/95 backdrop-blur-md`).
- Hero gradient overlay `from-primary/80 via-primary/70 to-primary/90`.
- Status color mapping: Active/Confirmed/Published = green; Inactive/Rejected = red; Pending = yellow; New = accent.

---

## 18. Components

### Layout / Shared
- **`Layout`** (`src/components/Layout.jsx`): wraps pages with `Navbar` + `<main className="flex-1 pt-16 md:pt-20"><Outlet /></main>` + `Footer`. Props: none (uses Outlet).
- **`Navbar`**: fixed top nav. Props: none. State: `open` (mobile menu). Links: Home, About, Academics, Gallery, Events, Payment, Complaint + Student Portal (primary), Admin (gold). Mobile menu adds Teacher Portal.
- **`Footer`**: 4-column footer (about, quick links, portals, contact) + WhatsApp button. Props: none.

### Home sections
- **`HeroSection`**: full-height hero with bg image + gradient, motto, 4 CTA buttons. Props: none.
- **`AboutPreview`**: image + text + "Learn More". Props: none.
- **`WhyChooseUs`**: 6 feature cards. Props: none. Data: internal `FEATURES`.
- **`EventsPreview`**: 3 latest published events. Props: none. Fetches `events-preview`.
- **`TestimonialsSection`**: carousel of `TESTIMONIALS` with prev/next + dots. Props: none. State: `current`.

### Portal
- **`ResultView`** (`src/components/portal/ResultView.jsx`): Props: `result`, `student`. Renders report card + PDF download. Behavior: `getGradeColor(grade)` maps A→green, B→blue, C→amber, D→orange, F→red. `downloadPDF` uses html2canvas + jsPDF with multi-page pagination.

### Admin (each is a default export, no props)
- **`ManageStudents`**: table + search + class filter + Add/Edit Dialog. State: `open, form, editing, search, filterClass, saving, errors`. Validation: REQUIRED = full_name, admission_number, class_name, password. Helpers: `field()`, `selectField()` render labelled inputs with red error state.
- **`ManageTeachers`**: table + Add/Edit Dialog. REQUIRED = full_name, staff_id, password. Cleans `assigned_class` to "" if falsy. Role + assigned_class selects.
- **`ManageResults`**: table + class/term filters + view Dialog + delete. No edit (results edited in Teacher Portal).
- **`ManagePayments`**: table + Confirm/Reject. `updateStatus(id, status)`.
- **`ManageComplaints`**: list + Mark Read / Resolve.
- **`ManageEvents`**: list + Add/Edit Dialog with image upload (`UploadFile`). Invalidates `admin-events` + `events-preview`.
- **`ManageGallery`**: grid + Add Dialog with image upload. Invalidates `admin-gallery` + `gallery`.
- **`ManageSubjects`**: add/delete/reset subjects, persisted to `localStorage.dbms_subjects`. Exports `getSubjects()` (used by TeacherPortal).
- **`AdminSettings`**: change admin password (current/new/confirm), validates, writes `localStorage.admin_password`.

### UI primitives (shadcn, `src/components/ui/*`)
Standard shadcn/ui exports: `button` (Button + buttonVariants), `input`, `label`, `textarea`, `select` (Select, SelectTrigger, SelectContent, SelectItem, SelectValue, etc.), `dialog` (Dialog, DialogContent, DialogHeader, DialogTitle), `badge`, `card`, `table`, `tabs`, `toast`/`toaster`/`use-toast`, `sonner`, `accordion`, `alert`/`alert-dialog`, `avatar`, `calendar`, `carousel`, `chart`, `checkbox`, `collapsible`, `command`, `context-menu`, `drawer`, `dropdown-menu`, `form`, `hover-card`, `input-otp`, `menubar`, `navigation-menu`, `pagination`, `popover`, `progress`, `radio-group`, `resizable`, `scroll-area`, `separator`, `sheet`, `sidebar`, `slider`, `switch`, `toggle`/`toggle-group`, `tooltip`. Each exports only its own primitives.

### Other
- **`UserNotRegisteredError`**: shown when platform auth returns user_not_registered.
- **`ProtectedRoute`**: exists but not actively used (app is public).
- **`PageNotFound`** (`src/lib/PageNotFound.jsx`): 404.

---

## 19. API Documentation

All endpoints are auto-generated by Base44 per entity. Base URL: the app's backend URL. Auth: platform token in header when present. Pattern: `POST/GET/PUT/DELETE /api/apps/prod/entities/<EntityName>...`. The frontend uses the SDK, not raw fetch. Below documents the **logical** operations used.

### Student
- `list(sort='-created_date', limit=200)` — admin list.
- `filter({ admission_number, status:'Active' })` — student login.
- `filter({ class_name, status:'Active' })` — teacher loads class students.
- `create(data)` — admin add. Input: full schema. Errors: RLS/validation. Auth: open.
- `update(id, data)` — admin edit. Auth: open.
- `delete(id)` — admin delete. Auth: open.

### Teacher
- `filter({ staff_id, status:'Active' })` — teacher login.
- `list('-created_date', 100)` — admin list.
- `create/update/delete` — admin. Auth: open.

### Result
- `filter({ student_id, term, session })` — load result (student + teacher).
- `list('-created_date', 200)` — admin list.
- `create(data)` — teacher save. Input: student_id, student_name, class_name, term, session, subjects[], total_score, average, overall_percentage, teacher_comment.
- `update(id, data)` — teacher update.
- `delete(id)` — admin delete. Auth: open.

### Payment
- `create({ student_name, class_name, amount, payment_date })` — public. Auth: open.
- `list('-created_date', 100)` — admin. Auth: read requires platform admin.
- `update(id, { status })` — admin confirm/reject. Auth: admin.

### Event
- `filter({ status:'Published' }, '-date', limit)` — public.
- `list('-date', 100)` — admin.
- `create/update/delete` — admin. Auth: admin.
- Image upload via `UploadFile`.

### GalleryImage
- `list('-created_date', 50/100)` — public/admin.
- `create/delete` — admin. Auth: admin.

### Complaint
- `create({ sender_name, student_name, email, message })` — public. Auth: open.
- `list('-created_date', 100)` — admin. Auth: admin.
- `update(id, { status })` — admin. Auth: admin.

### Integrations
- `UploadFile({ file })` → `{ file_url }`. Input: multipart file. Used by Events/Gallery.
- `SendEmail({ to, subject, body })` → sends to registered users only. Used by Complaint.

### Errors
- SDK throws on non-2xx; frontend catches in try/catch for result saving and shows toast. Public forms rely on toast error messages. Validation errors shown inline in red.

---

## 20. Folder Structure

```
/
├── index.html
├── package.json
├── vite.config.js
├── tailwind.config.js
├── postcss.config.js
├── jsconfig.json
├── eslint.config.js
├── README.md
├── DOCUMENTATION.md  (this file)
├── base44/
│   ├── config.jsonc
│   └── entities/
│       ├── Student.jsonc
│       ├── Teacher.jsonc
│       ├── Result.jsonc
│       ├── Payment.jsonc
│       ├── Event.jsonc
│       ├── GalleryImage.jsonc
│       └── Complaint.jsonc
└── src/
    ├── main.jsx
    ├── App.jsx
    ├── index.css
    ├── api/
    │   └── base44Client.js
    ├── lib/
    │   ├── app-params.js
    │   ├── AuthContext.jsx
    │   ├── constants.js
    │   ├── query-client.js
    │   ├── utils.js
    │   └── PageNotFound.jsx
    ├── components/
    │   ├── Layout.jsx
    │   ├── Navbar.jsx
    │   ├── Footer.jsx
    │   ├── ProtectedRoute.jsx
    │   ├── UserNotRegisteredError.jsx
    │   ├── ui/  (shadcn primitives)
    │   ├── home/
    │   │   ├── HeroSection.jsx
    │   │   ├── AboutPreview.jsx
    │   │   ├── WhyChooseUs.jsx
    │   │   ├── EventsPreview.jsx
    │   │   └── TestimonialsSection.jsx
    │   ├── portal/
    │   │   └── ResultView.jsx
    │   └── admin/
    │       ├── ManageStudents.jsx
    │       ├── ManageTeachers.jsx
    │       ├── ManageResults.jsx
    │       ├── ManagePayments.jsx
    │       ├── ManageComplaints.jsx
    │       ├── ManageEvents.jsx
    │       ├── ManageGallery.jsx
    │       ├── ManageSubjects.jsx
    │       └── AdminSettings.jsx
    ├── pages/
    │   ├── Home.jsx
    │   ├── About.jsx
    │   ├── Academics.jsx
    │   ├── Gallery.jsx
    │   ├── Events.jsx
    │   ├── Payment.jsx
    │   ├── Complaint.jsx
    │   ├── StudentPortal.jsx
    │   ├── TeacherPortal.jsx
    │   ├── AdminLogin.jsx
    │   └── AdminDashboard.jsx
    ├── hooks/
    │   └── use-mobile.jsx
    └── utils/
        └── index.ts
```

---

## 21. Source Code Explanation

### `src/App.jsx`
Wraps the app in `AuthProvider` → `QueryClientProvider` → `Router`. `AuthenticatedApp` shows a loading spinner while `isLoadingPublicSettings`/`isLoadingAuth`, handles `user_not_registered` and `auth_required` (allows public access on auth_required). Renders `<Routes>` with a layout route wrapping public+portal pages, standalone admin routes, and a 404 catch-all.

### `src/lib/AuthContext.jsx`
`AuthProvider` runs `checkAppState` on mount: fetches public settings via an axios client to `/api/apps/public/prod/public-settings/by-id/<appId>`; if a token exists, calls `base44.auth.me()`. Exposes `user, isAuthenticated, isLoadingAuth, isLoadingPublicSettings, authError, appPublicSettings, logout, navigateToLogin, checkAppState`. `logout` uses `base44.auth.logout`; `navigateToLogin` uses `base44.auth.redirectToLogin`.

### `src/api/base44Client.js`
Creates the SDK client with `appId, token, functionsVersion, appBaseUrl, requiresAuth: false`. Exported as `base44`.

### `src/lib/app-params.js`
Reads `app_id`, `access_token`, `from_url`, `functions_version`, `app_base_url` from URL params → localStorage → `import.meta.env`. Handles `clear_access_token=true`. SSR-safe (node vs window).

### `src/lib/constants.js`
All school identity, contact, banking, image URLs, admin credentials, CLASSES, TERMS, SUBJECTS, TESTIMONIALS.

### `src/components/admin/ManageSubjects.jsx`
`getSubjects()` reads `localStorage.dbms_subjects` (JSON) or falls back to `SUBJECTS`. Component keeps state in sync with localStorage via `useEffect`. Add (no duplicates), delete (confirm), reset to default.

### `src/pages/StudentPortal.jsx`
Login → state. `searchResults` filters Result by student_id+term+session. Renders `ResultView` per result. Logout resets.

### `src/pages/TeacherPortal.jsx`
Login → state. `ClassTeacherDashboard`: loads students by assigned_class; `loadResult` loads existing or seeds; `updateSubject` recalculates total+grade; `saveResult` creates or updates. `calcGrade` grading function.

### `src/components/portal/ResultView.jsx`
Renders report card. `downloadPDF`: html2canvas (scale 2, white bg, CORS) → jsPDF A4 portrait → paginates image across pages by decrementing `position` by `pdfPageHeight` while `heightLeft > 0`.

### Admin components
Each uses react-query for list, local state for form/dialog, `validate()` for required fields, try/catch with toast on save, `qc.invalidateQueries` after mutation. `field()`/`selectField()` helpers in ManageStudents/ManageTeachers produce labelled inputs with red error styling.

---

## 22. Deployment (outside Base44)

To deploy this as a static frontend + your own backend:

1. **Frontend (Netlify):**
   - `npm run build` produces `dist/`.
   - On Netlify: new site from Git, build command `npm run build`, publish directory `dist`.
   - Add a SPA redirect: `/* /index.html 200` (file `public/_redirects` or netlify.toml) so client-side routes work.
   - Set env vars `VITE_BASE44_APP_ID`, `VITE_BASE44_APP_BASE_URL` (or your new backend URL) in Netlify env settings.

2. **Backend (Supabase):** see Section 23.

3. **Environment:** set the API base URL to point to your backend; replace `@base44/sdk` calls with your own API client (see migration guide).

---

## 23. Migration Guide (React + GitHub + Netlify + Supabase)

### Step 1 — Source code
1. Create a GitHub repo. Push the entire project (`git init`, commit, `git remote add origin <url>`, `git push`).
2. The code is plain React + Vite — no Base44-specific build step is required to run the frontend.

### Step 2 — Replace the data layer with Supabase
1. Create a Supabase project. In the SQL editor, create tables mirroring the entities (Section 3):
   - `students` (id uuid pk default gen_random_uuid(), full_name text, admission_number text, class_name text, gender text, date_of_birth date, parent_name text, parent_phone text, parent_email text, boarding_status text, password text, status text default 'Active', created_at timestamptz default now(), updated_at timestamptz default now())
   - `teachers` (id, full_name, staff_id, email, phone, subject, role, assigned_class, password, status, created_at, updated_at)
   - `results` (id, student_id uuid, student_name, class_name, term, session, subjects jsonb, total_score numeric, average numeric, overall_percentage numeric, position text, teacher_comment text, principal_comment text, created_at, updated_at)
   - `payments` (id, student_name, class_name, amount numeric, status text default 'Pending', payment_date date, created_at, updated_at)
   - `events` (id, title, description, date date, image_url, category, status default 'Published', created_at, updated_at)
   - `gallery_images` (id, title, image_url, category, created_at, updated_at)
   - `complaints` (id, sender_name, student_name, email, message, status default 'New', created_at, updated_at)
2. Enable Row Level Security. Policies:
   - students/teachers/results: allow all (anon + authenticated) for select/insert/update/delete (matches current open RLS) — or tighten with a real auth design.
   - payments/complaints/events/gallery: anon can insert (payments/complaints) and select (events/gallery); only authenticated admin can update/delete.
3. Create a Supabase Storage bucket `school-assets` for image uploads; generate public URLs.

### Step 3 — Swap the SDK for Supabase client
1. `npm install @supabase/supabase-js`.
2. Create `src/api/supabaseClient.js`:
   ```js
   import { createClient } from '@supabase/supabase-js';
   export const supabase = createClient(import.meta.env.VITE_SUPABASE_URL, import.meta.env.VITE_SUPABASE_ANON_KEY);
   ```
3. Create a thin data layer `src/api/db.js` that mirrors the methods used:
   ```js
   // list, filter, get, create, update, delete per table
   // e.g.
   export const Students = {
     list: async () => { const { data } = await supabase.from('students').select('*').order('created_at',{ascending:false}).limit(200); return data; },
     filter: async (q) => { const { data } = await supabase.from('students').select('*').match(q); return data; },
     create: async (d) => { const { data } = await supabase.from('students').insert(d).select().single(); return data; },
     update: async (id, d) => { const { data } = await supabase.from('students').update(d).eq('id',id).select().single(); return data; },
     delete: async (id) => { await supabase.from('students').delete().eq('id',id); },
   };
   // ...repeat for Teachers, Results, Payments, Events, GalleryImages, Complaints
   ```
4. Replace `base44.entities.X` calls with the new `X` object across all pages/components.
5. Replace `UploadFile` with Supabase Storage:
   ```js
   const { data } = await supabase.storage.from('school-assets').upload(`${Date.now()}_${file.name}`, file);
   const { data: { publicUrl } } = supabase.storage.from('school-assets').getPublicUrl(data.path);
   ```
6. Replace `SendEmail` with a Supabase Edge Function (Deno) that uses Resend/SendGrid, or trigger a Supabase webhook. (Supabase has no built-in email to arbitrary addresses.)
7. Keep `react-query`, `sonner`, `framer-motion`, `lucide-react`, `jspdf`, `html2canvas`, `moment` as-is.

### Step 4 — Auth (optional hardening)
- Replace plain-text password checks with Supabase Auth: create auth users for teachers/admin, or keep the localStorage admin + entity-password approach but hash passwords (bcrypt) via an Edge Function.
- For a faithful migration, keep the current custom login but move password comparison to an Edge Function that hashes.

### Step 5 — Build & deploy
1. `npm run build`.
2. Netlify: connect the GitHub repo, build `npm run build`, publish `dist`, add `_redirects` with `/*  /index.html  200`.
3. Set env vars: `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`.
4. Push to GitHub → Netlify auto-deploys.

### Step 6 — Data migration
- Export existing Base44 records (via the admin UI or SDK) as JSON/CSV and import into Supabase tables (Supabase CSV import or `INSERT`).

---

## 24. Recreation Master Prompt

> **MASTER PROMPT — Recreate the D'Blossom Model Private Schools platform exactly.**
>
> Build a responsive React + Vite + Tailwind CSS single-page application for a Nigerian private school called "D'Blossom Model Private Schools" (Divine Blossom Model School), Abeokuta, Ogun State. Motto: "Indomino Confidimus (In God We Trust)". Use React 18, react-router-dom v6, @tanstack/react-query v5, framer-motion, lucide-react, sonner for toasts, moment for dates, jspdf + html2canvas for PDF. Use a Supabase backend (Postgres + Storage) with the schema below. Use shadcn/ui-style components (Radix UI primitives) with the exact design tokens below.
>
> **Design tokens (Tailwind + CSS vars):** primary = deep navy `hsl(220 70% 25%)`, gold = `hsl(43 85% 55%)`, accent = red `hsl(0 75% 50%)`, background white, foreground `hsl(220 30% 15%)`, card white, secondary `hsl(220 15% 95%)`, muted `hsl(220 15% 96%)`, border `hsl(220 15% 90%)`, radius 0.625rem. Fonts: Playfair Display (headings) + Inter (body) via Google Fonts. Dark mode tokens defined but light is default.
>
> **Database tables (Supabase):** `students` (full_name, admission_number, class_name enum JSS1–SS3, gender, date_of_birth, parent_name, parent_phone, parent_email, boarding_status Day/Boarding, password plain-text, status Active/Inactive), `teachers` (full_name, staff_id, email, phone, subject, role Class Teacher/Teaching Staff, assigned_class JSS1–SS3 or empty, password, status), `results` (student_id, student_name, class_name, term First/Second/Third, session string, subjects jsonb array of {name, ca_score 0–30, exam_score 0–70, total, grade}, total_score, average, overall_percentage 0–100, position, teacher_comment, principal_comment), `payments` (student_name, class_name, amount, status Pending/Confirmed/Rejected, payment_date), `events` (title, description, date, image_url, category Announcement/Activity/Exam/Holiday/Other, status Published/Draft), `gallery_images` (title, image_url, category Campus/Events/Sports/Academics/Activities/Other), `complaints` (sender_name, student_name, email, message, status New/Read/Resolved). All with id, created_at, updated_at.
>
> **Pages/routes:** `/` Home (hero with school image + 4 CTAs, about preview, 6 "why choose us" cards, 3 latest published events, testimonial carousel), `/about` (story, mission, vision, proprietor Mr. Akapo Philip), `/academics` (4 level cards: Preschool, Basic, JSS, SS), `/gallery` (category filter + lightbox grid), `/events` (published events list), `/payment` (bank details Wema Bank / D'Blossom Private School / 0250842056 with copy button + payment form → creates Pending payment → success screen with WhatsApp link +2347025065593), `/complaint` (contact info + form → creates complaint + sends email), `/student-portal` (login by admission_number+password, find matching student by password among duplicates, view results by term+session, download PDF report card with multi-page A4 pagination), `/teacher-portal` (login by staff_id+password, Class Teacher sees result entry dashboard for assigned_class students, Teaching Staff sees view-only message; result entry: subject table with CA 0–30 + Exam 0–70 auto total+grade, add/remove subject, overall percentage, teacher comment, save/update), `/admin-login` (username "DivineBlossom" + password, default "DBMS", stored in localStorage, persists login in localStorage.admin_logged_in), `/admin-dashboard` (9 tabs: Students, Teachers, Results, Payments, Events, Gallery, Complaints, Subjects, Settings; sidebar desktop + bottom bar mobile; full CRUD with react-query, red validation errors, sonner toasts, confirm dialogs, image upload to Supabase Storage). 404 catch-all.
>
> **Business rules:** public app (no auth to browse); admin login via localStorage; teacher/student login by entity lookup (Active only); only Class Teachers edit results; one result per student+term+session; grade A≥70 B≥60 C≥50 D≥40 F<40; average = round(mean×100)/100; overall_percentage manual; subjects list customisable in localStorage (default 18 subjects); payments Pending→Confirmed/Rejected; complaints New→Read→Resolved; only Published events show publicly; PDF report cards paginate across A4 pages; every form shows red errors; every portal/auth page has "Back to Home".
>
> **Components:** Layout (Navbar+Outlet+Footer), fixed blur Navbar with hamburger, 4-column Footer with WhatsApp button, HeroSection, AboutPreview, WhyChooseUs, EventsPreview, TestimonialsSection (carousel), ResultView (report card + PDF), and 9 admin Manage* components + AdminSettings. shadcn/ui primitives for Button, Input, Label, Textarea, Select, Dialog, Table, Badge, etc.
>
> **Constants:** SCHOOL_NAME "D'Blossom Model Private Schools", MOTTO "Indomino Confidimus (In God We Trust)", ADDRESS "Divine Blossom Street, Agbogun Dam View, Obada Oko, Abeokuta, Ogun State, Nigeria", EMAIL "divineblossom999@gmail.com", EMAIL2 "sabbatah77@gmail.com", PHONES ["08123447514","08037151183"], WHATSAPP "+2347025065593", BANK_NAME "Wema Bank", ACCOUNT_NAME "D'Blossom Private School", ACCOUNT_NUMBER "0250842056", ESTABLISHED "September 5th, 2017", ADMIN_USERNAME "DivineBlossom", ADMIN_DEFAULT_PASSWORD "DBMS", CLASSES ["JSS1","JSS2","JSS3","SS1","SS2","SS3"], TERMS ["First Term","Second Term","Third Term"], SUBJECTS (18 subjects), TESTIMONIALS (6 entries from parents/students).
>
> Recreate every page, every workflow, every design element, and every business rule exactly as described. Use Supabase for data + storage + (edge function for) email. Deploy frontend to Netlify. Preserve all animations, responsive behavior, and the navy/gold/red color scheme.