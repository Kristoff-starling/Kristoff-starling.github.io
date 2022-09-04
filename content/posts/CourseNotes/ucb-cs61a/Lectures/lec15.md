---
title: "UCB-CS61A Lecture 15: Syntax"
linktitle: '15: 语法'
type: docs
draft: false

math: true

menu:
    cs61a-lec:
        parent: Contents
        weight: 15
---

## Syntax Tree

中间的节点称为 non-terminal，叶子节点称为 terminal，一个 syntax tree 描述了一个句子的句法结构。

我们可以使用上一课中提到的嵌套列表的方法作为 syntax tree 这一抽象的实现：

```python
def phrase(tag, branches):
    return tree(tag, branches)

def word(tag, text):
    return tree([tag, text])

def text(word):
    return label(word)[1]

def tag(t):
    """Return the tag of a phrase or word."""
    if is_leaf(t):
        return label(t)[0]
    else:
        return label(t)
```

## Generating Sentences

一个 language model 描述了每种文本出现的概率。我们可以使用 sampling 的方法根据一个 language model 来生成语言。

language model 中有很多 syntax tree，树上的每个节点都被打了标签。我们首先遍历所有的树来获取每种 tag 的节点列表：

```python
def index_trees(trees):
    index = {}
    for t in trees:
        for tag, node in nodes(t):
            if tag not in index:
                index[tag] = []
            index[tag].append(node)
    return index
trees = [tokens_to_parse_tree(s) for s in all_sentences()]
tree_index = index_trees(trees)
```

接下来我们可以根据 index 列表来生成一句新的话：我们从一棵已有的 syntax tree 开始，每次随机决定是否将一个子树换成 tag 相同的另一个子树：

```python
def gen_tree(t, tree_index, flip):
    new_branches = []
    if is_leaf(t):
        return t
    for b in branches(t):
        if flip():
            b = random.choice(tree_index[tag(b)])
        new_branches.append(gen_tree(b, tree_index, flip))
    return phrase(tag(t), new_branches)
```

