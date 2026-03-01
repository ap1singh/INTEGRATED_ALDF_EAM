# GitHub Repository Setup Guide for ALDF-EAM
## Complete Step-by-Step Instructions (First-Time GitHub User)

---

## Part 1: Pre-requisites (15 minutes)

### Step 1: Create GitHub Account
1. Go to https://github.com
2. Click "Sign up"
3. Choose username (suggestion: `atulpratapsingh` or `atul-singh-research`)
4. Use your professional email
5. Verify email address

### Step 2: Install Git on Your Computer
**Windows:**
1. Download from https://git-scm.com/download/win
2. Run installer (accept default options)
3. Open "Git Bash" from Start menu

**Mac:**
1. Open Terminal
2. Type: `git --version`
3. If not installed, follow prompts to install Xcode Command Line Tools

**Linux:**
```bash
sudo apt-get update
sudo apt-get install git
```

### Step 3: Configure Git
Open terminal/Git Bash and run:
```bash
git config --global user.name "Atul Pratap Singh"
git config --global user.email "your.email@example.com"
git config --global init.defaultBranch main
```

---

## Part 2: Create GitHub Repository (10 minutes)

### Step 1: Create New Repository on GitHub
1. Log in to GitHub
2. Click the **"+"** button (top right) → "New repository"
3. Fill in:
   - **Repository name:** `ALDF-EAM`
   - **Description:** "Adaptive Leakage Detection Framework for Evolving AMI Maturity"
   - **Visibility:** Public ✅ (for open-source)
   - **Initialize:** 
     - ❌ Do NOT check "Add a README" (we have our own)
     - ❌ Do NOT add .gitignore (we have our own)
     - ❌ Do NOT choose a license (we have MIT)
4. Click **"Create repository"**

### Step 2: Note Your Repository URL
You'll see a page with setup instructions. Note the URL:
```
https://github.com/YOUR_USERNAME/ALDF-EAM.git
```

---

## Part 3: Prepare Files Locally (5 minutes)

### Step 1: Create Project Folder
```bash
# Create a folder for your project
mkdir ALDF-EAM
cd ALDF-EAM
```

### Step 2: Copy Files You've Created
Copy these files into the `ALDF-EAM` folder:

**From this chat session, you should have:**
1. `README.md` ✅ (we just created)
2. `LICENSE` ✅ (we just created)
3. `requirements.txt` ✅ (we just created)
4. `CONTRIBUTING.md` ✅ (we just created)
5. `.gitignore` ✅ (we just created)

**You need to add:**
6. `ALDF_EAM_Implementation.ipynb` (your Colab notebook)
7. `data/synthetic_dataset.csv` (if you have it ready)

**Folder structure should look like:**
```
ALDF-EAM/
├── README.md
├── LICENSE
├── requirements.txt
├── CONTRIBUTING.md
├── .gitignore
├── ALDF_EAM_Implementation.ipynb
└── data/
    └── synthetic_dataset.csv
```

---

## Part 4: Initialize Git and Push to GitHub (10 minutes)

### Step 1: Initialize Git Repository
Open terminal in the `ALDF-EAM` folder and run:

```bash
# Initialize Git repository
git init

# Add all files
git add .

# Check what will be committed
git status
```

You should see all your files listed in green.

### Step 2: Make First Commit
```bash
git commit -m "Initial commit: ALDF-EAM framework with documentation"
```

### Step 3: Connect to GitHub
Replace `YOUR_USERNAME` with your actual GitHub username:

```bash
# Add GitHub as remote
git remote add origin https://github.com/YOUR_USERNAME/ALDF-EAM.git

# Rename branch to 'main' (if needed)
git branch -M main

# Push to GitHub
git push -u origin main
```

**Authentication:**
- GitHub will ask for username and password
- **Important:** You need a Personal Access Token (PAT), not your password
- Follow the next step to create one

### Step 4: Create Personal Access Token (PAT)
1. Go to GitHub → Click your profile picture → Settings
2. Scroll down → Click "Developer settings" (bottom left)
3. Click "Personal access tokens" → "Tokens (classic)"
4. Click "Generate new token" → "Generate new token (classic)"
5. Fill in:
   - **Note:** "ALDF-EAM repository access"
   - **Expiration:** 90 days (or custom)
   - **Scopes:** Check ✅ `repo` (all sub-items)
