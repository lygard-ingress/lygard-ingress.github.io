# lygard-ingress.github.io

Source of the Lygard documentation site, https://lygard-ingress.github.io.

The pages under `docs/` and `mkdocs.yml` are generated from the Lygard repository by
`tools/docs-site/build.py`; change them there and regenerate rather than editing here. Every push
to `main` builds the site with MkDocs (`--strict`) and publishes it with GitHub Pages.

Build locally:

```sh
pip install -r requirements.txt
mkdocs serve
```

Content is licensed under the Apache License 2.0, like Lygard itself.
