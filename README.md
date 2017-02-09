Brian Weber's unittest.mock talk!
---------------------------------

To generate the HTML for this talk from the asciidoc format, use cdk

```
pip install cdk
```

Serve the slides locally by generating the new "sample.html"
```
cdk -o sample.asc
``` 

Old RestructuredText method using pandoc for generation:
--------------------------------------------------------

To generate the html for this talk from the RST, use pandoc:

```
pandoc -t slidy -s mocktalk.rst  -o index.html
```

To serve this up, spin up an HTTP server to serve the static file

```
python -m SimpleHTTPServer
```
