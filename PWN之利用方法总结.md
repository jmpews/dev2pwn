## 触发恶意函数的方法
#### 代码注入类
栈溢出后在栈上执行代码 - 受NX(DEP) 和 栈保护限制

#### 伪造替换正常函数
修改正常函数 `.got.plt` 地址, 指向恶意函数.

`ret_2_dl_resolve`