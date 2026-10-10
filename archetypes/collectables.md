+++
date = '{{ .Date }}' # when the snippet was written, not today
author = '' # who wrote it; the surname becomes the `ls -l` owner
draft = true
title = '{{ replace .File.ContentBaseName "-" " " | title }}'
+++

One or two sentences on why it's beautiful.

<!-- the language tag below becomes the `ls -l` group -->
```go
```

Source: [where it was found](https://example.com)
