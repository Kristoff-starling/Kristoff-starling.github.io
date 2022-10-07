---
title: Slides
authors: []
tags: []
categories: []
date: "2022-10-06"
slides:
  # Choose a theme from https://github.com/hakimel/reveal.js#theming
  theme: serif  
  # Choose a code highlighting style (if highlighting enabled in `params.toml`)
  #   Light style: github. Dark style: dracula (default).
  highlight_style: dracula
---

## TF with Sanitizers

---

### Address/Memory/UB Sanitizer

* asan/msan: build fail (GCC/Clang) 
* ubsan: stuck during linking

<p align=left>GitHub issue (#50892): "Currently we don't officially support an OSS ASAN build, although one is in the long term roadmap."</p>
<p align=right>Aug 2, 2021</p>

---

### Compute Sanitizer

* Don't need additional flags for compilation, even a binary version works (?)
* Command: `compute-sanitizer --tool memcheck python3 test.py`
  ```python
  import tensorflow as tf
  _ = tf.config.list_physical_device('GPU')
  ```
  => terminate w/o error
  
  ```python
  import tensorflow as tf
  _ = tf.abs(1)
  ```
  => deadlock