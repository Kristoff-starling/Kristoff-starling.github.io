---
title: Slides
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

## Atheris Automation

---

## Methodology

---

* Use FreeFuzz's tests as templates
* Substitute concrete values with appropriate functions
    ```python
    arg_1 = 56          --> arg_1 = fh.get_int()
    arg_1 = [-1.0, 6.0] --> arg_1 = fh.get_float_list()
    padding = 'VALID'   --> padding = fh.get_string(type=padding)
    ```
* An Example:
    ```python
    def TestOneInput(data):
        fh = FuzzingHelper(data)
        arg_0_tensor = fh.get_random_tensor(
            shape=None,
            dtype_set=[tf.float16, tf.float32, tf.float64, 
                       tf.int32, tf.int64],
            min_size=1,
            max_size=8)
        arg_0 = tf.identity(arg_0_tensor)
        dtype = tf.float16
        _ = tf.cast(arg_0,dtype=dtype,)
    ```

---

<p align=left>Issue: lots of APIs contains hidden specifications.</p>
<p align=left>Example: tf.nn.conv2d()</p>

* Specification:
    * The input tensor may have rank 4 or higher.
    * `padding` should be in `{'VALID', 'SAME'}`
* A dummy test such as
    ```python
    arg_0 = fh.get_random_tensor()
    arg_1 = fh.get_random_tensor()
    strides = fh.get_int()
    padding = fh.get_string()
    _ = tf.nn.conv2d(arg_0,arg_1,strides=strides,padding=padding,)
    ```
  will fail.

---
<p align=left>Solution: use FreeFuzz to find specifications through trials and errors.</p>

<p align=left>Example:</p>

* Goal: Identify whether the API accepts negative inputs.
* Steps:
    * Obtain a valid FreeFuzz test.
    * Substitute the argument with "-1".
    * Execute the modified tests and try to catch exceptions.

---

* `int/float`: value range
* `int/float` list: length, negative value
* `string`: special names (reduction/padding/activation/channel)
* Tensor: dtype, shape(length, value range)

<p align=left>Sometimes, two lists/tensors are required to have the same length/dimension. In this situation, we analyze the structure of an invocation and try to add restrictions on arguments on the same level.

---

## Results

---

### Success Rate

<p align=left>(Here success means that the test terminates without "InvalidArgument" errors.)</p>

* Before specification learning: <50%
* After specification learning: 478/533, 89.6%

---

### Coverage

<p align=left>5 untrivial APIs were selected for coverage test:</p>

* `tf.random.stateless_parameterized_truncated_normal`
* `tf.optimizers.schedules.ExponentialDecay`
* `tf.keras.layers.SpatialDropout3D`
* `tf.keras.layers.Convolution3DTranspose`
* `tf.keras.initializers.LecunNormal`

<p align=left>150 tests were generated for each API</p>

---

### Coverage

<p align=left>The dummy test</p>

```python
import tensorflow as tf
```

<p align=left>covers 70005 lines. Besides that,</p>

* FreeFuzz's tests cover 1151 lines.
* Atheris's tests cover 1040 lines.
* 1038 of which are common.

---

### Current Problems

* We cannot learn complicated specifications through simple experiments.
    * FreeFuzz itself is not very stable: some tests rely on random seeds to run normally.
* We haven't found bugs through these tests.
    * Some OOM bugs have been caught but it seems that they are false positive bugs due to my local machine limitations.