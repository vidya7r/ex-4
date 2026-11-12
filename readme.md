# CI Pipeline Lab: Python App → GitHub Actions → Docker Hub

I cleaned up the code, because the pasted version had broken line breaks and lost indentation. Python and YAML both depend on indentation, so copy the code from here. I also removed the extras (QEMU setup, multi-platform build, pull-request trigger, stretch goals).

## Key concepts

| Term | Meaning |
|---|---|
| **CI (Continuous Integration)** | Every time you push code, a server automatically tests and builds it. |
| **GitHub Actions** | GitHub's built-in CI service. It reads workflow files from `.github/workflows/`. |
| **Workflow** | A YAML file describing *when* to run and *what steps* to run. |
| **Runner** | The temporary Ubuntu machine GitHub gives you to run the workflow. |
| **Secret** | An encrypted value (like a password) stored in GitHub and used by the workflow. |
| **pytest** | A Python testing tool. A failing test stops the pipeline, so broken code never reaches Docker Hub. |

**The flow:**
```
git push → GitHub Actions starts → run pytest → build image → push to Docker Hub
                                       ↓ (if tests fail)
                                 pipeline stops, nothing pushed
```

## Step 0: Prerequisites

- **Where:** browser and terminal.
- **What to do:**
  1. Make sure you have a GitHub account and a Docker Hub account.
  2. Check the tools:
```powershell
git --version
python --version
docker --version
```
- **Expected:** a version number for each. Python must be 3.11 or higher.
- **Purpose:** Git uploads your code, Python runs the app and tests, and Docker is for the optional local test.

## Step 1: Create the project folder

- **Where:** PowerShell.
- **How:**
```powershell
mkdir python-ci-docker-lab
cd python-ci-docker-lab
mkdir tests
```
- **Expected:** you are inside the new folder, which contains an empty `tests` folder.
- **Purpose:** all files must live in one folder, which becomes the Git repository.

## Step 2: Create the application files

- **Where:** a text editor such as VS Code. Save each file at the path shown.
- **Expected behavior:** nothing runs yet, you are only creating files.

**`app.py`** (the program)
```python
def add(a, b):
    return a + b

if __name__ == "__main__":
    print("Hello from Python CI Lab!")
    print("2 + 3 =", add(2, 3))
```
- **Purpose:** `add()` is the function we will test. The `if __name__ == "__main__":` block runs only when you execute the file directly. Remember that `"__main__"` needs quotes, which was the bug in your last experiment.

**`tests/test_app.py`** (the test)
```python
from app import add

def test_add():
    assert add(2, 3) == 5
    assert add(-1, 1) == 0
```
- **Purpose:** pytest runs any function starting with `test_`. `assert` fails the test if the condition is false. This test is the "gate" that CI checks.

**`conftest.py`** (empty file in the project root, *not* inside `tests/`)
```powershell
New-Item conftest.py -ItemType File
```
- **Purpose:** this fixes a common error. Without it, pytest can't find `app.py` from inside the `tests/` folder and fails with `ModuleNotFoundError: No module named 'app'`. An empty `conftest.py` in the root tells pytest to add the root folder to its search path.

**`requirements.txt`**
```
pytest==8.3.2
```
- **Purpose:** the libraries to install, both locally and in CI.

**`Dockerfile`** (no extension)
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```
- **Purpose:** this builds an image containing Python and your app. Same recipe as the last experiment. `--no-cache-dir` keeps the image smaller. There is no `EXPOSE` or port, because this app just prints text and exits. It is not a web server.

**`.gitignore`**
```
__pycache__/
.venv/
.pytest_cache/
*.pyc
```
- **Purpose:** tells Git to skip generated files so they aren't uploaded.

## Step 3: Add the GitHub Actions workflow

- **Where:** PowerShell to create the folder, then the editor for the file.
- **How:**
```powershell
mkdir .github\workflows
```
(In PowerShell this creates both folders at once. The `mkdir -p` from the lab sheet is for Linux/Mac.)

Create **`.github/workflows/ci-dockerhub.yml`**:
```yaml
name: ci-dockerhub

on:
  push:
    branches: ["main"]
    tags: ["*"]

