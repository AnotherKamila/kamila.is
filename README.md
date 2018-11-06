# WIP

:-)

# Notes for the future me

I am currently using **both** git submodules **and** git subrepo, for hysterical raisins. This is terrible and somebody should fix it one day. For now, I have to:

* do the usual submodule farce
* `git subrepo push --all` in addition to `git push` to update the remote repos
  * hey -- this could be a git hook!
