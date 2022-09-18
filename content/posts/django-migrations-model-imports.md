---
title: "Avoid breaking your Django migrations accidentally"
date: 2022-09-18T19:18:14+01:00
draft: false
featured_image: '/images/django.png'
---

Django migrations can be very powerful, and allow you to effectively execute arbitrary code to modify your database, by migrating your database from one schema to another. A common pattern that beginners in Django often get wrong is to do something like:

```python3
from app import MyModel

def my_migration(apps, schema_editor):
    objects = MyModel.objects.all()
    # Some code that then modifies the objects...
```

What is wrong with this is not immediately obvious. When you run code like this the first time, it will likely certainly work correctly, and you'd be inclined to think that it's correct. The problem comes when you later try and spin up an instance of your server, either in testing or in production where you have a clean database. If `MyModel` has been modified since your migration was written, you may well find that the migration won't run - it can't find properties on the model because they may no longer exist, have been renamed, or removed entirely. The solution to doing this is to replace the direct import of the Model as follows:

```python3
def my_migration(apps, schema_editor):
    MyModel = apps.get_model("app", "MyModel")
    objects = MyModel.objects.all()
    # Some code that then modifies the objects
```

By doing this, Django knows to import the **version** of the model that was used at the time that the migration was written, rather than the version that exists today. It's relatively trivial to do this after the fact, but why allow it into the code base in the first place? Writing a test using the abstract syntax tree (ast) library allows you to stop an accidental introduction of this:

```python3
import os
import ast
import glob
from collections import namedtuple
from django.conf import settings

Import = namedtuple("Import", ["module", "name", "alias"])

def get_imports(path):
    with open(path) as fh:
        root = ast.parse(fh.read(), path)

    for node in ast.iter_child_nodes(root):
        if isinstance(node, ast.Import):
            module = []
        elif isinstance(node, ast.ImportFrom):
            module = node.module.split(".")
        else:
            continue

        for n in node.names:
            yield Import(module, n.name.split("."), n.asname)


def test_check_migrations_import_apps_instead_of_directly():
    """
    Check that Django migrations use the pattern:

    apps.get_model("appname", "ModelName")
    
    instead of

    from appname.models import ModelName
    """
    migrations = glob.glob(os.path.join(settings.BASE_DIR, "migrations", "*.py"))

    forbidden_imports = [{"app", "models"}]

    for migration in migrations:
        imported_libs = get_imports(migration)
        for imported_lib in imported_libs:
            assert not any(
                [forbidden.issubset(set(imported_lib.module)) for forbidden in forbidden_imports]
            )
```
