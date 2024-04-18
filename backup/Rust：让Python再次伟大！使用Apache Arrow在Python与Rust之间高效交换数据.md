本篇中你将看到：

   - 使用pyarrow在Python和使用PyO3开发的Python模块间传递数据
 
# Round2 使用Apache Arrow在Python和Rust代码之间飞速传输数据

## Apache Arrow
先让GPT来一段
> Apache Arrow 是一个开源软件项目，提供了一种跨语言和系统以高效且可互操作的方式处理大型数据集的标准方法。它提供了一组通用数据结构和序列化格式，使不同语言和工具可以轻松交换数据，而无需进行昂贵的转换或复制。
>
> Apache Arrow 的主要功能包括：
>
>   内存中列式数据格式： Arrow 定义了一种内存中列式数据格式，称为 Arrow 格式。此格式优化了数据存储和处理，因为它将数据存储为按列组织的连续内存块，而不是按行组织。
>   跨语言支持： Arrow 为多种编程语言（包括 Python、Java、C++、R 和 Go）提供了绑定，使开发人员可以在不同的语言之间轻松交换数据。
>   高性能： Arrow 针对高性能进行了优化，它提供了高效的数据读取、写入和处理操作。
>   可扩展性： Arrow 是一个可扩展的框架，允许开发人员创建自定义数据类型和操作。
>
> Apache Arrow 在以下领域有广泛的应用：
>
>    大数据分析： Arrow 可用于处理和分析大数据集，例如在数据仓库和数据湖中。
>   机器学习： Arrow 可用于高效地加载和处理机器学习模型所需的数据。
>   数据可视化： Arrow 可用于将数据快速加载到数据可视化工具中。
>   数据交换： Arrow 可用于在不同的系统和应用程序之间交换数据，而无需进行昂贵的转换。
>
> 总体而言，Apache Arrow 是一个功能强大且易于使用的工具，可用于处理和交换大型数据集，从而简化数据分析和处理任务。

总之，Apache Arrow是一种内存数据的组织形式，目的是为了在不同语言不同系统之间高效的传递数据，Apache Arrow 通过使用称为 零拷贝内存映射 的技术来实现 0 复制。也就是说，我们在Python和Rust代码之间只需要增加一点调用成本，没有数据传输成本！
## 准备工作
Python中我们选择的Apache Arrow实现为pyarrow，而在Rust中我们使用的是arrow-rs，首先将我们需要的包分别安装在我们的环境当中
```bash
# 安装pyarrow
pip install pyarrow
# 添加arrow包
cargo add arrow -F "pyarrow,pyo3"
```
## 使用pyarrow传递数据
然后我们稍微修改一下我们的模块代码,首先用数组数据创建一个Array类型的一个引用，然后将这个Array类型类型转为我们实际的数据类型，最后我们故意使用for循环模拟一下只能使用循环解决问题的情况。
```rust
use std::sync::Arc;

use pyo3::exceptions::PyValueError;
use pyo3::prelude::*;
use  arrow::pyarrow::PyArrowType;
use  arrow::array::{ArrayData,Int64Array,Array,make_array};
/// Formats the sum of two numbers as string.
#[pyfunction]
fn sum_arrow_array(array: PyArrowType<ArrayData>) -> PyResult<i64> {
    let array = array.0;
    let array: Arc<dyn Array> = make_array(array);
    let array: &Int64Array = array.as_any().downcast_ref()
        .ok_or_else(|| PyValueError::new_err("expected int64 array"))?;
    //let k:i64 = array.iter().fold(0, |acc, x| acc + x.unwrap());
    let mut k:i64 = 0;
    for x in array.iter(){
        k = k+ x.unwrap();
    }
    Ok(k)
}

/// A Python module implemented in Rust.
#[pymodule]
fn mypym(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(sum_arrow_array, m)?)?;
    Ok(())
}


```
## 大功告成
最后我们在Python中尝试一下我们的加速器
```python
import mypym
import timeit
import pyarrow as pa
import numpy as np

# 生产一亿条测试数据
test_arr = np.random.randint(0,1000,size=100000000,dtype='int64')

# 转为pyarrow格式
test_arr = pa.array(test_arr,type='int64')

# 测试一下，这都是0拷贝的，一眨眼~
t_start = timeit.default_timer()
mypym.sum_arrow_array(test_arr)
chkt = timeit.default_timer() - t_start
print(chkt)
#0.13071506191045046

# 使用Python循环计算
t_start = timeit.default_timer()
k = 0
for i in atest_arr:
    k += i.as_py()
chkt = timeit.default_timer() - t_start
print(chkt)
#33.00633535394445
```
## 总结
简简单单单线程就将Python循环操作提升250倍！

## 号外
```python
# 使用pyarrow的sum方法
t_start = timeit.default_timer()
test_arr.sum()
chkt = timeit.default_timer() - t_start
print(chkt)
# 0.05785635602660477
```
奉劝大家，像pyarrow这种流行的、基础的库，默认的方案大概率比自己实现的要强，只有遇到UDF(User-defined functions)这种难以优化的情形才考虑DIY。
下期对比两种方案处理数据的性能