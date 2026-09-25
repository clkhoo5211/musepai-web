# musepai-web

The MusePAI web build. The same bundle is served from two places:

| | address | notes |
|---|---|---|
| GitHub Pages | <https://clkhoo5211.github.io/musepai-web/> | built from this repository, `base-href` `/musepai-web/` |
| here.now | <https://blissful-hazel-kkr9.here.now/> | a mirror published straight from `build/web`, `base-href` `/`, SPA fallback on |

They are separate builds, because the base href is compiled in: a bundle
built for one path loads nothing at the other.

**This repository holds the compiled app and nothing else.** No music, no
metadata, no credentials — those live in a private repository and behind the
backend.

The backend address is compiled into the bundle and is therefore public,
which is why it names the origins it accepts instead of allowing `*` — with
`*` it would be an open proxy for anyone who read the JavaScript. Both
addresses above are on that list (`CORS_ALLOW_ORIGIN` on Vercel); a new one
serves the app but cannot reach the API until it is added there too.

```
GitHub Pages (this)            Vercel (sin1)              GitHub (private)
  the app          ──────→  /api/ncm       Netease
                   ──────→  /api/unlock    resolve + stream
                   ──────→  /api/generated ──────→  musepai-library
                                                      metadata in the tree
                                                      audio in releases
```

Rebuilt with `MusePAI/deploy/publish-web.sh` in the main repository; nothing
here is edited by hand. That script publishes the Pages copy only — the
here.now mirror is published separately from `build/web` built with
`--base-href "/"`.
