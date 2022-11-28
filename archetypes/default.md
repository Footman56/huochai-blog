---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
draft: false


categories: ["{{ trim (replace .File.Dir "posts/" "") "/" }}"]
---