jobs:
  build-test-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install deps
        run: pip install -r requirements.txt

      - name: Run tests
        run: pytest -q

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ secrets.DOCKERHUB_USERNAME }}/python-ci-lab
          tags: |
            type=raw,value=latest,enable={{is_default_branch}}
            type=sha,prefix=sha-,format=short
            type=ref,event=tag

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```
- **Expected behavior:** nothing runs yet. It only runs after you push to GitHub.
- **Purpose of each part:**

| Part | Purpose |
|---|---|
| `on: push` (main and tags) | Run the workflow when you push to `main` or push a tag like `v1.0.0`. |
| `runs-on: ubuntu-latest` | Use a fresh Ubuntu machine from GitHub. |
| `checkout` | Download your repo code onto the runner. |
| `setup-python` and `Install deps` | Install Python 3.11 and pytest. |
| `Run tests` | If this fails, **all later steps are skipped**. This is the CI gate. |
| `Docker meta` | Generates the image tags: `latest`, `sha-abc1234` (commit ID), and the git tag (like `v1.0.0`). |
| `Buildx` | Docker's modern build engine, required by the build action. |
| `Login` | Logs in to Docker Hub using your secrets. |
| `Build and push` | Builds the image from your Dockerfile and uploads it with those tags. |

- **Warning:** YAML is strict about indentation. Use spaces, never tabs, and keep the nesting exactly as shown.

## Step 4: Test locally (optional but recommended)

- **Where:** PowerShell, inside `python-ci-docker-lab`.
- **How:**
```powershell
pip install -r requirements.txt
python app.py
pytest -q
docker build -t python-ci-lab:local .
docker run --rm python-ci-lab:local
```
- **Expected output:**

| Command | Expected |
|---|---|
| `python app.py` | `Hello from Python CI Lab!` and `2 + 3 = 5` |
| `pytest -q` | `1 passed` |
| `docker build` | Build completes without errors. |
| `docker run --rm` | The same two lines print, then the container exits and is removed. |

- **Purpose:** it is much faster to find mistakes on your own machine than to wait for the cloud pipeline. `--rm` deletes the container after it finishes.

## Step 5: Create the GitHub repository and push

- **Where:** first the browser, then PowerShell.
- **In the browser:**
  1. Go to https://github.com/new.
  2. Name it `python-ci-docker-lab`, keep it **empty** (no README or .gitignore), and click **Create repository**.
- **In PowerShell:**
```powershell
git init
git add .
git commit -m "init lab"
git branch -M main
git remote add origin https://github.com/<your-username>/python-ci-docker-lab.git
git push -u origin main
```
- **Expected behavior:** the files appear in your GitHub repo. Warnings like `LF will be replaced by CRLF` are harmless on Windows. If asked to authenticate, sign in through the browser prompt.
- **Purpose:** the workflow lives on GitHub and only triggers when code is pushed there.
- **Note:** this first push will start the workflow, but it will **fail at the Docker login step** because the secrets don't exist yet. That is fine. After Step 6 you will re-run it.

## Step 6: Add the GitHub secrets

**6a. Create a Docker Hub access token** (browser)
1. Log in to https://hub.docker.com.
2. Click your profile picture → **Account settings** → **Personal access tokens** (under Security).
3. Click **Generate new token**, give it a name like `github-actions`, and set permissions to **Read & Write**.
4. **Copy the token immediately.** You can't view it again.

**6b. Add secrets to GitHub** (browser)
1. Open your repo → **Settings** → **Secrets and variables** → **Actions**.
2. Click **New repository secret** and add two secrets:

| Name | Value |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | The token you just copied |

- **Expected behavior:** both names appear in the secrets list, with values hidden.
- **Purpose:** the pipeline needs to log in to Docker Hub, but passwords must never be written in code files. Secrets keep them encrypted and hidden from logs. A token is safer than your real password because you can limit it and revoke it anytime.
- **Important:** the names must match the workflow **exactly**, including capitals.

## Step 7: Watch the CI run

- **Where:** browser, your GitHub repo.
- **How:**
  1. Click the **Actions** tab.
  2. Click the latest run (`ci-dockerhub`). If the first run failed because of missing secrets, click **Re-run all jobs**. You can also trigger a new run with an empty commit:
```powershell
git commit --allow-empty -m "trigger ci"
git push
```
  3. Click the job `build-test-push` to expand each step and read the logs.
- **Expected behavior:** every step has a green ✅ tick. In **Run tests** you will see `1 passed`.
- **Purpose:** this is the "CI" in action. Every step is automated and visible.
- **If a step is red ❌:** click it to read the error. See Troubleshooting below.

## Step 8: Verify the image on Docker Hub

- **Where:** browser.
- **How:** go to `https://hub.docker.com/r/<your-username>/python-ci-lab` and open the **Tags** tab.
- **Expected:** tags `latest` and `sha-xxxxxxx` are listed.
- **Optional check on your machine:**
```powershell
docker pull <your-username>/python-ci-lab:latest
docker run --rm <your-username>/python-ci-lab:latest
```
- **Expected output:** `Hello from Python CI Lab!` and `2 + 3 = 5`.
- **Purpose:** this proves the image was built and published by the pipeline, not by you manually.

## Step 9: Release with a tag (optional)

- **Where:** PowerShell.
- **How:**
```powershell
git tag v1.0.0
git push origin v1.0.0
```
- **Expected behavior:** a new workflow run starts. On Docker Hub, a new tag **`v1.0.0`** appears next to `latest`.
- **Purpose:** version tags give you fixed, named releases. `latest` keeps changing, but `v1.0.0` always points to this exact version, so you can roll back to it.

## Troubleshooting

| Problem | Cause and fix |
|---|---|
| `ModuleNotFoundError: No module named 'app'` in tests | The `conftest.py` file is missing from the project root. Create it (Step 2). |
| `Error: Username and password required` | Secret names are misspelled or missing. Recheck Step 6b. |
| `unauthorized` or `denied` on login or push | Token is wrong, expired, or has only Read permission. Create a new Read & Write token and update the secret. |
| `invalid reference format` in Docker meta | `DOCKERHUB_USERNAME` secret is empty or missing. |
| Workflow doesn't start | The file must be at exactly `.github/workflows/ci-dockerhub.yml`, and you must push to `main`. |
| `Invalid workflow file` or YAML error | Indentation problem. Use spaces only and copy the YAML exactly. |
| Tests fail on the runner | Run `pytest -q` locally and fix first. |
| No image on Docker Hub | Open the failed step in Actions and read the log. The pipeline stops at the first failure. |

## What to remember for viva

1. **CI** means automatically testing and building on every push.
2. The pipeline order is **test → build → push**, and a failed test stops everything.
3. **Secrets** keep credentials out of source code.
4. **Tags** (`latest`, `sha-...`, `v1.0.0`) identify image versions.
5. The Dockerfile is the same concept as in your Flask experiment. The only new part is that GitHub's server runs `docker build` and `docker push` for you.

If you get an error in any step, paste the exact error message or the failing log lines from the Actions tab and I'll help you fix it.