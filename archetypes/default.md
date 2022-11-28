---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
draft: false

typora-root-url: ../../static
typora-copy-images-to: ../../static/images

categories: ["{{ trim (replace .File.Dir "posts/" "") "/" }}"]
---