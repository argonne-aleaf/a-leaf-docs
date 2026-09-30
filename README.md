# A-LEAF Documentation

Source of the documentation site for the Argonne Large-Scale Electricity Analysis Framework (A-LEAF):
https://argonne-aleaf.github.io/a-leaf-docs/

The A-LEAF code is at https://github.com/argonne-aleaf/a-leaf.

## Build locally
```bash
pip install -r requirements.txt
mkdocs serve
```
Open http://127.0.0.1:8000. Pages are the Markdown files under `docs/`; the navigation is in `mkdocs.yml`.
Pushing to `main` builds and publishes the site with GitHub Pages (`.github/workflows/docs.yml`).

## License
BSD 3-Clause, UChicago Argonne, LLC. See [LICENSE](LICENSE).
