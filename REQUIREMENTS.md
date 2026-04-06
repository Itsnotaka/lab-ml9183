# Course project — ML Systems (requirements)

Design and implement an **end-to-end ML system**. You must use techniques from the lectures to address challenges discussed there.

---

## Project context

You will integrate **one or more complementary ML features** into an **existing open-source, self-hosted** system that you run on **Chameleon**.

**Why:** In practice, ML models usually operate inside larger systems with constraints on data, latency, reliability, deployment, and ownership. Building a new service “around the model” avoids those constraints. This course requires designing a **complementary feature** in the context of an **existing system** and its constraints.

### Example host systems (non-exhaustive)

Examples of systems you may complement:

- Meetings/webinars — Jitsi  
- Diagram sketching — Excalidraw  
- Audiobook/podcast library — Audiobookshelf  
- Recipe manager — **Mealie**  
- Photo library — Immich, PhotoPrism  
- Book library/reading — BookLore  
- Blog/publishing — Ghost  
- Team chat — Mattermost, Zulip  
- Media server — Jellyfin  
- Music streaming — Navidrome  
- Note taking — Trilium, DocMost  
- Document storage/sharing — NextCloud  
- Internet search — SearXNG  
- Livestreaming — Owncast  
- Document management — Paperless-ngx  
- Caregiving — BabyBuddy  
- Project management — AppFlowy  
- Personal finance — ActualBudget  
- Travel planning — AdventureLog  
- Fitness tracking — SparkyFitness  
- …

If the base project has **fewer than 2.5k GitHub stars**, get **advance approval** from course staff before the proposal.

You do not have to use the product exactly as intended (e.g. a student-focused chat feature in Zulip is fine).

### Additional requirements

- The ML feature must support **new data and feedback from “users”** in “production,” usable for **retraining**.
- You may use an **LLM out of the box** for part of the project, but you must also include **another model that you train or retrain**.
- You must use at least **one high-quality non-synthetic external dataset** with **known lineage** (who created it, how, etc.).

---

## Group work expectations

Projects are completed in **groups of 3 or 4**. Some elements are **jointly owned**; others are **owned by individual members**.

| Role | Responsibilities |
|------|------------------|
| **All members (joint)** | Project idea and value proposition; high-level approach; overall system integration |
| **(3-person team):** Platform / DevOps | **Shared** across members; each owns automation related to their **primary role** (Unit 3) |
| **Training** | Model training and retraining pipelines (Units 5–6); offline evaluation (part of Unit 8); safeguarding for this role (Unit 10) |
| **Serving** | Model serving (Unit 7); online evaluation and monitoring (part of Unit 8); safeguarding for this role (Unit 10) |
| **Data** | Data pipeline (Unit 4); closing the feedback loop / labels in production (part of Unit 8); emulated operational data; safeguarding for this role (Unit 10) |
| **DevOps / Platform (4-person team only)** | Infrastructure as code, CI/CD/CT, automation (Unit 3); infrastructure monitoring and observability; safeguarding for this role (Unit 10) |

Grading is **partly common** (joint elements) and **partly individual** (work in your assigned role).

**Solo projects are not allowed:** the course requires practicing integration across independently developed components with **well-defined contracts**.

---

## Project deliverables and deadlines

| Milestone | Due date | Points | Scope |
|-----------|----------|--------|--------|
| Project proposal | Mar 2, 2026 | 5 / 40 | Problem statement, data sources, modeling approach, alignment with business requirements |
| **Initial implementation** | **Apr 6, 2026** | **10 / 40** | Data, model training, model serving implemented **individually** (not necessarily integrated); overall pipeline with dummy steps also for 4-person groups |
| System implementation | Apr 20, 2026 | 15 / 40 | All components integrated end-to-end, including safeguarding |
| Ongoing operation | May 4, 2026 | 10 / 40 | Emulated “live” data; operational behavior, stability, evaluation over time |

More detail is released ahead of each deadline.

---

## Project proposal (due Mar 2)

**Focus:** Intent, feasibility, business alignment.

**Format:** Document (max **2 pages**) and **slides** for a **10 min** (3-person) or **12 min** (4-person) presentation; sign up for a slot in the week of Mar 2.

### Requirements checklist (all must be satisfied)

- Team defines a **hypothetical service** into which the ML feature integrates.  
- Service is realized with an **existing open-source** project (**≥ 2.5k** GitHub stars unless approved).  
- ML feature(s) are **complementary**.  
- Service is **fully hosted on Chameleon**.  
- Design includes **at least one trained/retrained** model.  
- Training uses **≥ one high-quality non-synthetic external dataset** with known lineage.  
- In “production,” the system gets **new data and feedback** for **retraining**.

