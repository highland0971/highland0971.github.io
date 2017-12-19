---
Alayout: post
title: "Allow HTTP Redirect for curl"
date: 2017-12-19 07:37:01 +0800
tags: ["curl","linux"]
---

When fetching cloud hosted files, we always encounter redirection for the download link, which is not downloadable by `curl` with default option.

To allow `curl` to download redirected pages/links you have to add `-L` option, like :
```bash
$ curl -L -o kubernetes-v1_9_0.tar.gz https://github.com/kubernetes/kubernetes/releases/download/v1.9.0/kubernetes.tar.gz
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   607    0   607    0     0    238      0 --:--:--  0:00:02 --:--:--   238
100 2805k  100 2805k    0     0  56805      0  0:00:50  0:00:50 --:--:--  111k
```

