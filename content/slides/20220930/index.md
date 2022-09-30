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

## Scalability

---

### Setup

* 1000 times for each API (FreeFuzz's standard)
* Including invalid inputs
* Total coverage (including C++'s libraries)

---

### Coverage

* 209 APIs were tested.
* Approximately 150 new lines per API.
* 4158 new lines are covered, 567 of which are "completely new".

---

### Speed

* Approximately 5 min per API (with coverage test) :(
* Several seconds per API (generate tests only)

---

## Optimization

---

<p align=left>Three strategies:</p>

* Successful-execution path:
  * Leverage FreeFuzz's mutation strategies. (working)
  * Non-aggresive argument generation (with learned specifications)
* Error-handling path:
  * Aggresive argument generation (no specification)

---

### Atheris Test Framework

<p align=left>Example: tf.tile</p>

```python
def TestOneInput(data):
    fh = FuzzingHelper(data)
    aggresive = False if fh.random_dice(0.3) else True
    follow_freefuzz = False if fh.random_dice(0.6) else True
    arg_0_tensor = get_argument_arg_0_tensor(
        name='arg_0_tensor', fh, 
        aggresive=aggresive, follow_freefuzz=follow_freefuzz)
    arg_0 = tf.identity(arg_0_tensor)
    arg_1 = get_argument_arg_1(
        name='arg_1', fh, 
        aggresive=aggresive, follow_freefuzz=follow_freefuzz)
    _ = tf.tile(arg_0, arg_1,)
```

---

### Atheris Test Framework (cont'd)

```python
def get_argument_arg_0_tensor(name, fh, aggresive=False, follow_freefuzz=False):
    global argument_dict
    if name not in argument_dict:
        follow_freefuzz = False
    if follow_freefuzz is True:
        res = fh.mutate(argument_dict[name])
    elif not aggresive:
        res = fh.get_random_tensor(shape=None, 
            dtype_set=[tf.float16, tf.float32, tf.float64, tf.int32, tf.int64], 
            min_size=3, max_size=3)
    else:
        res = fh.get_random_tensor()
    argument_dict[name] = res
    return res
```