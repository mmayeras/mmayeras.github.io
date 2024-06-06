---
title: "hugo-learn config"
date: 2023-03-24
tags:
- hugo
categories:
- misc
draft: true
---


I recently switch to the LoveIt theme
Here is my previous config using hugo-learn theme

```toml
baseURL = 'http://mmayeras.github.io'
languageCode = 'en-us'
title = "mmayeras's notes"
theme = 'hugo-theme-learn'

[outputs]
home = [ "HTML", "RSS", "JSON"]

[params]
themeVariant = "red"
showvisitedlinks = false
disableMermaid = true

[[menu.shortcuts]] 
name = "<i class='fab fa-github'></i> Github repo"
identifier = "ds"
url = "https://github.com/mmayeras"
weight = 10

[[menu.shortcuts]]
name = "<i class='fas fa-bookmark'></i> Linkedin"
url = "https://www.linkedin.com/in/mickaelmayeras"
weight = 11
```
