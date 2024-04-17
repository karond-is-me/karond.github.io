本篇中你将看到：
- 从零使用PyO3创建一个Python模块


# Round1 使用PyO3创建第一个Rust语言的Python模块
## 准备工作
首先要安装Python和Rust，PyO3官方的版本要求是：
- Python 3.7 and up (CPython, PyPy, and GraalPy)
- Rust 1.56 and up

我在本机安装的基本是最新版本
```bash
(base) karond:~/code/rust/mypym$ python --version
Python 3.12.1
(base) karond:~/code/rust/mypym$ pip --version
pip 23.3.1 from /home/karond/.miniconda3/lib/python3.12/site-packages/pip (python 3.12)
(base) karond:~/code/rust/mypym$ rustc --version
rustc 1.77.2 (25ef9e3d8 2024-04-09)
(base) karond:~/code/rust/mypym$ cargo --version
cargo 1.77.2 (e52e36006 2024-03-26)
```
## 创建一个项目
先不要急于使用cargo创建一个项目，我们创建的项目最终是要生成一个可以使用pip安装的Python模块，而生成模块可以使用setuptools和maturin，官方推荐maturin因此我们使用pip安装maturin并创建项目，需要选择pyo3项目
```bash
(base) karond:~/code/rust/mypym$ pip install maturin
Looking in indexes: https://pypi.tuna.tsinghua.edu.cn/simple
Collecting maturin
  Downloading https://pypi.tuna.tsinghua.edu.cn/packages/d2/e0/886abd982f4dc1031cc947909e75e9bbbb4c2d76f5ffd23a7236784135af/maturin-1.5.1-py3-none-manylinux_2_12_x86_64.manylinux2010_x86_64.musllinux_1_1_x86_64.whl (10.3 MB)
     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 10.3/10.3 MB 2.7 MB/s eta 0:00:00
Installing collected packages: maturin
Successfully installed maturin-1.5.1
(base) karond:~/code/rust/mypym$ maturin init
? 🤷 Which kind of bindings to use?
  📖 Documentation: https://maturin.rs/bindings.html ›
❯ pyo3
  rust-cpython
  cffi
  uniffi
  bin
```
可以看到生成的项目里有一段实例代码，作用就是创建一个模块模块下面存在一个函数，是让两个数相加并返回结果的字符串
```rust
use pyo3::prelude::*;

/// Formats the sum of two numbers as string.
#[pyfunction]
fn sum_as_string(a: usize, b: usize) -> PyResult<String> {
    Ok((a + b).to_string())
}

/// A Python module implemented in Rust.
#[pymodule]
fn mypym(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(sum_as_string, m)?)?;
    Ok(())
}
```
## 构建模块
使用`maturin build --release`命令构建模块，经过漫长的输出以后最终的输出指出构建好的模块保存在target/wheels文件夹下，模块大小仅有225KB
```bash
karond:~/code/rust/mypym$ maturin build --release
  Downloaded bitflags v1.3.2 (registry `rsproxy-sparse`)
  Downloaded redox_syscall v0.4.1 (registry `rsproxy-sparse`)
...
   Compiling mypym v0.1.0 (/home/karond/code/rust/mypym)
    Finished release [optimized] target(s) in 23.61s
📦 Built wheel for CPython 3.12 to /home/karond/code/rust/mypym/target/wheels/mypym-0.1.0-cp312-cp312-manylinux_2_34_x86_64.whl

karond:~/code/rust/mypym$ ls -lh /home/karond/code/rust/mypym/target/wheels/mypym-0.1.0-cp312-cp312-manylinux_2_34_x86_64.whl
-rw-r--r-- 1 karond karond 225K Apr 17 21:38 /home/karond/code/rust/mypym/target/wheels/mypym-0.1.0-cp312-cp312-manylinux_2_34_x86_64.whl
```

我们就可以使用pip安装这个模块试一试
```bash
(base) karond:~/code/rust/mypym$ pip install target/wheels/mypym-0.1.0-cp312-cp312-manylinux_2_34_x86_64.whl
Looking in indexes: https://pypi.tuna.tsinghua.edu.cn/simple
Processing ./target/wheels/mypym-0.1.0-cp312-cp312-manylinux_2_34_x86_64.whl
Installing collected packages: mypym
Successfully installed mypym-0.1.0
(base) karond:~/code/rust/mypym$ python
Python 3.12.1 | packaged by Anaconda, Inc. | (main, Jan 19 2024, 15:51:05) [GCC 11.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import mypym
>>> mypym.
mypym.mypym           mypym.sum_as_string(
>>> mypym.sum_as_string(1234,4321)
'5555'
```
## 总结
创建第一Rust语言的Python模块就是这么简单