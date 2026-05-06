Run this command whenever cloning blog repo from scratch:
```bash
git submodule update --init --recursive # needed when you reclone your repo (submodules may not get cloned automatically)
```

Update theme with:
```bash
git submodule update --remote --merge
```