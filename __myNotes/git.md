# Git

# Squash last N commits
```sh
m="$(git log -1 --pretty=%B)"; git reset --soft HEAD~$N && git commit -m"$m"
```