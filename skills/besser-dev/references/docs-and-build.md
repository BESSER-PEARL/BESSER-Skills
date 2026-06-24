# Building Documentation

BESSER docs use Sphinx with the Furo theme. Source files are
reStructuredText (`.rst`) under `docs/source/`.

## Build

```bash
cd docs

# Windows:
make.bat html
# macOS/Linux:
make html

# View: open docs/build/html/index.html
```

## Doc structure

| Path | Contains |
|------|----------|
| `docs/source/buml_language/` | Metamodel documentation |
| `docs/source/generators/` | Generator documentation |
| `docs/source/web_editor.rst` | Web editor API docs |
| `docs/source/utilities/` | Utility documentation |
| `docs/source/contributing/` | Contributor guides |
| `docs/source/releases/` | Release notes |
| `docs/source/_static/` | Shared images and assets |
| `docs/source/img/` | Diagrams and screenshots |

## Cross-references

Use Sphinx roles for cross-linking:
- `:doc:\`generators/django\`` — link to another doc page
- `:mod:\`besser.generators.django\`` — link to module API
- `:class:\`besser.BUML.metamodel.structural.Class\`` — link to class API
- `:ref:\`label-name\`` — link to a labeled section
