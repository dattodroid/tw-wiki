### Install MkDocs on Windows

```cmd
mkdir tw-wiki
cd tw-wiki
python -m venv venv
.\venv\Scripts\activate
pip install mkdocs-material mkdocs-minify-plugin
mkdocs new .
```

### Start MkDocs server on Windows

``` cmd
.\venv\Scripts\activate
mkdocs serve
```