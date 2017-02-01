To generate the html for this talk from the RST, use pandoc:

```
pandoc -t slidy -s mocktalk.rst  -o index.html
```

To serve this up, spin up an HTTP server to serve the static file

```
python -m SimpleHTTPServer
```
