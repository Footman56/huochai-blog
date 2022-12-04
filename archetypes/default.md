---
title: "{{ replace .Name "-" " " | title }}"
author: "huochai"
date: {{ .Date }}
draft: false

categories: {{  (split (replace .File.Dir "posts/" "") "/") }}
---