### tags: #git
```bash
git rebase -i commit_hash
```
or
```sh
git rebase -i main
```
or
```sh
git rebase -i HEAD~N
```

Change all **pick** words to **s** (squash), starting from HEAD (latest commit)
```sh
pick -> s
```