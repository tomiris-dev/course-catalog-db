# course-catalog-db

## Constraint violation log

Commands were run with `\set VERBOSITY verbose`.

| # | What we violate | SQLSTATE | Constraint | API response |
|---|-----------------|----------|------------|--------------|
| 1 | NULL in courses.title | 23502 | NOT NULL (title)| 422 |
| 2 | courses.credits = 0 | 23514 | courses_credits_check | 422 |
| 3 | courses.likes = -1 | 23514 | courses_likes_check | 500  |
| 4 | a one-letter student name | 23514 | students_name_check |422 |
| 5 | a duplicate student email | 23505 | students_email_key |  409 Conflict |
| 6 | a duplicate course id | 23505 | courses_pkey | 409 Conflict |
| 7 | a repeated enrollment in a course | 23505 | enrollments_course_id_student_id_key | 409 Conflict |
| 8 | enrolling in a nonexistent course | 23503 | enrollments_course_id_fkey | 404 |
| 9 | enrolling a nonexistent student | 23503 | enrollments_student_id_fkey | 404 |
| 10 | deleting a course that has enrollments | 23001 | enrollments_course_id_fkey | 409 Conflict |
| 11 | deleting a student (cascade) | - | ON DELETE CASCADE | 200 OK |

## Pydantic question
Pydantic already checks the name length (min_length=2) in our API. Why duplicate this in the database? The database is the last line of defense. Pydantic only protects our API, but if someone else (such as another service, a script, or an administrator) decides to write data directly to the database, Pydantic will no longer work. Constraints within the database itself ensure that no one will ever be able to violate the rules, regardless of how the data gets there.

## Index: EXPLAIN before and after

Query: `SELECT * FROM enrollments WHERE student_id = 777;`
Database idx_demo: 500 courses, 20,000 students, 50,000 enrollments.

| | Before the index | After the index |
|---|---|---|
| Access method | Seq Scan | Bitmap Index Scan |
| cost | 0.00..1042.00 | 4.31..15.59 |
| Rows Removed by Filter | 49997 | - |
| Execution Time | 1.949 ms | 0.176 ms |

**Plans:**

```sql
-- Before the index
Seq Scan on enrollments  (cost=0.00..1042.00 rows=3 width=34) (actual time=0.057..1.934 rows=3.00 loops=1)
  Filter: (student_id = 777)
  Rows Removed by Filter: 49997
  Buffers: shared hit=417
Planning Time: 0.082 ms
Execution Time: 1.949 ms
-- After the index
Bitmap Heap Scan on enrollments  (cost=4.31..15.59 rows=3 width=34) (actual time=0.149..0.152 rows=3.00 loops=1)
  Recheck Cond: (student_id = 777)
  Heap Blocks: exact=3
  Buffers: shared hit=6 read=2
  ->  Bitmap Index Scan on enrollments_student_id_idx  (cost=0.00..4.31 rows=3 width=0) (actual time=0.135..0.135 rows=3.00 loops=1)
        Index Cond: (student_id = 777)
        Index Searches: 1
        Buffers: shared hit=3 read=2
Planning Time: 0.136 ms
Execution Time: 0.176 ms
```
**Answers to the questions:**
 
1. TODO: why didn't PostgreSQL create an index on `enrollments.student_id`,
   although there are indexes on `email` and `course_id`?
   PostgreSQL creates indexes automatically only for PRIMARY KEY and UNIQUE constraints. Foreign keys like student_id do not get automatic indexes because they do not guarantee uniqueness. The course_id has an index because it is the first column in the UNIQUE constraint for the pair of course_id and student_id.
2. TODO: why does a primary-key lookup on `courses` (six rows) use `Seq Scan`,
   while on `enrollments` (50 thousand rows) it goes through an index?
   The query planner compares the cost of different ways to get data. The courses table takes just one memory page because it has only six rows. Reading the whole table is faster and cheaper than using an index. For the enrollments table with fifty thousand rows reading the whole table is too expensive so the planner uses an index.
3. TODO: name two operations that an index on `student_id` will speed up.
    First it speeds up finding all enrollments for a specific student. Second it speeds up deleting a student because the database uses the ON DELETE CASCADE rule to quickly find and remove all records of that student in the enrollments table.
