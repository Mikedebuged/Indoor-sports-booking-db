# Build roadmap

This file splits the build of the Indoor Sports Facility Booking System into ten parts (0 to 9). Each part adds a few files, has a test you can run, and ends with one commit. The design comes from our mid-term report: its tables, constraints, assumptions, trigger rules 1 to 6, indexes and planned views. If this file and the report ever disagree, the report wins.

## How every part ends

1. Run `install.sql`. It drops everything and rebuilds from scratch, so you find out straight away if the scripts stop working in order.
2. Run the part's test script and check that the results match what the part says to expect.
3. Tick the part in README.md, then commit:

```
git status
git add <files for this part> README.md install.sql
git commit -m "<message given in the part>"
git push
```

If a test fails, fix it before you commit. A commit should always leave the repo in a state that builds.

## Run order

`install.sql` runs these files in this order. Each part adds its own line.

| Order | File | Added in |
|---|---|---|
| 1 | sql/00_drop_all.sql | Part 1 |
| 2 | sql/01_parent_tables.sql | Part 1 |
| 3 | sql/02_child_tables.sql | Part 2 |
| 4 | sql/03_indexes.sql | Part 3 |
| 5 | sql/04_triggers_row.sql | Part 4 |
| 6 | sql/05_triggers_compound.sql | Part 5 |
| 7 | sql/06_sample_data.sql | Part 6 |
| 8 | sql/07_views.sql | Part 7 |

Tests live in `tests/` and run separately, after `install.sql`. Each test script ends with `ROLLBACK`, so testing never leaves extra rows behind.

## Part 0: Tools and repository

Get a working Oracle connection and put this repository on GitHub.

- Oracle: the course uses FreeSQL (freesql.com). Sign in, click Connect at the top right, then click Regenerate to get a password. Note the hostname, port, username and service name the dialog shows. If you regenerate the password later, update it in VS Code too.
- SQL tool: install the SQL Developer extension in VS Code and add a connection with those FreeSQL details. Run the repo's scripts from VS Code, because `@install.sql` needs a tool that can read the files in this folder. The FreeSQL browser worksheet can't read your files, so use it for quick queries.
- GitHub: create a private repo named `indoor-sports-booking-db` with no README or .gitignore (this folder already has both), then connect this folder and push.

You learn: in Oracle a user and a schema are the same thing; connecting with host, port and service name; `git init`, `add`, `commit`, `push`.

Done when: `SELECT USER, SYSDATE FROM dual;` returns your username, `SELECT * FROM product_component_version;` shows version 12 or later, and GitHub shows README.md, ROADMAP.md, .gitignore and docs/er-diagram.png.

Commit: `Set up repository with README, roadmap and ER diagram`

## Part 1: Parent tables and the drop script

Files: `sql/00_drop_all.sql`, `sql/01_parent_tables.sql`, `install.sql`, `tests/t01_parent_tables.sql`

Tables: Membership_Plan, Customer, Sport, Court, then Equipment (last, because it points to Sport).

You learn:
- The Oracle types from Section 4 of the report: NUMBER(p,s), VARCHAR2, DATE and GENERATED ALWAYS AS IDENTITY
- PRIMARY KEY, UNIQUE, NOT NULL, CHECK and DEFAULT, and why each status column is NOT NULL as well as checked (a CHECK lets NULL through)
- Naming constraints (`pk_customer`, `uq_customer_email`, `ck_court_type`) so an error message tells you which rule broke
- Why the drop script removes child tables before parents, and how to make it safe on an empty schema (Oracle 12c has no `DROP TABLE IF EXISTS`)

Done when: valid rows insert, and each of these fails: a malformed email, a duplicate phone, a court type of 'Grass', a negative hourly rate, a 120% discount, a customer with no last name. Expect ORA-02290 (check constraint), ORA-00001 (unique) and ORA-01400 (NULL into NOT NULL).

Commit: `Add parent tables, drop script and install script`

## Part 2: Child tables

Files: `sql/02_child_tables.sql`, `tests/t02_child_tables.sql`

Tables: Membership, Court_Sport, Booking, Equipment_Rental, Payment.

You learn:
- Foreign keys and Oracle's default delete rule, which blocks deleting a parent row that is in use (assumption 6)
- Composite keys: the primary keys of Court_Sport and Equipment_Rental, and Booking's (court_id, sport_id) foreign key to Court_Sport
- Virtual columns: booking_date and duration_hours
- CHECK constraints that compare two columns (end_time after start_time, a booking stays within one day)

Done when: booking soccer on a court that only hosts badminton fails with ORA-02291, deleting a court that has a booking fails with ORA-02292, and a 90-minute booking shows duration_hours = 1.5. Until Part 4 you type court_fee, fee_charged and end_date by hand, because the triggers that calculate them come later.

Commit: `Add child tables with composite and virtual columns`

## Part 3: Indexes

Files: `sql/03_indexes.sql` (the seven indexes in Section 4)

You learn: which indexes Oracle creates for you (primary key and UNIQUE) and which it leaves to you (foreign keys); reading `user_indexes` and `user_ind_columns`; a first look at `EXPLAIN PLAN`.

Done when: a query on `user_ind_columns` lists all seven indexes with the right columns. On a nearly empty table the optimizer may still pick a full scan. Check the plan again after Part 6 loads data.

Commit: `Add foreign key and report indexes`

## Part 4: Row-level triggers (rules 6, 4, 2 and 3)

