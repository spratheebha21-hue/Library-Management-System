# Data Dictionary

This document describes every table and column in the Library Management System database
(`library_db`, PostgreSQL). It reflects the schema created by the Alembic migration in
`backend/alembic/versions/`.

---

## 1. `categories`

Normalized lookup table for book categories/genres, referenced by `books.category_id`.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `category_id` | `INTEGER` | **PK**, auto-increment | Unique identifier for the category. |
| `category_name` | `VARCHAR(100)` | **UNIQUE**, **NOT NULL**, indexed | Human-readable category name (e.g. "Computer Science", "Fiction"). |

---

## 2. `books`

Master catalogue of every title held by the library.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `book_id` | `INTEGER` | **PK**, auto-increment | Unique identifier for the book record. |
| `isbn` | `VARCHAR(20)` | **UNIQUE**, **NOT NULL**, indexed | 10 or 13 digit ISBN. Validated at the API layer with a regex (digits, optional trailing `X` for ISBN-10). |
| `title` | `VARCHAR(255)` | **NOT NULL**, indexed | Book title. Indexed (and part of composite index) for search/sort. |
| `author` | `VARCHAR(255)` | **NOT NULL**, indexed | Primary author. Indexed (and part of composite index) for search/sort. |
| `publisher` | `VARCHAR(255)` | nullable | Publishing house. |
| `category_id` | `INTEGER` | **FK** → `categories.category_id`, **NOT NULL**, `ON DELETE RESTRICT`, indexed | Category the book belongs to. |
| `total_copies` | `INTEGER` | **NOT NULL**, **CHECK** (`total_copies >= 0`) | Total physical copies owned by the library. |
| `available_copies` | `INTEGER` | **NOT NULL**, **CHECK** (`available_copies >= 0` AND `available_copies <= total_copies`) | Copies currently on the shelf (not issued). Decremented on issue, incremented on return. |
| `shelf_location` | `VARCHAR(50)` | nullable | Physical shelf/rack code (e.g. `A1-01`). |
| `added_date` | `DATE` | **NOT NULL** | Date the title was added to the catalogue. |

**Composite index:** `ix_books_title_author (title, author)` — speeds up combined search/sort/filter queries.

---

## 3. `members`

Library members: students, faculty, and staff who can borrow books.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `member_id` | `INTEGER` | **PK**, auto-increment | Unique identifier for the member. |
| `name` | `VARCHAR(255)` | **NOT NULL**, indexed | Full name of the member. |
| `email` | `VARCHAR(255)` | **UNIQUE**, **NOT NULL**, indexed | Contact email; validated as a proper email format at the API layer. |
| `phone` | `VARCHAR(20)` | nullable | Contact phone number. |
| `member_type` | `ENUM('Student','Faculty','Staff')` | **NOT NULL** | Category of member, drives borrowing policy in a full implementation. |
| `department` | `VARCHAR(100)` | nullable | Academic/administrative department. |
| `join_date` | `DATE` | **NOT NULL** | Date the member registered with the library. |
| `status` | `ENUM('Active','Inactive')` | **NOT NULL**, default `Active` | Whether the member is currently allowed to borrow books. |

---

## 4. `staff`

Librarians/staff who process issue and return transactions (distinct from library *members*).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `staff_id` | `INTEGER` | **PK**, auto-increment | Unique identifier for the staff/librarian. |
| `name` | `VARCHAR(255)` | **NOT NULL** | Full name. |
| `email` | `VARCHAR(255)` | **UNIQUE**, **NOT NULL**, indexed | Staff contact email. |
| `role` | `VARCHAR(100)` | **NOT NULL**, default `Librarian` | Job title / role (e.g. "Head Librarian", "Assistant Librarian"). |
| `join_date` | `DATE` | **NOT NULL** | Date staff joined the library. |

---

## 5. `book_issues`

Transaction log — the associative table connecting `books`, `members`, and `staff`. Each row is one
issue/borrow event, including its due date, optional return date, and any fine incurred.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `issue_id` | `INTEGER` | **PK**, auto-increment | Unique identifier for the transaction. |
| `book_id` | `INTEGER` | **FK** → `books.book_id`, **NOT NULL**, `ON DELETE RESTRICT`, indexed | The book being issued. |
| `member_id` | `INTEGER` | **FK** → `members.member_id`, **NOT NULL**, `ON DELETE RESTRICT`, indexed | The member borrowing the book. |
| `staff_id` | `INTEGER` | **FK** → `staff.staff_id`, nullable, `ON DELETE SET NULL`, indexed | The librarian who processed the transaction. |
| `issue_date` | `DATE` | **NOT NULL** | Date the book was issued. |
| `due_date` | `DATE` | **NOT NULL**, **CHECK** (`due_date > issue_date`) | Date the book is due back. |
| `return_date` | `DATE` | nullable, **CHECK** (`return_date IS NULL OR return_date >= issue_date`) | Date the book was actually returned; `NULL` while still on loan. |
| `fine_amount` | `NUMERIC(10,2)` | **NOT NULL**, default `0.00`, **CHECK** (`fine_amount >= 0`) | Fine owed for late return (auto-calculated by the API at ₹5/day late if not supplied manually). |
| `status` | `ENUM('Issued','Returned','Overdue')` | **NOT NULL**, default `Issued` | Current transaction state. The API recalculates `Issued → Overdue` automatically once `due_date` has passed and the book has not been returned. |

**Composite index:** `ix_issues_member_status (member_id, status)` — speeds up "show all overdue books for
member X" style queries used heavily on the Issues grid page.

---

## Enumerations reference

| Enum type | Values | Used by |
|---|---|---|
| `member_type_enum` | `Student`, `Faculty`, `Staff` | `members.member_type` |
| `member_status_enum` | `Active`, `Inactive` | `members.status` |
| `issue_status_enum` | `Issued`, `Returned`, `Overdue` | `book_issues.status` |

---

## Referential integrity summary

| Child table.column | Parent table.column | On Delete | Justification |
|---|---|---|---|
| `books.category_id` | `categories.category_id` | RESTRICT | A category in use by books can't be deleted until books are reassigned. |
| `book_issues.book_id` | `books.book_id` | RESTRICT | Preserves audit trail; a book with transaction history cannot be deleted. |
| `book_issues.member_id` | `members.member_id` | RESTRICT | Preserves audit trail; deactivate (`status = Inactive`) instead of deleting members with history. |
| `book_issues.staff_id` | `staff.staff_id` | SET NULL | Staff turnover shouldn't block deletion or destroy transaction history — the link is simply cleared. |
