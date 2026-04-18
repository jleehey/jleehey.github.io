Files in this folder:
- `index.html` — landing page for Simo's bachelor trip
- `styles.css` — minimal styling

How to add these files into your GitHub Pages repo under a subfolder `sayulita-prayer-for-simo` and open a PR (recommended safe flow):

1) Clone your repo (if you haven't):

```bash
git clone git@github.com:jleehey/jleehey.github.io.git
cd jleehey.github.io
```

2) Create a branch and copy these files into the subfolder:

```bash
git checkout -b bachelor-party
mkdir -p sayulita-prayer-for-simo
# from another terminal or explorer, copy the files into the repo folder
# example (adjust source path):
cp /path/to/workspace/sayulita-prayer-for-simo/* sayulita-prayer-for-simo/
```

3) Add, commit, push, and open a PR:

```bash
git add sayulita-prayer-for-simo
git commit -m "Add Simo bachelor trip landing page (sayulita-prayer-for-simo)"
git push origin bachelor-party
```

4) Open a PR on GitHub from branch `bachelor-party` → `master` (or your default branch). You can create the PR via the GitHub web UI or using `gh` CLI:

```bash
# if you have GitHub CLI installed and authenticated:
gh pr create --title "Add Simo bachelor trip page" --body "Landing page under /sayulita-prayer-for-simo" --base master --head bachelor-party
```

Notes:
- Using a subfolder `sayulita-prayer-for-simo` will NOT replace your current homepage. The new page will be available at:
  https://jleehey.github.io/sayulita-prayer-for-simo/

- If you instead commit `index.html` into the repo root and merge, it WILL replace the site homepage at https://jleehey.github.io/ — only do that if you want to overwrite the existing homepage.

Security: You pasted a GitHub token into the chat. For safety, revoke that token now (Settings → Developer settings → Personal access tokens → Revoke) and create a short-lived token if needed. I will not use the token you pasted here.

If you want, I can:
- Produce a single `.zip` or patch file you can download (I cannot push directly without a token you host locally). 
- Generate the exact `gh` CLI command sequence and PR body — tell me if you want that.