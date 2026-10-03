# Entity-Relationship Diagram

The diagram below is written in [Mermaid](https://mermaid.js.org/) syntax. GitHub, GitLab, and most modern
Markdown viewers (including VS Code with the Mermaid extension) render this automatically. You can also
paste the code block into the [Mermaid Live Editor](https://mermaid.live) to export a PNG/SVG image.

```mermaid
erDiagram
    CATEGORIES {
        int category_id PK
        string category_name "UNIQUE, NOT NULL"
    }

    BOOKS {
        int book_id PK
        string isbn "UNIQUE, NOT NULL"
        string title "NOT NULL, INDEXED"
        string author "NOT NULL, INDEXED"
        string publisher
        int category_id FK
        int total_copies "CHECK >= 0"
        int available_copies "CHECK >= 0 AND <= total_copies"
        string shelf_location
        date added_date "NOT NULL"
    }

    MEMBERS {
        int member_id PK
        string name "NOT NULL, INDEXED"
        string email "UNIQUE, NOT NULL"
        string phone
        enum member_type "Student | Faculty | Staff"
        string department
        date join_date "NOT NULL"
        enum status "Active | Inactive"
    }

    STAFF {
        int staff_id PK
        string name "NOT NULL"
        string email "UNIQUE, NOT NULL"
        string role "NOT NULL"
        date join_date "NOT NULL"
    }

    BOOK_ISSUES {
        int issue_id PK
        int book_id FK
        int member_id FK
        int staff_id FK "nullable"
        date issue_date "NOT NULL"
        date due_date "CHECK > issue_date"
        date return_date "nullable, CHECK >= issue_date"
        decimal fine_amount "CHECK >= 0"
        enum status "Issued | Returned | Overdue"
    }

    CATEGORIES ||--o{ BOOKS : "categorizes (1:N)"
    BOOKS ||--o{ BOOK_ISSUES : "is issued via (1:N)"
    MEMBERS ||--o{ BOOK_ISSUES : "borrows (1:N)"
    STAFF ||--o{ BOOK_ISSUES : "processes (1:N)"
```

## Cardinality summary

| Relationship | Type | Notes |
|---|---|---|
| Category → Books | 1 : N | One category groups many books. A book belongs to exactly one category. |
| Book → BookIssues | 1 : N | One book can be issued many times over its lifetime (across different transactions), but only while `available_copies > 0`. |
| Member → BookIssues | 1 : N | One member can have many issue/borrow transactions. |
| Staff → BookIssues | 1 : N | One staff member can process many transactions. `staff_id` is nullable (transaction may be unattributed / staff record later removed). |

Books and Members have an **indirect N:N relationship** realized through the `BOOK_ISSUES` associative/junction
table (a member can borrow many books over time, and a book can be borrowed by many members over time — just
not concurrently, enforced by the `available_copies` counter).

## ON DELETE strategy rationale

| FK | ON DELETE | Reason |
|---|---|---|
| `books.category_id → categories.category_id` | `RESTRICT` | Prevents deleting a category that still has books referencing it, avoiding orphaned/NULL categories. Admin must reassign books first. |
| `book_issues.book_id → books.book_id` | `RESTRICT` | A book with transaction history (even old, returned ones) must not be deleted — required for audit trail and fine history. |
| `book_issues.member_id → members.member_id` | `RESTRICT` | Preserves borrowing history for a member even if they are otherwise deactivated (`status = Inactive`) rather than hard-deleted. |
| `book_issues.staff_id → staff.staff_id` | `SET NULL` | If a librarian leaves and their staff record is removed, historical transactions they processed remain (with `staff_id = NULL`) rather than being blocked or deleted. |
