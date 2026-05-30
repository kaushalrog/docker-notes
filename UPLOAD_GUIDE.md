# 📦 How to Upload These Docker Notes to GitHub

A step-by-step guide to push your Docker notes to a GitHub repository.

---

## Step 1 — Create a GitHub Repository

1. Go to [https://github.com/new](https://github.com/new)
2. Fill in:
   - **Repository name**: e.g., `docker-notes`
   - **Description**: e.g., `Complete Docker reference notes`
   - **Visibility**: Public or Private
   - ❌ Don't initialize with README (we'll push our own)
3. Click **Create repository**

---

## Step 2 — Set Up Git Locally

```bash
# Configure Git (first time only)
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
```

---

## Step 3 — Initialize and Push

```bash
# Navigate to your notes folder
cd docker-notes/

# Initialize a git repository
git init

# Add all files
git add .

# First commit
git commit -m "feat: add complete Docker notes"

# Add GitHub as remote origin
git remote add origin https://github.com/YOUR_USERNAME/docker-notes.git

# Push to GitHub
git push -u origin main
```

> If your branch is named `master` instead of `main`, use:
> ```bash
> git branch -M main   # rename to main first
> git push -u origin main
> ```

---

## Step 4 — Verify

Visit `https://github.com/YOUR_USERNAME/docker-notes`
You should see your `README.md` rendered with all the Docker notes.

---

## Step 5 — Making Updates Later

```bash
# After editing any file:
git add .
git commit -m "docs: update Docker Compose section"
git push
```

---

## Recommended Folder Structure

```
docker-notes/
├── README.md                      ← Main notes (renders on GitHub)
├── Dockerfile.example             ← Multi-stage Dockerfile reference
├── docker-compose.example.yml     ← Full Compose example
├── .dockerignore.example          ← .dockerignore template
├── UPLOAD_GUIDE.md                ← This file
└── examples/
    ├── nodejs/
    │   ├── Dockerfile
    │   └── docker-compose.yml
    ├── python/
    │   └── Dockerfile
    └── golang/
        └── Dockerfile
```

---

## Pro Tips

### Add a License
```bash
# Add MIT license
curl -o LICENSE https://raw.githubusercontent.com/nicklockwood/MIT-LICENSE/master/LICENSE
git add LICENSE && git commit -m "chore: add MIT license"
```

### Add GitHub Topics
On your repo page → ⚙ Settings → scroll to **Topics** → add:
`docker`, `devops`, `containers`, `docker-compose`, `notes`, `cheatsheet`

### Pin the Repository
Go to your GitHub profile → click **Customize your pins** → add `docker-notes`

### Enable GitHub Pages (optional)
Settings → Pages → Source: `main` branch, `/root` folder
→ Your notes become a website at `https://YOUR_USERNAME.github.io/docker-notes`

---

## Useful Git Commands

```bash
git status                          # Check what changed
git log --oneline                   # View commit history
git diff                            # See uncommitted changes
git remote -v                       # Check remote URL
git pull                            # Get latest changes
```