### Rubric (summary)

- **Joint (3 / 5 points):** Public-facing service (not the ML feature), audience; ML feature design (incl. feedback for retraining); external dataset(s), examples, lineage.  
- **Training (2 / 5):** Model type(s), training/retraining; inputs and outputs.  
- **Serving (2 / 5):** Operational requirements (RPS, latency, etc.) with justification; how model output becomes a real outcome.  
- **Data (2 / 5):** Data flow (real-time vs training); training data, candidate selection, leakage avoidance.  
- **DevOps / Platform — 4-person only (2 / 5):** Model **freshness** / retraining cadence in the automation lifecycle; **scaling / right-sizing** (peak vs typical usage).

---

## Initial implementation (due Apr 6)

**Focus:** Each member delivers a **runnable, role-owned subsystem** on Chameleon, built to a **shared interface** (example payloads). Components **need not** be integrated end-to-end or with the open-source system yet.

**Resources (Chameleon):** Include **project ID** (e.g. `proj99`) in names for leases, instances, volumes, buckets, security groups. Follow cost practices (shut down compute when idle, object storage for large data/checkpoints, block storage for small state, smallest suitable instance, typically **one floating IP** as jump host). Plan **GPU** reservations ahead (course lists instance types and lead times).

**Format symbols:** 📝 written · 🎥 video · 📄 in repo · 💻 live on Chameleon

### Joint responsibilities (1 / 10 points)

- 📄 **One JSON input sample** and **one model output sample** (real representative values), agreed by Training, Serving, and Data. One pair per model if multiple models.

- **(4-person team only)** 📝 **Table:** All **containers involved in each role**, with links to **Dockerfiles / Docker Compose** and to the **equivalent Kubernetes manifest** for each. Goal: show each **role-owned system** will be supported by **DevOps/Platform**, even while members work independently.

### Training (9 / 10 points)

Deliverables include: 📝 **training runs table** (rows link to **MLflow** runs; highlight best candidates); 📄 **Dockerfile(s)** for training (optional dev Dockerfile), **training code as Python scripts** (not notebook-only), **config** if separate; 🎥 sped-up **full training run** in Docker on Chameleon; 💻 **live MLflow** on Chameleon with all runs.

**Requirements:** Runs on Chameleon **in containers**; tracked in MLflow; **config-driven** hyperparameters (single config dict, YAML/JSON, or CLI—not one-off scripts per run); log config, quality metrics, **cost metrics**, environment (e.g. GPU); include **≥ 1 baseline** and alternatives for tradeoff discussion; reasonable hyperparameter search.

### Serving (9 / 10 points)

Deliverables: 📝 **Serving options table** (latency, throughput, hardware, etc.; mark best options); 📄 **Dockerfile(s)**, serving code/config, eval scripts/notebooks; 🎥 sped-up demo of **most promising** option on Chameleon responding to agreed requests.

**Requirements:** Experiments on Chameleon **in containers**; **multiple options** including baseline and optimizations (**model-, system-, and infrastructure-level**, separately and combined); right-size CPU/memory (**GPU** if applicable) for top options.

### Data (9 / 10 points)

Deliverables: 📝 **Data design doc** (repos, schema, writers, versioning, diagrams); 💻 **live object storage** on Chameleon; 🎥📄 pipelines for ingest, generator hitting endpoints, online features, batch training/eval datasets (each with demo video as specified).

### DevOps / Platform — 4-person only (9 / 10 points)

Deliverables: 📝 **Infrastructure requirements table** (GPU/CPU/memory requests & limits + Chameleon evidence); 📄 **IaC/CaC** provisioning Chameleon + **Kubernetes**, manifests to deploy **open-source base** and **platform services** for teammates; 🎥 open-source service **in Kubernetes** on Chameleon; 🎥 **shared platform services** in Kubernetes **with persistence**.

**Requirements:** Kubernetes required for 4-person teams; Git source of truth; durability across pod restarts; no secrets in Git.

---

## Document map (this repo)

| File | Contents |
|------|----------|
| [task.md](task.md) | **Q2 DevOps/Platform** Gradescope prompts (Q2.1–Q2.4, Q3 bonus) — formatted |
| [REQUIREMENTS.md](REQUIREMENTS.md) | Full **course project** requirements (this file) |

---

*Source: course project specification and Gradescope text as provided for ChefMate / ML Systems. Verify dates and point totals against the live course site if they change.*
