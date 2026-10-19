# Course enrolment advisor: five data models

Week 7 design for the enrolment decision. One program decides whether a student may enrol in a section, and which courses match a free-text interest. This file is the data model only.

PostgreSQL is the system of record for students, courses, sections, enrolments, grades, and the decision. MongoDB is the system of record for course descriptions. Redis, Neo4j, pgvector, and Qdrant are derived from those two. Each derived record keeps the source id and the time it was refreshed (`refreshed_at`).

## PostgreSQL

**Question:** is this student allowed to take this section, and is a seat free?

| Entity | Fields the decision uses |
| --- | --- |
| Student | `student_id` |
| Course | `course_id`, `code` |
| Section | `section_id`, `course_id`, `capacity`, `seats_taken` |
| Enrolment | `student_id`, `section_id` |
| Grade | `student_id`, `course_id`, `grade` |

- `UNIQUE (student_id, section_id)` — one enrolment per student per section.
- `CHECK (seats_taken <= capacity)` — the seat count cannot pass capacity.

A free seat is `seats_taken < capacity` on the locked section row. The recorded `grade` is the result later copied as completion in the graph.

## MongoDB

**Question:** what does this course contain?

One course document. `course_id` is the PostgreSQL course id. Fields that are not true of every course are left out.

- `course_id`
- `title`
- `description`
- `syllabus`
- `semester`
- `credits`
- `assessment` — included when the course assesses that way. `lab` and `exam` are the parts the lab-and-exam query looks for
- `projects` — only on courses that have a project
- `reading_list` — only on courses that publish one

The query this shape answers is courses that have both a lab and an exam.

## Neo4j

**Question:** are prerequisites done, or covered by an equivalent?

```
(:Course)-[:REQUIRES]->(:Course)
(:Course)-[:EQUIVALENT]->(:Course)
(:Student)-[:COMPLETED]->(:Course)
```

- `Course` carries `course_id` (the PostgreSQL source id) and `refreshed_at`.
- `Student` carries `student_id`.
- `COMPLETED` is copied from a recorded grade and carries `refreshed_at`.

The same edges answer which courses have their prerequisites completed, and the shortest prerequisite path to a target course. An equivalent can stand in for a prerequisite.

## Redis

**Question:** what is this student currently planning?

- Key: `plan:{student_id}`
- Value: the course codes in the current plan. Each code is stored with its `course_id` and `refreshed_at`.
- TTL: the plan survives a reload and expires after inactivity.

A plan is a session. It is not an enrolment.

## pgvector and Qdrant

**Question:** which courses are close to this student's interests?

Both indexes store the same normalised embedding of the MongoDB description, model `all-MiniLM-L6-v2`, cosine distance, text id = `course_id`.

Payload on both:

- `course_id`
- `semester` (the search filter)
- `title`
- `credits`
- `refreshed_at`

pgvector keeps that row beside the business tables, so one SQL statement can join nearest courses to courses and sections. Qdrant stores the same point, with a payload index on `semester`.

## Who owns which fact

| Fact | Source of truth | What other stores keep |
| --- | --- | --- |
| Who the student is; course `code`; section `capacity` and `seats_taken`; the enrolment; the `grade` | PostgreSQL | Neo4j copies `student_id`, `course_id`, and completion. Redis copies course codes into the plan. |
| `title`, `description`, `syllabus`, `semester`, `credits`, and assessment (`lab`, `exam`, `projects`, `reading_list`) | MongoDB | pgvector and Qdrant copy `semester`, `title`, and `credits`, and embed the description. |
| Prerequisite, equivalent, and the completed-course walk | Neo4j | The result that creates `COMPLETED` is the PostgreSQL `grade`. The graph does not own the grade. |
| Courses the student is currently planning | Redis | A plan entry is not an enrolment and is not a catalogue row. |
| The interest-search embedding | Neither vector store | pgvector and Qdrant are two derived copies of one MongoDB description, kept so the indexes can be compared. |

## Not in this step

200–1,000 records per store, Docker Compose, and tests are later steps. This file does not implement them.
