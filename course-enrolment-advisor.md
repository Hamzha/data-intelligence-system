# Course enrolment advisor

One program decides whether a student may enrol in a course, and suggests courses that match the student’s interest. The decision uses five stores. PostgreSQL is the system of record for the business entities and the enrolment decision. MongoDB is the system of record for course descriptions. Redis, the graph, and both vector indexes are derived from those sources. The same course-description embeddings are stored in pgvector and in Qdrant and compared.

## Features

### Eligibility decision

The program returns whether the student may enrol, and the reason when they may not.

Enrol in one PostgreSQL transaction:

- Lock the section row with `SELECT ... FOR UPDATE`.
- Check `seats_taken < capacity`.
- Insert the enrolment.
- Increment `seats_taken`.

Schema checks that the decision relies on:

- `UNIQUE (student_id, section_id)`
- `CHECK (seats_taken <= capacity)`

A student may enrol only in a course whose prerequisites they have completed. Completed courses are `(:Student)-[:COMPLETED]->(:Course)`, copied from recorded grades.

Reasons the decision can return:

- The locked section has no free seat (`seats_taken` is not less than `capacity`).
- The student already has an enrolment for that section (`UNIQUE (student_id, section_id)`).
- A prerequisite is not completed.
- Two students take the last seat: one enrolment commits, the other is rejected.

Also required, and not a separate product:

- Courses whose prerequisites the student has completed.
- The shortest prerequisite path to a target course.

### Interest-based course suggestions

Suggest courses similar to a free-text interest. Search the course-description embeddings in pgvector and in Qdrant, filtered by semester. Semester and credits are metadata on each description.

### What each store is responsible for

PostgreSQL holds students, courses, sections (`seats_taken` and `capacity`), enrolments, and grades. It runs the enrolment transaction above.

MongoDB holds course descriptions, the system of record for them. Structure differs by course (labs, projects, reading lists). Required query: courses that have both a lab and an exam.

Redis holds the student’s planned courses (session), with a TTL. The plan survives a page reload and expires after inactivity.

Neo4j holds prerequisites, equivalent courses, and `(:Student)-[:COMPLETED]->(:Course)` copied from recorded grades. Required queries: courses whose prerequisites the student has completed; the shortest prerequisite path to a target course.

pgvector and Qdrant both hold course descriptions with semester and credits as metadata. Required query: courses similar to a free-text interest, filtered by semester.

### Cross-store update

Change one source course description and show what happens to its derived copies until they are refreshed:

- the Redis plan entry that refers to that course
- the pgvector embedding
- the Qdrant point
- the Neo4j course node

Until refresh, those copies stay stale. Each derived record keeps the source id and the time it was refreshed.

### pgvector and Qdrant comparison

Both indexes use the same normalised embeddings from `all-MiniLM-L6-v2`, cosine distance, and text ids. The same 20 fixed queries run in both.

For each system:

- Compare its HNSW top-10 with its own exact top-10.
- Report ANN recall@10 (shared ids / 10). Exact top-10 and recall@10 use numpy.

Also report:

- The number of shared ids in the two systems’ top-10 sets.
- With a filter that keeps about 5% of the rows: the number of results returned, ANN recall@10, and latency.
- `hnsw.ef_search`, and whether an iterative scan is on in pgvector.
- Payload indexes on the filtered fields, and `hnsw_ef` in Qdrant.
- One query that pgvector answers in a single SQL statement joined with the business tables (courses and sections), and the same query with Qdrant.

The project search filter is semester. Credits stay on the payload as metadata.

### Data volume

Each store holds 200–1,000 realistic records, generated or taken from a public dataset. Note the source. Faker may be used to generate them. PostgreSQL remains the system of record for students, courses, sections, enrolments, grades, and the decision. MongoDB remains the system of record for course descriptions.

### Tests

Test cases and labels are written before tuning, and handed in. Every number in the report is printed by the submitted code.