Files: `sql/04_triggers_row.sql`, `tests/t04_row_triggers.sql`

Write them from simplest to hardest: rule 6 (membership end date), rule 4 (rental fee), rule 2 (no bookings on closed courts), rule 3 (court fee with membership discount).

You learn:
- PL/SQL block structure, BEFORE INSERT and BEFORE UPDATE OF triggers, `:NEW` and `:OLD`
- `SELECT ... INTO`, and getting "highest discount, or 0 if none" with `NVL(MAX(...), 0)`
- `RAISE_APPLICATION_ERROR` with your own error numbers (-20001 to -20999)
- Finding compile errors with `SHOW ERRORS` and `user_errors`

Done when: a new membership gets end_date = start_date plus the plan length; renting 3 rackets at $5 stores 15; a booking on a court under Maintenance is rejected with your message; a 2-hour booking at $40 an hour stores 80 for a non-member, 64 for a member with a 20% plan, and uses the higher discount when a customer holds two active memberships.

Commit: `Add row-level triggers for fees, membership end dates and closed courts`

## Part 5: Compound triggers (rules 1 and 5)

This is the hardest part, so give it the most time.

Files: `sql/05_triggers_compound.sql`, `tests/t05_compound_triggers.sql`

You learn:
- The mutating-table error, ORA-04091. Write rule 1 as a plain row trigger first, watch it fail on an UPDATE, then rewrite it as a compound trigger.
- Compound trigger sections: note the affected rows in the row-level section and check them in AFTER STATEMENT
- The overlap test: two bookings overlap when each starts before the other ends
- Locking the court row with `SELECT ... FOR UPDATE` so two sessions can't book the same slot at once
- For rule 5, why adding up every overlapping rental can reject a valid booking. Usage peaks at the start of some booking, so check the total at each start time that falls inside the booking.

Done when: 18:00 to 19:00 and then 19:00 to 20:00 on the same court both succeed; 18:30 to 19:30 fails; cancelling the first booking frees its slot; reinstating a cancelled booking that now clashes fails; with 4 rackets owned, renting 3 and 2 on overlapping bookings fails, while the same rentals on back-to-back bookings succeed. Optional: open two sessions (VS Code and the FreeSQL worksheet count as two) and book the same slot in both to watch the lock make the second one wait.

Commit: `Add compound triggers for double booking and equipment stock`

## Part 6: Sample data

Files: `sql/06_sample_data.sql`

You learn:
- Inserting into identity tables: you never type an ID, you look it up by name with a subquery
- Writing dates and times: `DATE '2026-11-14'` and `TO_DATE('2026-11-14 18:00', 'YYYY-MM-DD HH24:MI')`
- Loading in dependency order and letting the triggers fill in court_fee, fee_charged and end_date
- Designing data so every report has a pattern to show: bookings spread over several months, weekdays and hours, some cancellations, unpaid and part-paid bookings, a refund, members and non-members

Rough size: 20 customers, 3 plans, 10 memberships, 5 sports, 6 courts, 8 equipment items, 80 to 100 bookings, with rentals and payments to match.

Done when: the whole file loads with no errors, row counts per table look right, and three bookings you check by hand have the court_fee the formula gives.

Commit: `Add sample data`

## Part 7: Report views

Files: `sql/07_views.sql` and `queries/court_availability.sql`

Views: vw_sport_popularity, vw_peak_hours, vw_top_customers, vw_monthly_revenue, vw_equipment_demand, vw_booking_balance. The free-courts question takes a date, time and sport as inputs, so it is a query with bind variables instead of a view.

You learn:
- GROUP BY, HAVING, ranking, and TO_CHAR for weekday and hour
- Outer joins, so a booking with no payments still appears in the balance view
- The double-counting trap: joining a booking to its rentals and its payments in one step multiplies the sums. Total each one in its own subquery, then join. Interviewers ask about this.
- NOT EXISTS with the overlap test, for availability

Done when: you work out one booking's balance by hand (court fee plus rentals minus Paid payments) and vw_booking_balance agrees, and Pending and Refunded payments are left out of vw_monthly_revenue.

Commit: `Add management report views and court availability query`

## Part 8: Break-it tests

Files: `tests/test_business_rules.sql`, `docs/test_results.md`

Step 5 of the report's plan: try to break every rule and record what Oracle says. Double-book a court, rent more rackets than the centre owns, book a court under maintenance and enter an invalid status, then add one attempt against each CHECK, UNIQUE, foreign key and trigger.

You learn: negative testing, `SPOOL` to save output to a file, `SET ECHO ON` so the log shows each statement next to its error.

Done when: every attempt fails with the error you expected, every valid control case succeeds, and `docs/test_results.md` lists the rule, the attempt, the expected result and the actual Oracle error.

Commit: `Add business rule tests and recorded results`

## Part 9: One-step rebuild and finish

Files: final `install.sql` (with `WHENEVER SQLERROR EXIT` and a spooled log) and the finished README (what the system does, the ER diagram, how to run it, design decisions, what the tests prove).

You learn: why a rebuild script should stop at the first error, and how to tag a release in git.

Done when: you run `install.sql` twice in a row and both runs finish cleanly, with the test suite passing after each.

Commit: `Finish one-step install and README`, then `git tag v1.0` and `git push --tags`.

## Before the repo goes public

Keep it private while the course runs. The mid-term report lists all four students' IDs, so it stays out of the repo; `.gitignore` blocks .docx files. Once grades are final and the group agrees, make the repo public and pin it on your GitHub profile.
