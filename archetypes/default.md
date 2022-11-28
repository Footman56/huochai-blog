---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
draft: true
---


typora-root-url: ../../static
typora-copy-images-to: ../../static/img

categories: ["{{ trim (replace .File.Dir "posts/" "") "/" }}"]