Measured cases named for this project:

- 15 eligibility test cases with the expected decision.
- recall@5 and MRR on 20 labelled interest queries.

Acceptance checks from the required queries and schema:

- Two students taking the last seat: one enrolment commits, the other is rejected.
- Courses that have both a lab and an exam.
- The plan survives a page reload and expires after inactivity.
- Courses whose prerequisites the student has completed, and the shortest prerequisite path to a target course.
- The pgvector and Qdrant measurements in the comparison section, including ANN recall@10.

### Docker Compose

One `docker compose up` starts all five services:

- PostgreSQL with pgvector (`pgvector/pgvector:pg16`)
- MongoDB (`mongo:7`)
- Redis (`redis:7-alpine`)
- Neo4j (`neo4j:5-community`)
- Qdrant (`qdrant/qdrant`)

Library versions are pinned in `requirements.txt` when that file is added.

### Deliverables

These are hand-ins, not application features.

- Code, data, and the test cases.
- A 3–4 page report: the role of each store, the main queries, the measured results, and one failure case explained. The last-seat conflict is the failure case to explain.
- Project presentation I (week 7): the data model of all five stores, the vector search in both pgvector and Qdrant, and the measurement plan.

## Technology and libraries

| Piece | Name | What it is for |
| --- | --- | --- |
| Language | Python 3.11+ | The whole program |
| API | FastAPI, uvicorn | Thin HTTP API, no separate frontend |
| PostgreSQL driver | psycopg[binary] | Transactions, `SELECT ... FOR UPDATE`, enrolment and grades |
| PostgreSQL vector type | pgvector | Vector column and cosine search inside PostgreSQL |
| MongoDB driver | pymongo | Course descriptions and the lab-and-exam query |
| Redis client | redis | Planned-course session keys with a TTL |
| Neo4j driver | neo4j | Prerequisites, equivalents, completed courses, shortest path |
| Qdrant client | qdrant-client | The second vector index, payload filters, payload indexes |
| Embeddings | sentence-transformers, model `all-MiniLM-L6-v2` | Normalised embeddings; cosine similarity |
| Exact search | numpy | Exact top-10 and recall@10 |
| Tests | pytest, httpx | Tests against the API, written before tuning |
| Sample data | Faker (optional) | Generating 200–1,000 records per store |
| PostgreSQL image | `pgvector/pgvector:pg16` | PostgreSQL with pgvector |
| MongoDB image | `mongo:7` | Document store |
| Redis image | `redis:7-alpine` | Key-value store |
| Neo4j image | `neo4j:5-community` | Graph store |
| Qdrant image | `qdrant/qdrant` | Vector server |

## What each store holds

PostgreSQL holds students, courses, sections (`seats_taken`, `capacity`), enrolments, and grades, with `UNIQUE (student_id, section_id)` and `CHECK (seats_taken <= capacity)`. It is the system of record because the enrolment decision needs one transaction, a row lock, and those constraints.

MongoDB holds course descriptions whose structure differs by course (labs, projects, reading lists). It is the system of record because those documents do not share one fixed set of fields.

Redis holds the student’s planned courses for the session, under a key with a TTL. A key with a TTL is enough for a plan that must survive a reload and then expire after inactivity.

Neo4j holds prerequisites, equivalent courses, and `(:Student)-[:COMPLETED]->(:Course)` copied from recorded grades. Completion and the shortest prerequisite path are walks over that graph.

pgvector holds the same course-description embeddings, with semester and credits as metadata, next to the business tables so one SQL statement can join similarity to courses and sections.

Qdrant holds the same embeddings and metadata so the filtered ANN search can be compared with pgvector. Derived Redis, graph, and vector records keep the source id and the time they were refreshed.

## Out of scope

- No separate frontend. FastAPI is the whole interface.
- No extra observability stack.
- No login system, payments, or other product features the brief does not ask for.
