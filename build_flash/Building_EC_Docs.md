For betrusted-ec, this is necessary for `cargo xtask docs` to work:

```
sudo apt install python3-pip
pip install sphinxcontrib-wavedrom
```

## Initial Error

```
$ sphinx-build -M html build/documentation/ build/documentation/_build
Running Sphinx v1.8.5

Extension error:
Could not import extension sphinxcontrib.wavedrom (exception: No module named 'sphinxcontrib')
```


## After installing sphinxcontrib-wavedrom

```
$ sphinx-build -M html build/documentation/ build/documentation/_build
Running Sphinx v1.8.5
building [mo]: targets for 0 po files that are out of date
building [html]: targets for 13 source files that are out of date
updating environment: 13 added, 0 changed, 0 removed
reading sources... [100%] wifi

Sphinx error:
master file build/documentation/contents.rst not found
```

Fix
```
echo "master_doc = 'index'" >> build/documentation/conf.py
```


## After adding master_doc index to conf.py

```
$ sphinx-build -M html build/documentation/ build/documentation/_build
Running Sphinx v1.8.5
building [mo]: targets for 0 po files that are out of date
building [html]: targets for 13 source files that are out of date
updating environment: 13 added, 0 changed, 0 removed
reading sources... [100%] wifi
looking for now-outdated files... none found
pickling environment... done
checking consistency... done
preparing documents... done
writing output... [100%] wifi
generating indices... genindex
writing additional pages... search
copying static files... done
copying extra files... done
dumping search index in English (code: en) ... done
dumping object inventory... done
build succeeded.

The HTML pages are in build/documentation/_build/html.
```
