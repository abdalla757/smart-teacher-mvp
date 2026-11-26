# Smart Teacher Platform (منصة المدرس الذكي) - MVP

A graduation project by Arab Automators. The name is a working name (the brief calls it "اسم مبدئي", a provisional name).

An automated system for a teacher who runs groups of 200 to 300 students: from registration to the ID card, attendance, exams, grades, parent communication and dashboards.

> **Status: design phase.** The design is done (this README is based on the project brief deck). Implementation is not committed to this repo yet. Nothing below describes shipped code.

## The problem

Who it serves: private tutoring centers and teachers in Egypt, starting in Ismailia. The brief describes a teacher with 500+ students in total and up to 300 in a single group.

| Area | Problem today | What the system does |
|---|---|---|
| Attendance | Calling names or a paper sheet for 300 students | The assistant scans the student's QR card at the classroom door |
| Absence | The parent finds out late, or never | A WhatsApp message goes to the parent 30 minutes after the session starts |
| Parent | Nobody checks that the registered number really belongs to the parent | Name match, then confirmation from the parent by WhatsApp |
| Exam | Printing, handing out and collecting papers with no clear link to the student | A random secret number per paper, linked to the student at entry |
| Grades | A grade book and a handwritten message to each parent | The grade is saved and reaches the parent at the same moment |
| Follow-up | No full picture of attendance and level | A teacher dashboard and an in-class display screen |

Goal in the brief's words: the student registers once, then everything runs with the card, and no manual work is needed after registration.

## Scope: five stages, built in order

The student journey has five stages. The brief's implementation plan builds them in this order, and each stage ends with something that works.

1. **Registration and ID card.** Registration form, parent verification, ID card generation. Done when a student registers, their status shows to the reviewer, and a printable card comes out. The card is a PDF at 54 x 85.6 mm (national ID card size) with the teacher's logo, colour and name, the student's photo, name, grade, group and a QR code. First version is QR; NFC is listed as later.
2. **Attendance.** Scanner page, scheduled absence message. Done when the assistant scans the card, sees one of three results, and the parent of an absent student gets a message after 30 minutes.
3. **Exams.** Word-to-PDF conversion with barcodes, paper-linking mode in the scanner. Done when a 200-page PDF with secret numbers comes out and each paper is linked to a student at entry.
4. **Grading and grades.** Grade entry screen, grade and congratulation messages. Done when the teacher scans the paper, types the grade, and the parent receives it on WhatsApp.
5. **Dashboards.** Admin dashboard, display screen, public link. Done when both show real data and the classroom screen updates on its own without sensitive data.

Development order inside the brief: build and test each stage on the Development environment first, then move it to Production, and test it against the next stage.

### Stage 1: registration, verification, card

Flow: registration form, save data and photo, check the parent's name, then generate the card. If the verification status is "mismatch / needs review", the parent gets a WhatsApp message with a button ("I confirm I am the student's parent"), and the card is issued after the confirmation.

- Trigger: form submitted (webhook).
- Steps: upload the photo and save data, create the code and QR token, status "needs review", the reviewer picks the status, on "match" or on parent confirmation generate the card PDF.
- Result: a print-ready card and a recorded verification status.
- Screens: registration form (10), card (11), confirmation message (13), verification review (16).

### Stage 2: attendance

- Trigger: scanner request (webhook), and a schedule for absences.
- Steps: look up the student by `qr_token`, compare their group with the session, look for an earlier scan, write the attendance record, return the status and photo to the scanner.
- The scanner returns one of three results immediately: **success** (attendance recorded), **duplicate** (already scanned this hour), **wrong group** (not in this group).
- 30 minutes after the session starts, a schedule builds the list of students who did not scan and sends each absent student's parent one WhatsApp message, once only (`absence_alert_sent_at`).
- Screens: scanner with its three states (12), absence message (13), display screen (17).
- The scanner is a lightweight web page the assistant opens on their phone. No app install.

### Stage 3: exams

- Trigger: exam upload form, then scanning the paper and then the student.
- Steps: save the file, generate 200 unique random numbers, convert Word to PDF, add a barcode to each page, merge into one file.
- Result: a 200-page PDF for printing, one paper per student, each with its own secret number.
- At entry: the student takes a paper, the assistant scans the paper barcode and then the student's QR card, and the system links the student to the secret number.
- The student does not write their name on the paper.
- Screens: exam paper (14), scanner in linking mode (12), exams in the admin dashboard (16).

### Stage 4: grading and grades

- Trigger: save the grade from the entry screen (webhook).
- Steps: save the grade, find the student from `secret_code`, record the send time, send the WhatsApp message.
- After grading is complete: rank the students and send a congratulation message for the top grade, (the brief mentions ranking and the top 10), and the display screen shows the exam top students.
- Screens: grade entry (15), grade and congratulation messages (13), exam top students (17).

### Stage 5: dashboards

