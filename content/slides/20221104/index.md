---
title: Slides
authors: []
tags: []
categories: []
date: "2022-11-03"
slides:
  # Choose a theme from https://github.com/hakimel/reveal.js#theming
  theme: serif  
  # Choose a code highlighting style (if highlighting enabled in `params.toml`)
  #   Light style: github. Dark style: dracula (default).
  highlight_style: dracula
---

## TF/Pytorch with Sanitizers

---

## PyTorch

---

### Summary
* 1085 APIs are tested.
* ASan+UBSan / Compute Sanitizer 
* Two kinds of false positives are filtered:
  * Invalid argument error
  * Out-of-memory allocations

---

### Results

|          | #APIs  |    #Bugs    |
| :------: | :----: | :---------: |
| ASan     |   7    |      5      |
| UBSan    |   19   |      8      |
| CSan     |   2+     |     2+        | 

---

* ASan: 
  * heap-buffer-overflow
  * heap use-after-free
  * SEGV on unknown address
* UBSan:
  * signed integer overflow
  * non-zero offset to null pointer
  * negative shift exponent
* CSan:
  * Warp assertion
  * CudaInvalidConfiguration

---

## TensorFlow

---

### Results

|          | #APIs  |    #Bugs    |
| :------: | :----: | :---------: |
| CSan     |   2+    |      2+      |

* Out-of-bound reads