6. Click "Generate token"
7. **COPY THE TOKEN IMMEDIATELY** (you won't see it again!)
8. Use this token as your password when pushing

### Step 5: Push Again with PAT
```bash
git push -u origin main
```
- Username: `YOUR_USERNAME`
- Password: `paste_your_token_here`

**Success!** Your repository is now on GitHub! 🎉

---

## Part 5: Verify and Customize (5 minutes)

### Step 1: Check GitHub
1. Go to `https://github.com/YOUR_USERNAME/ALDF-EAM`
2. You should see your README.md displayed automatically
3. Check that all files are present

### Step 2: Update README.md Placeholders
Edit `README.md` and replace:
- `YOUR_USERNAME` → Your actual GitHub username
- `YOUR_EMAIL` → Your email
- `YOUR_LINKEDIN` → Your LinkedIn URL
- `YOUR_FILING_DATE` → Patent filing date

Then update:
```bash
git add README.md
git commit -m "Update contact information in README"
git push
```

### Step 3: Add Topics (Tags)
On GitHub repository page:
1. Click ⚙️ (gear icon) next to "About"
2. Add topics (helps discoverability):
   - `ami`
   - `smart-grid`
   - `ntl-detection`
   - `electricity-theft`
   - `machine-learning`
   - `bayesian-inference`
   - `india`
   - `utility-analytics`
3. Save changes

---

## Part 6: Add Colab Badge (5 minutes)

### Step 1: Upload Notebook to GitHub
Make sure your `ALDF_EAM_Implementation.ipynb` is in the repository.

### Step 2: Get Colab Link
The badge in README.md has:
```markdown
[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)]
(https://colab.research.google.com/github/YOUR_USERNAME/ALDF-EAM/blob/main/ALDF_EAM_Implementation.ipynb)
```

Replace `YOUR_USERNAME` with your actual username.

### Step 3: Test the Badge
1. Commit and push the updated README
2. Click the Colab badge on your GitHub repo
3. It should open the notebook in Google Colab

---

## Part 7: Optional Enhancements (10-30 minutes)

### Add a Logo/Banner
Create a simple banner image:
1. Use Canva or similar (free)
2. Create 1200x300px image with text "ALDF-EAM"
3. Save as `images/banner.png`
4. Add to repository
5. Update README.md top:
```markdown
![ALDF-EAM Banner](images/banner.png)
```

### Create a GitHub Pages Website
1. Go to repository Settings
2. Scroll to "Pages"
3. Source: Deploy from branch "main"
4. Folder: `/docs` or `/root`
5. Save
6. Your repo will be live at: `https://YOUR_USERNAME.github.io/ALDF-EAM/`

### Add Shields/Badges
Add more badges to README:
```markdown
![Python](https://img.shields.io/badge/Python-3.8+-blue.svg)
![License](https://img.shields.io/badge/License-MIT-green.svg)
![Status](https://img.shields.io/badge/Status-Alpha-yellow.svg)
```

---

## Part 8: Making Future Updates (Ongoing)

### Adding New Files
```bash
# Navigate to your project folder
cd ALDF-EAM

# Make your changes (edit files, add new files, etc.)

# Check what changed
git status

# Add specific file
git add filename.py

# Or add all changes
git add .

# Commit with descriptive message
git commit -m "Add Method M7 implementation"

# Push to GitHub
git push
```

### Updating Existing Files
```bash
# Edit the file
# Save changes

git add filename.md
git commit -m "Update documentation with deployment examples"
git push
```

### Creating Branches (for experimental features)
```bash
# Create new branch
git checkout -b feature/new-detection-method

# Make changes, commit
git add .
git commit -m "Implement experimental Method M7"

# Push branch to GitHub
git push -u origin feature/new-detection-method

# On GitHub, create Pull Request to merge into main
```

---

## Troubleshooting

### "Permission denied (publickey)"
→ You're using SSH instead of HTTPS. Use HTTPS URL:
```bash
git remote set-url origin https://github.com/YOUR_USERNAME/ALDF-EAM.git
```

### "fatal: not a git repository"
→ You're not in the project folder:
```bash
cd ALDF-EAM
```

### "Everything up-to-date" but changes not pushed
→ You forgot to commit:
```bash
git add .
git commit -m "Describe your changes"
git push
```

### Large file error (>100MB)
→ Don't commit large datasets directly. Use:
1. Git LFS (Large File Storage), or
2. Link to external dataset (Google Drive, Zenodo)

---

## Quick Reference

### Most Common Commands
```bash
# Check status
git status

# Add all changes
git add .

# Commit
git commit -m "Your message here"

# Push to GitHub
git push

# Pull latest changes (if working from multiple computers)
git pull

# See commit history
git log --oneline
```

---

## Next Steps After Repository is Live

1. ✅ **Share on LinkedIn:** Post about open-source release
2. ✅ **Tweet/post:** Use hashtags #SmartGrid #AMI #OpenSource
3. ✅ **Submit to Awesome Lists:** 
   - Awesome Smart Grid
   - Awesome Machine Learning
   - Awesome India
4. ✅ **Register on Research Repositories:**
   - Papers with Code
   - Hugging Face (if you add models)
5. ✅ **Email colleagues** in utility sector

---

## Success Checklist

Before announcing your repository, verify:

- [ ] README.md displays correctly on GitHub
- [ ] Colab badge opens notebook successfully
- [ ] LICENSE file is present
- [ ] All placeholder text updated (YOUR_USERNAME, YOUR_EMAIL, etc.)
- [ ] requirements.txt has all dependencies
- [ ] .gitignore prevents sensitive data from being committed
- [ ] Repository topics/tags added
- [ ] At least 3-5 commits (shows active development)

---

## You're Done! 🎉

Your ALDF-EAM repository is now live on GitHub and ready for:
- ⭐ Stars from the community
- 🍴 Forks from other researchers
- 🤝 Contributions from developers
- 📊 Pilot deployments from utilities
- 📝 Citations in academic papers

**Remember:** GitHub is a living document. Keep it updated as you:
- Publish the BES paper
- Complete pilot deployments
- Receive patent publication number
- Add new features

Good luck with your open-source journey! 🚀
