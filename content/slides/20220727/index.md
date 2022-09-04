---
title: Slides
summary: An introduction to using Academic's Slides feature.
authors: []
tags: []
categories: []
date: "2019-02-05T00:00:00Z"
slides:
  # Choose a theme from https://github.com/hakimel/reveal.js#theming
  theme: serif  
  # Choose a code highlighting style (if highlighting enabled in `params.toml`)
  #   Light style: github. Dark style: dracula (default).
  highlight_style: dracula
---

## DL Library Fuzzing

---

## Bug Report

* Source: release notes of latest versions, Github PRs
* Version: TensorFlow 2.8&2.9, PyTorch 1.12
* Amount: 40
* Multi-API triggered bugs: 8 (all in PyTorch)

---

### Typical Bugs
PyTorch, GitHub #73187

<p align=left>Error: unexpected RuntimeError</p>

```python
import torch

grad_output = torch.full((1, 1, 1, 4, 4,), 1, dtype=torch.float64, requires_grad=True)
input = torch.full((5, 5, 5, 5, 5,), 3.5e+35, dtype=torch.float64, requires_grad=True)
grid = torch.full((1, 1, 1, 4, 4,), 1, dtype=torch.float64, requires_grad=True)
interpolation_mode = 0
padding_mode = 0
align_corners = True
res = torch.grid_sampler_3d(input, grid, interpolation_mode, padding_mode, align_corners)
grad_out = torch.zeros_like(res)
torch.autograd.backward(res, grad_tensors=grad_out)
```
`grid_sampler_3d()` + `backward()`

---

### Typical Bugs
PyTorch, GitHub #75781

<p align=left>Error: unexpected warning</p>

```python
import torch

if __name__ == "__main__":

    n = 8
    x = torch.zeros(n).normal_()
    x.requires_grad = True
    z = torch.fft.irfft(x).sum()
    z.backward()
```
`fft()` + `irfft()` + `sum()` + `backward()`

---

### Typical Bugs

PyTorch, GitHub #77245

<p align=left>Error: unexpected RuntimeError</p>

```python
import torch

def fn(input):
    offset = 0
    fn_res = torch.diagonal(input, offset=offset, )
    return fn_res

input = torch.rand([0, 1], dtype=torch.complex128, requires_grad=True)
torch.autograd.gradcheck(fn, (input), check_forward_ad=True, check_backward_ad=False)
```

`gradcheck()` + `diagonal()` + function parameter

---

### Typical Bugs

PyTorch, GitHub #77526
<p align=left>Error: man-made assertion error</p>

```python
a = torch.randn((2, 2), dtype=torch.cfloat).transpose(0, 1)
result = torch.abs(a)
assert a.stride() == b.stride()
```
`transpose()` + `abs()` + `stride()`

---

### Summary

* Linux kernel(system) v.s. DL libraries(tool kit)
* Common bug types
    * Integer overflow, division by zero
    * Out of memory(OOM), Out of bound(OOB)
    * missing validation

---

## Documentation

---

### Documentation Semantics

Learning-based semantics 'understanding' seems inevitable. $\Rightarrow$ NLP work

<p align=left>An interesting bug: PyTorch, GitHub #70657</p>

```python
import torch
assert torch.ones(10)[::2].ravel().is_contiguous() == True
```

<p align=left>The assertion comes from the document: "ravel() returns a <b>contiguous</b> flattened tensor".</p>

---

### Documentation Structure

* The "Python API" module is divided into 54 sections, including `torch`, `torch.nn`, `torch.cuda` etc.

* The `torch.nn` section is divided into 20 subsections, including "Convolutional layers", "Pooling layers", "Padding layers" etc. The section contains 186 APIs in total.

* Subsection "Convolutional layers" includes 14 APIs, offering rich relational information.

---

### Documentation Structure

<p align=left>Drawback: scalability (e.g. TensorFlow's doc only has coarse categories and lots of APIs are sorted in chronological order.)</p>

---

## Fuzzing

---

### Fuzzing perspectives

* Model-level fuzzing: CRADLE, LEMON
* Sub-Model?
* API-level fuzzing: FreeFuzz, DeepREL, DocTer
* Sub-API?

---

### Sub-Model Fuzzing

* Model-level: precison loss, hard for mutation...
* API-level: complicated situations uncovered

<p align=left>
Sub-model level fuzzing serves as an auxiliary approach to cover more cases. The scale of sub-models/combinations of APIs should be small.
</p>

---

#### Example: PyTorch, GitHub #74404
```python
import torch
from torch import nn

class MyModule(nn.Module):
    def __init__(self):
        super().__init__()
        self.module_list = nn.ModuleList([nn.Linear(1,1) for _ in range(10)])
        self.parameter_list = nn.ParameterList([nn.Parameter(torch.zeros(1)) for _ in range(10)])
    def forward(self, x):
        for m in self.module_list:
            x = m(x)
        return x

if __name__ == '__main__':
    model = MyModule()
    optimize = True
    with torch.jit.optimized_execution(optimize):
      a = torch.jit.script(model, 2)
```

---

### Sub-API Fuzzing

Under Python APIs: C++ codes

<p align=left>Idea:</p>

* Most bugs come from missing validations/boundary argument values.
* The propagation chain of a bug: fault $\to$ error $\to$ failure
* #error > #failure

<p align=left>Open the state machine</p>

---

### Example: assertion injections

<p align=left>TensorFlow: CVE-2022-21725 (division by 0)</p>

```diff
int64_t GetOutputSize(const int64_t input, const int64_t filter,
                      const int64_t stride, const Padding& padding) {
+ assert(stride != 0);
  if (padding == Padding::VALID) {
    return (input - filter + stride) / stride;  // what if stride = 0 ?
  } else {  // SAME.
    return (input + stride - 1) / stride;
  }
} 
```

---

### Example: assertion injections

<p align=left>TensorFlow: CVE-2022-21728 (heap OOB)</p>

```diff
static DimensionHandle DimKnownRank(ShapeHandle s, int64_t idx) {
+ assert(-s->dims_.size() <= idx && idx < s->dims_.size());
  CHECK_NE(s->rank_, kUnknownRank);
  if (idx < 0) {
    return s->dims_[s->dims_.size() + idx];
  }
  return s->dims_[idx];
}
```

---

### Example: assertion injections

<p align=left>TensorFlow: CVE-2022-23589 (null pointer dereference)</p>

```diff
NodeDef* mul_left_child = node_map_->GetNode(node->input(0));
NodeDef* mul_right_child = node_map_->GetNode(node->input(1));
+ assert(mul_left_child != NULL && mul_right_child != NULL);
const bool left_child_is_constant = IsReallyConstant(*mul_left_child);
const bool right_child_is_constant = IsReallyConstant(*mul_right_child);
```