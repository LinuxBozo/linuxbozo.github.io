---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
draft: true

slug: {{ .File.BaseFileName }} # Will take the filename as the slug. 

# Lists
keywords:
tags:
---
