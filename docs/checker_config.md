# Public Export — Automatic Task Export

## Overview

**Public Export** — a mechanism for automatic export of tasks from the instructors' private repository (`private`) to the public repository (`public`) accessible to students.

### How It Works

```
┌─────────────────┐      CI Pipeline      ┌─────────────────┐
│     private     │ ──────────────────▶   │     public      │
│  (instructors)  │   checker export      │   (students)    │
│                 │      --commit         │                 │
│  - solutions    │                       │  - templates    │
│  - tests        │                       │  - public       │
│  - configs      │                       │    tests        │
└─────────────────┘                       └─────────────────┘
```

On push to the `main` branch of the `private` repository:
1. CI pipeline starts
2. Job `deploy-public` copies public files
3. Solutions are replaced with templates (`.template` files)
4. Changes are committed and pushed to `public`

---

## Repository Structure

### Private Repository (Instructors)

```
private/
├── .checker.yml          # Checker configuration (private)
├── .manytask.yml         # Course configuration (private)
├── .releaser-ci.yml      # CI for instructors (private)
├── .gitlab-ci.yml        # CI for students (public)
├── base.docker           # Dockerfile for base image
├── testenv.docker        # Dockerfile for test environment
├── pyproject.toml        # Python dependencies
├── tools/                # Testing tools (public)
│   ├── plugins/
│   └── testlib/
└── Python/               # Task group
    ├── .group.yml
    ├── add/              # Task
    │   ├── .task.yml
    │   ├── add.py            # Solution (NOT exported)
    │   ├── add.py.template   # Template (exported as add.py)
    │   ├── test_public.py    # Public tests
    │   └── test_private.py   # Private tests (NOT exported)
    └── subtract/
        └── ...
```

### Public Repository (Students)

```
public/
├── .gitlab-ci.yml        # CI for testing
├── pyproject.toml
├── tools/
└── Python/
    ├── .group.yml
    ├── add/
    │   ├── .task.yml
    │   ├── add.py            # Template (from add.py.template)
    │   └── test_public.py
    └── subtract/
        └── ...
```

---

## Configuration

### 1. `.checker.yml` — Structure and Export

```yaml
version: 1

structure:
  # Ignored files/folders
  ignore_patterns:
    - ".git"
    - "__pycache__"
    - ".venv"
    - "*.pyc"
  
  # Public files — exported, overwritten during testing
  public_patterns:
    - ".gitlab-ci.yml"
    - ".task.yml"
    - ".group.yml"
    - "README.md"
    - "test_public.py"
    - "tools"
    - ".gitignore"
  
  # Private files — NOT exported
  private_patterns:
    - ".*"              # All dot-files
    - "test_private.py"

export:
  destination: https://gitlab.manytask.org/sandbox/public
  default_branch: main
  commit_message: "chore(auto): export new tasks"
  templates: search_or_create  # search, create, search_or_create
```

### 2. `.manytask.yml` — Task Schedule

```yaml
version: 1

settings:
  course_name: MyCourseName
  gitlab_base_url: https://gitlab.manytask.org
  public_repo: sandbox/public
  students_group: sandbox/students

deadlines:
  timezone: Europe/Moscow
  deadlines: hard

  schedule:
    - group: Python
      start: 2025-01-01 18:00:00
      end: 2025-06-01 23:59:00
      enabled: true
      tasks:
        - task: add
          score: 100
        - task: subtract
          score: 100
```

### 3. `.releaser-ci.yml` — CI for Export

```yaml
variables:
  REGISTRY: gitlab.manytask.org:5050/sandbox/public

stages:
  - build
  - deploy

deploy-public:
  image: $REGISTRY/base-image:latest
  stage: deploy
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: on_success
    - when: never
  script:
    # Clone the public repository
    - git clone https://oauth2:$GITLAB_API_TOKEN@gitlab.manytask.org/sandbox/public ./export
    - cd ./export && git config user.email "ci@manytask.org" && git config user.name "CI Bot" && cd ..
    # Export and push
    - python3 -m checker export --commit
```

