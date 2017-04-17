---
draft: true
---

Set up repo:

```sh
cd _git
git clone --bare <url>
# I keep my hooks in _git/hooks, because I tend to recycle them
cd <repo>/hooks
ln -s ../../hooks/<something>-post-receive ./post-receive
```
