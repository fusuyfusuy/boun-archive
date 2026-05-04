# BOUN Archive: Comprehensive Technical Specification

## 1. Executive Summary
"BOUN Archive" is an academic analytics platform designed to visualize, analyze, and predict course scheduling trends at Bogazici University. Utilizing a dataset of over 140,000 historical course slots spanning 50+ years, the platform provides tools for longitudinal research (e.g., department growth, instructor legacies) and predictive scheduling (e.g., forecasting future course availability).

## 2. Tech Stack Definition
*   **Frontend:** SvelteKit (App Router equivalent), Tailwind CSS, D3.js or LayerChart (for visualizations).
*   **Backend:** FastAPI (Python), Uvicorn, Pydantic (data validation), Pandas/Polars (for data processing).
*   **Database:** PostgreSQL (Primary OLTP), Meilisearch (Secondary for fast faceted search).
*   **Deployment:** Dockerized containers, Nginx reverse proxy.

---

## 3. Database Schema (PostgreSQL)

The primary data source must be normalized from the raw scraped HTML/CSV into a highly queryable relational structure.

### `terms` (Academic Semesters)
| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | VARCHAR(15) | PRIMARY KEY | e.g., `2024/2025-1` |
| `academic_year` | VARCHAR(9) | INDEX | e.g., `2024/2025` |
| `semester_num` | INTEGER | | `1` (Fall), `2` (Spring), `3` (Summer) |

### `departments`
| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `kisaadi` | VARCHAR(10) | PRIMARY KEY | Short code, e.g., `INTT` |
| `bolum` | VARCHAR(50) | | Full name, e.g., `INTERNATIONAL TRADE` |

### `instructors`
| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | SERIAL | PRIMARY KEY | |
| `full_name` | VARCHAR(100)| UNIQUE, INDEX | e.g., `ASLI DENİZ HELVACIOĞLU` |

### `courses`
| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | SERIAL | PRIMARY KEY | |
| `term_id` | VARCHAR(15) | FK (terms.id), INDEX | |
| `dept_kisaadi`| VARCHAR(10) | FK (departments.kisaadi) | |
| `course_code` | VARCHAR(20) | INDEX | e.g., `INTT514` |
| `section` | VARCHAR(5) | | e.g., `01` |
| `title` | VARCHAR(255)| | e.g., `INTERNATIONAL POLITICAL ECONOMY` |
| `instructor_id`| INTEGER | FK (instructors.id), INDEX| |
| `credits` | INTEGER | | |
| `ects` | INTEGER | | |
| `delivery_method`| VARCHAR(50) | | `Online`, `Face-to-Face`, etc. |

### `course_slots`
| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | SERIAL | PRIMARY KEY | |
| `course_id` | INTEGER | FK (courses.id), INDEX | |
| `day_code` | VARCHAR(2) | INDEX | `M`, `T`, `W`, `Th`, `F`, `St`, `Su` |
| `slot_hour` | INTEGER | INDEX | `1` through `14` |
| `room` | VARCHAR(50) | INDEX | e.g., `HKC325` |

---

## 4. Analytical Engine & Algorithms

### A. The "Ghost Schedule" (Time Machine)
**Objective:** Reconstruct the physical campus layout for any historical hour.
**Algorithm:**
1.  Query `course_slots` joining `courses` and `terms` where `term_id = X` and `day_code = Y`.
2.  Group by `room`.
3.  Output a 2D matrix (Rooms vs. Hours) for rendering on the frontend.

### B. The Trend Engine (Statistical Prediction)
**Objective:** Predict the likelihood of a course being offered in an upcoming semester and its likely time slot.
**Algorithm (Frequency & Moving Average):**
1.  **Availability Probability:**
    *   Look back $N$ years (e.g., $N=5$).
    *   Count how many times course $C$ was offered in Fall vs. Spring vs. Summer.
    *   Probability of Fall offering = $(Count_{Fall} / N) * 100$.
2.  **Time Slot Prediction:**
    *   Fetch all historical `course_slots` for course $C$.
    *   Create a frequency map of `(day_code, slot_hour)`.
    *   Apply a recency weight (slots from the last 2 years carry 1.5x weight compared to slots from 10 years ago).
    *   Return the top 3 highest-scoring slot combinations.

---

## 5. API Specification (FastAPI)

### `GET /api/v1/courses/search`
*   **Description:** Full-text search across all historical courses (proxied to Meilisearch).
*   **Params:** `q` (string), `term` (optional), `dept` (optional).
*   **Response:** List of course objects with matched highlights.

### `GET /api/v1/analytics/department/{kisaadi}/growth`
*   **Description:** Time-series data for department size.
*   **Response:**
    ```json
    {
      "years": ["2010", "2011", "2012"],
      "total_courses": [45, 48, 52],
      "unique_instructors": [12, 14, 15]
    }
    ```

### `GET /api/v1/analytics/instructor/{id}/legacy`
*   **Description:** Comprehensive teaching history for an instructor.
*   **Response:**
    ```json
    {
      "instructor_name": "SEMA SAKARYA",
      "total_semesters_taught": 30,
      "most_frequent_courses": ["INTT551", "INTT690"],
      "preferred_slots": [{"day": "T", "hour": 5, "frequency": 45}]
    }
    ```

### `GET /api/v1/predict/course/{course_code}`
*   **Description:** The Trend Engine forecast output.
*   **Response:**
    ```json
    {
      "course_code": "INTT514",
      "next_semester_probability": {"Fall": 85, "Spring": 10, "Summer": 5},
      "predicted_slots": [
        {"day": "M", "hour": 2, "confidence_score": 0.78},
        {"day": "W", "hour": 3, "confidence_score": 0.65}
      ]
    }
    ```

---

## 6. Frontend Architecture (SvelteKit)

### Key Routes / Views
1.  **`/` (Dashboard):** Global university statistics (e.g., "Total courses taught since 1970", a massive heatmap of all campus activity).
2.  **`/search`:** The Meilisearch-powered instant search interface. Filters for Decade, Instructor, and Department.
3.  **`/ghost-schedule/{term}`:**
    *   A massive, interactive grid.
    *   X-axis: Days/Hours. Y-axis: Rooms or Departments.
    *   Affordance: A slider to "play" the campus activity hour-by-hour (visualizing building utilization).
4.  **`/trends/course/{course_code}`:**
    *   Shows the predictive dashboard for a specific class.
    *   Line charts showing enrollment/credit trends.
    *   A "Radar" chart showing the time-slot probability distribution.
5.  **`/legacy/{instructor_id}`:**
    *   The "Instructor DNA" page.
    *   A timeline visualization (Gantt chart) of their teaching history.

---

## 7. Data Ingestion Pipeline

To move data from the current scraper outputs to the production database:
1.  **Cleanse:** Read `schedules.csv`. Normalize whitespace.
2.  **Entity Resolution:** Standardize instructor names (e.g., mapping "STAFF STAFF" to a generic ID, handling minor typos using Levenshtein distance).
3.  **Load PostgreSQL:** Use bulk inserts (`COPY` command or Pandas `to_sql` for speed).
4.  **Sync Meilisearch:** Create a flattened JSON structure for every course (combining metadata and slots) and push to the Meilisearch index.