### Optional: an explicit allow-list with `allow_change`

Student-editable files are not a built-in concept — by convention they are listed in a task parameter (commonly
named `allow_change`), a list of glob patterns relative to the task dir. Set it once for all tasks via
`default_parameters` in `.checker.yml`, and override it per group/task in `.group.yml`/`.task.yml` when a task
needs different files editable. It is used by the [`copy_files` plugin](checker_plugins.md) to pick which files
to collect from the student's repo for testing, and by `testing.skip_unchanged_tasks` (see the
[reference](checker_yml_reference.md#skip_unchanged_tasks)) to detect a freshly pulled, untouched task:

```yaml
# .checker.yml
default_parameters:
  allow_change: ["solution.c"]

testing:
  skip_unchanged_tasks: allow_change
```

```yaml
# Python/new_task/.task.yml — override for this task only
version: 1
parameters:
  allow_change: ["solution.c", "extra_file.h"]
```

Without `skip_unchanged_tasks`, a student running `git pull` on the public repo to fetch newly released tasks
gets those tasks graded immediately against the untouched template `solution.c` — they fail, and with
`report_on_failure` that failure is reported to manytask as a submission, charging a penalty for a task the
student hasn't started. With `skip_unchanged_tasks: allow_change` set, `checker grade` compares `solution.c`
byte-for-byte against the published template and skips grading/reporting for that task instead.

With `changes_detection: last_commit_changes` (the default), `checker grade` tests only the tasks touched by the
pushed commits (diff against `CI_COMMIT_BEFORE_SHA` in GitLab CI, or `--base-ref`; `HEAD~1` otherwise), so a push
of several commits at once still grades tasks from all of them, not only the last one.

---

## CI/CD Variables Setup

### Required Variables

| Variable | Description | Where to Create |
|----------|-------------|-----------------|
| `GITLAB_API_TOKEN` | Token for push to public repo | Project Settings → CI/CD → Variables |
| `DOCKER_AUTH_CONFIG` | Docker Registry authentication | Group Settings → CI/CD → Variables |
| `TESTER_TOKEN` | Manytask API token | Group Settings → CI/CD → Variables |

### Creating GITLAB_API_TOKEN

1. Go to GitLab → **Group Settings → Access Tokens**
2. Create a token:
   - **Name:** `ci-deploy-public`
   - **Role:** **Maintainer**
   - **Scopes:** `write_repository`
3. Add to **CI/CD Variables** in the `private` repository:
   - **Key:** `GITLAB_API_TOKEN`
   - **Value:** the created token

---

## Task Templates

### Option 1: `.template` Files (Recommended)

Create a `solution.py.template` file next to `solution.py`:

**solution.py** (solution):
```python
def add(a, b):
    return a + b
```

**solution.py.template** (template for students):
```python
def add(a, b):
    # Implement me
```

During export, `solution.py.template` will replace `solution.py`.

### Option 2: Template Comments

Use comments in the code:

```python
def add(a, b):
    # SOLUTION BEGIN
    return a + b
    # SOLUTION END
```

During export, code between the comments will be replaced with `# TODO: Your solution`.

---

## Adding a New Task

### 1. Create the Structure

```bash
mkdir -p Python/new_task
```

### 2. Create the Files

**Python/new_task/.task.yml:**
```yaml
version: 1
```

**Python/new_task/solution.py:**
```python
def solve():
    return 42
```

**Python/new_task/solution.py.template:**
```python
def solve():
    # Implement me
```

**Python/new_task/test_public.py:**
```python
from solution import solve

def test_solve():
    assert solve() == 42
```

### 3. Add to `.manytask.yml`

```yaml
tasks:
  - task: new_task
    score: 100
```

### 4. Commit and Push

```bash
git add .
git commit -m "Add new_task"
git push origin main
```

The pipeline will automatically export the task to `public`.
