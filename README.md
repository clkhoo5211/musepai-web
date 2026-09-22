# musepai-web

The MusePAI web build, served by GitHub Pages at
<https://clkhoo5211.github.io/musepai-web/>.

**This repository holds the compiled app and nothing else.** No music, no
metadata, no credentials — those live in a private repository and behind the
backend.

The backend address is compiled into the bundle and is therefore public,
which is why it only accepts this origin: with `*` it would be an open proxy
for anyone who read the JavaScript.

```
GitHub Pages (this)            Vercel (sin1)              GitHub (private)
  the app          ──────→  /api/ncm       Netease
                   ──────→  /api/unlock    resolve + stream
                   ──────→  /api/generated ──────→  musepai-library
                                                      metadata in the tree
                                                      audio in releases
```

Rebuilt with `MusePAI/deploy/publish-web.sh` in the main repository; nothing
here is edited by hand.
