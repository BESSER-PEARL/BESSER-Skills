# GUI Modeling Reference

GUI models define screens, data bindings, and navigation for generators
that produce UI code. Required by `WebAppGenerator` and `FlutterGenerator`,
optional for `DjangoGenerator`.

## Imports

```python
from besser.BUML.metamodel.gui import (
    GUIModel, Module, Screen,
    DataList, DataSourceElement,
    Button, ButtonType, ButtonActionType,
)
```

## Data binding

`DataSourceElement` binds a domain class to a UI element, picking which
fields appear. `DataList` groups one or more sources into a displayable
list.

```python
# Bind the Book class — show title and pages columns
book_source = DataSourceElement(
    name="book_list",
    dataSourceClass=book,        # a Class from your DomainModel
    fields={title, pages},       # Property objects from that Class
)

author_source = DataSourceElement(
    name="author_list",
    dataSourceClass=author,
    fields={author_name, email},
)

book_list   = DataList(name="BookList",   description="Shows books",
                       list_sources={book_source})
author_list = DataList(name="AuthorList", description="Shows authors",
                       list_sources={author_source})
```

## Screens and navigation

Mark exactly one screen as `is_main_page=True` — that is the landing page.

```python
home = Screen(
    name="Home",
    description="Home page",
    view_elements={book_list},
    is_main_page=True,
)

authors_page = Screen(
    name="Authors",
    description="Author management",
    view_elements={author_list},
)
```

If no screen is marked `is_main_page=True`, `DjangoGenerator` prints a
warning and continues (incomplete views); `WebAppGenerator` may render an
empty landing page.

## Assembling the model

```python
module = Module(name="main", screens={home, authors_page})

gui = GUIModel(
    name="MyApp",
    package="com.example",
    versionCode="1",
    versionName="1.0",
    description="Library app",
    modules={module},
)
```

## Using with WebAppGenerator

```python
from besser.generators.web_app import WebAppGenerator

gen = WebAppGenerator(
    model=domain_model,   # your DomainModel
    gui_model=gui,        # the GUIModel above
    output_dir="./webapp",
)
gen.generate()
# Run with: cd webapp && docker-compose up --build
```

## Gotchas

- **No main page**: at least one `Screen` must have `is_main_page=True`.
- **Field references**: `fields={...}` must reference `Property` objects
  from the bound class, not just names. Reuse the `Property` instances you
  built when defining the class.
- **Class missing from DomainModel**: the class referenced by
  `dataSourceClass` must be in `domain_model.types`. Otherwise the
  generated app cannot resolve it.
