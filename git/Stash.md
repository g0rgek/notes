To _save_ a stash with a message:

```
git stash push -m "my_stash_name"
```

---

To _list_ stashes:

```
git stash list
```

All the stashes are stored in a stack.

---

To _pop_ (i.e. apply and drop) the `n`th stash:

```
git stash pop stash@{n}
```

To _pop_ (i.e. apply and drop) a stash by name is not possible with `git stash pop` (see footnote-1).

---

To _apply_ the `n`th stash:

```
git stash apply stash@{n}
```

**To _apply_ a stash by name**:

```
git stash apply stash^{/my_stash_name}
```
