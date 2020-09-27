---
title: Activation Functions Introduce
date: 2020-09-23 22:17:06
tags: Activation
categories: ML Basic
---
## 1.**Mish**

```python
# -*- coding: utf-8 -*-
import torch
import torch.nn as nn
import torch.nn.functional as F
from matplotlib import pyplot as plt

class Mish(nn.Module):
    def __init__(self):
        super().__init__()
        print("Mish activation loaded...")
    def forward(self,x):
        x = x * (torch.tanh(F.softplus(x)))
        return x

mish = Mish()
x = torch.linspace(-10,10,1000)
y = mish(x)

plt.plot(x,y)
plt.grid()
plt.show()
```

<!--more-->

![Mish](https://raw.githubusercontent.com/sunnygy/blogImages/master/img/mish.png)

## 2. Leaky Relu

>  CLASS* `torch.nn.``LeakyReLU`(*negative_slope: float = 0.01*, *inplace: bool = False*)

>  LeakyReLU(x)=max(0,x)+negative_slope∗min(0,x)  
>
> Examples:
>
> ```python
> >>> m = nn.LeakyReLU(0.1)
> >>> input = torch.randn(2)
> >>> output = m(input)
> ```

![LeakyReLU](https://raw.githubusercontent.com/sunnygy/blogImages/master/LeakyReLU.png)