- Trigger: a schedule every 30 seconds during the session.
- Steps: count who is present now, calculate the attendance rate, find the exam top students, find the last student scanned, apply the display settings.
- Admin dashboard: teacher-facing, with attendance, groups, sessions, exams and verification review.
- Display screen: shown in the classroom, shows the number present, attendance rate, recent arrivals, the student of the month and the exam top students. The teacher controls what appears (`display_settings`).
- Privacy rule: the display screen reads only from a public view (`public_display`) that holds name, photo, score and scan time, and no phone numbers.

## Key design ideas

- **Secret number links exam sheet to student.** Each page of the exam PDF gets a random, non-sequential secret code with a barcode. The student is linked to the code at the door, so the paper carries no name. Grading starts from the code and finds the student through `exam_papers`.
- **Parent confirms their own number.** Nobody checks today that the number on file is really the parent's. The system matches the parent's name and sends a WhatsApp confirmation request. The ID card is issued after confirmation. (`guardians.confirmed_at`, `students.verify_status`)
- **One card, used everywhere.** The card QR is shown at the door and again at exam entry. It is personal and not used by another student. If lost, the assistant asks for a new code.
- **Messages sent once.** The absence alert is recorded per session so a parent is never messaged twice.
- **Two environments.** Development: fake data, its own Supabase project and its own n8n copy. Production: a separate copy for the real teacher, not edited directly.

## Data model (Supabase / PostgreSQL, 9 tables)

A `->` means the column points to another table.

| Table | Holds |
|---|---|
| `teachers` | `id`, `name`, `subject`, `logo_url`, `brand_color`, `display_settings` (also `student_of_month_id`, which the dashboard stage reads) |
| `groups` | `id`, `teacher_id -> teachers`, `name`, `grade`, `weekday`, `start_time` |
| `sessions` | `id`, `group_id -> groups`, `starts_at`, `absence_alert_sent_at` |
| `guardians` | `id`, `full_name`, `relation`, `phone`, `confirmed_at` |
| `students` | `id`, `code`, `qr_token`, `group_id -> groups`, `guardian_id -> guardians`, `full_name`, `photo_url`, `verify_status` |
| `attendance` | `id`, `session_id -> sessions`, `student_id -> students`, `scanned_at`, `result` |
| `exams` | `id`, `teacher_id -> teachers`, `title`, `grade`, `copies`, `file_url`, `pdf_url` |
| `exam_papers` | `id`, `secret_code`, `exam_id -> exams`, `student_id -> students`, `page_no`, `linked_at` |
| `grades` | `id`, `paper_id -> exam_papers`, `student_id -> students`, `score`, `max_score`, `notified_at` |

Plus `public_display`, a view with only `full_name`, `photo_url`, `score`, `scanned_at`, used by the display screen.

Storage buckets: `photos/` (student photos), `cards/` (ID card PDFs), `exams/` (Word and PDF files).

Note: the brief's dashboard slide lists `student_of_month_id` under teachers, but the 9-table diagram does not show that column on `teachers`. Treat it as an open detail to settle when the schema is written.

## Architecture

```
Inputs                         Automation (n8n)           Outputs
- Registration form   --\                              /-- WhatsApp to the parent
- Scanner page        ---+--> webhooks --> n8n  <-->  +--- Card and exam PDFs
- Exam upload form    ---+                  |          \-- Admin dashboard
- Grade entry screen  --/               Supabase       \-- Display screen
                               (PostgreSQL, 9 tables + Storage)
```

- **Supabase**: PostgreSQL with the 9 tables, plus Storage for photos, exams and cards.
- **n8n**: runs all the automations as workflows, started by webhooks from the front-end pages or by schedules:

  | Workflow | Purpose |
  |---|---|
  | `wf-01 register` | Take the registration form |
  | `wf-02 verify` | Parent verification |
  | `wf-03 id-card` | Generate the card |
  | `wf-04 check-in` | Scan and record attendance |
  | `wf-05 absence` | Scheduled absence message |
  | `wf-06 exam-pdf` | Build the exam PDF |
  | `wf-07 link-paper` | Link a paper to a student |
  | `wf-08 grade` | Save the grade and notify |

- **Front-end pages** (inputs): registration form, scanner (a lightweight web page), exam upload form, grade entry screen. Outputs: card and exam PDFs, admin dashboard, display screen.
- **WhatsApp**: messages to parents (confirmation, absence, grade, final-grade congratulation).

The brief lists eight screens it will mock up (registration form, card front and back, scanner in three states, WhatsApp messages, barcode exam paper, grade entry, admin dashboard, display screen). All data in those mockups is fictional.

## Status and next steps

- Done: problem, scope, student journey, data model, workflows and screens are designed in the brief.
- Not done: no code, schema, workflows or pages are committed to this repo yet.
- Next: follow the plan above, starting with stage 1 (registration and ID card) on the Development environment.

## Open details

Items the brief does not settle and this README does not invent:

- Hosting and front-end technology for the web pages.
- WhatsApp delivery route and message-template approval.
- Exact values for the verification statuses and the attendance `result` field beyond the three scanner outcomes.
- Final project name.
