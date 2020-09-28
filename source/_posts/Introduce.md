---
title: Introduce CLASS torch.NN.Module
date: 2020-09-23 22:17:06
tags: torch.NN.Module
categories: ML Basic 
---

## 1.why our module must inherit the NN.Model

<!--more-->

**Base class for all neural network modules. Your models should also subclass this class.** 

**Example:**

```python
class XX(torch.nn.Module):
    def __init__(self):
        super().__init__()
```

