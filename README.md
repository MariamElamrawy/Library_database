# Library Database System

A relational database for a public library, built in SQLite, with a Python command-line app on top that members use to search the catalogue, borrow and return items, place holds, donate items, and sign up for events.

The interesting part is the data model. Business rules like "you can't borrow if you owe more than $20" or "you can't place a hold on something that's sitting on the shelf" are enforced inside the database itself with constraints and triggers, so bad data gets rejected no matter what code is talking to it.

## My contributions

Built with a teammate name for CMPT 354 (Database Systems I) at Simon Fraser University.

**My part:** I designed and built the database: the schema, all constraints and triggers, data validation, and the sample dataset, all in `librarydata.ipynb` and detailed 'Anomolies.pdf' and 'Requirments.pdf'

## Data model

18 tables covering the catalogue, physical copies, members, loans, fines, holds, events, rooms, staff, and donations.
### Design decisions

**Item subtypes.** Every item shares common fields (title, creator, publisher, year, genre, language), but a print book needs an ISBN and page count while a journal needs an ISSN, issue, and volume. Instead of one wide table full of NULLs, `Item` holds the shared fields and each subtype gets its own table keyed on `item_id`.

**Items vs. copies.** A title and a physical copy are different things. The library might own three copies of *Pride and Prejudice*, and each copy has its own status. `Copy` uses a composite key `(item_id, copy_number)`, and loans point at a specific copy, not just a title.

**Donations at the copy level.** Each donated copy has exactly one donor, so `Donates` is keyed on `(item_id, copy_number)`. Donating a book the library already owns adds a new copy instead of a duplicate catalogue entry.

**Fines keyed on loans.** A fine belongs to exactly one loan, so `Fine` uses `loan_id` as its primary key. That makes duplicate fines for the same loan impossible.

## Data validation

**CHECK constraints** restrict status fields to valid values:

| Table | Allowed values |
|---|---|
| `Copy.status` | available, on loan, on hold, lost, in repair, reference-only |
| `Member.status` | active, expired, suspended |
| `Fine.status` | unpaid, paid |

**UNIQUE constraints** prevent duplicate ISBNs and duplicate member emails. **Foreign keys** are enforced on every relationship.

**Triggers** keep the data consistent automatically:

| Trigger | What it does |
|---|---|
| `copy_on_loan` | Marks a copy as "on loan" the moment a loan is created |
| `copy_returned` | Sets the copy back to "available" when a return date is recorded |
| `block_loan_if_fines_exceed` | Rejects a new loan if the member has more than $20 in unpaid fines |
| `block_hold_if_available` | Rejects a hold if a copy of that item is already available |

## The app

`LibraryDBApp.py` lets a member:

- Log in or register
- Search by title, author, ISBN/ISSN, or year
- Borrow an available copy, or place a hold if none are free
- Return items, with late fines calculated automatically ($0.50 per day)
- Donate items
- Browse upcoming events and register or volunteer
- Contact library staff

All queries are parameterized, and multi-step changes (like creating a loan and clearing the member's hold) run inside a single transaction.

## Running it

Requires Python 3 and Jupyter.

```bash
pip install jupysql
```

1. Open `librarydata.ipynb` and run all cells. This creates `library.db` with the schema, triggers, and sample data. (If `library.db` already exists, delete it first.)
2. Run the app:

```bash
python LibraryDBApp.py
```

3. Log in with any member ID from 1 to 10, or register a new member.

All data in the database is fictional.
