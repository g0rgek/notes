# Change last commit message
```sh
git commit --amend
```
# Change oldest commit message
```sh
git rebase -i main
```
or
```sh
git rebase -i HEAD~N
```
N - count of commits from the latest
```sh
pick -> reword
```
# Change default git editor (must be in PATH)

```shell
git config --global core.editor "code --wait"
```

```sh
git config --global core.editor "nvim"
```
