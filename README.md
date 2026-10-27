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