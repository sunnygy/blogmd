---
title: Introduce CLASS torch.NN.Module
---

## 1. why our module must inherit the NN.Model

**Base class for all neural network modules. Your models should also subclass this class.** 

**Example:**

```python
class XX(torch.nn.Module):
    def __init__(self):
        super().__init__()
```

