# 缘起

今天公司的顾问写了一段Python用于大量作图并保存在本地，但是运行了一段时间后出现快速的内存泄露，本文还原内存泄露发现的过程。
# 建议
如果不想看过程，使用这些建议可能会解决90%的Matplotlib后台绘制导致的内存泄漏问题

1. 使用`matplotlib.figure.Figure`代替`matplotlib.pyplot.figure`
2. 保存图片用`figure.savefig`代替`plt.savefig`
3. 使用非互动式后端 `matplotlib.use('agg')`
4. 及时关闭绘制资源`figure.clf()` `plt.close(figure)`
# 过程
发生的环境是：
```
python==3.7.7
matplotlib==3.5.3
numpy==1.21.1
pandas==1.3.5
seaborn==0.12.2
```
代码涉及商业代码我就用一段测试代码代替了：
```python
import pandas as pd  
import numpy as np  
import seaborn as sns  
import matplotlib.pyplot as plt 
import os 
import gc
import uuid
import time
import timeit


class ChartHelper:
    def createImg(self,chart_key, data, folderPath_Scatter):
        df = pd.DataFrame(data) 
        self.createScatterImg(chart_key,  df,folderPath_Scatter)
        plt.close('all')
        df=None
        data=None
         
    def createScatterImg(self,chart_key,df, folderPath):
        
        num_series = df.shape[1]  
        print("scatter")
        figure = plt.figure(figsize=(12, 8))  # 你可以根据需要调整图形大小  
        grid = plt.GridSpec(num_series, num_series, figure=figure, hspace=0.1, wspace=0.1)  # 调整间距  
        # 绘制散点图  
        for i in range(num_series):  
            for j in range(num_series):  
                ax = figure.add_subplot(grid[i, j])  
                ax.scatter(df.iloc[:, j], df.iloc[:, i],s=3,color='black', alpha=0.5)  
                ax.set_xticks([])  # 隐藏x轴刻度  
                ax.set_yticks([])  # 隐藏y轴刻度  
                if i == num_series - 1:  # 最下方的图显示x轴标签  
                    ax.set_xlabel(df.columns[j])  
                if j == 0:  # 最左侧的图显示y轴标签  
                    ax.set_ylabel(df.columns[i], rotation=0, labelpad=15)   
  
       
        # 保存散点矩阵图为图片
        t_start = timeit.default_timer()
        plt.savefig(os.path.join(folderPath, chart_key+'_scatter_matrix.png'),dpi=10)
        figure.clear()
        figure.clf()
        figure=None
        plt.clf()
        plt.close('all')
        gc.collect()
        t = timeit.default_timer() - t_start
        print(t)

        print("heatmap")
        correlation_matrix = df.corr()  
        figure=plt.figure(figsize=(8, 6))  
  
        # 绘制相关性热图
        sns.heatmap(correlation_matrix, annot=True, cmap='coolwarm')  
  
       
        # 保存相关性热图为图片
        t_start = timeit.default_timer()
        plt.savefig(os.path.join(folderPath, chart_key+'_correlation_heatmap.png'),dpi=10) 
        figure.clear()
        figure.clf()
        figure=None
        plt.clf()
        plt.close('all')
        gc.collect()
        t = timeit.default_timer() - t_start
        print(t)


if __name__ == "__main__":
    np.random.seed(0)  
    data = np.random.rand(4, 8)  
    df = pd.DataFrame(data) 
    chart = ChartHelper()
    for i in range(5):
        for j in range(50):
            print(i,j)
            chart.createImg(data=df,folderPath_Scatter="download",chart_key=str(uuid.uuid4()))
        time.sleep(20)
```

顾问原本是写C#的，不知道聪明的你有没有看出问题在哪里。其实罪魁祸首都藏在createScatterImg方法中。
我呢，直接上三板斧

## 第一步 确定是否存在泄露

安装工具
`pip install memory_profiler`
在犯罪嫌疑人createScatterImg加上装饰器[@profile ](/profile ) 
运行测试`mprof run plot`查看结果`mprof plot`
![image.png](https://cdn.nlark.com/yuque/0/2024/png/1289688/1714059087452-0828a942-8c70-48b6-b79a-c69b2f8fb535.png#averageHue=%23f9f9f9&clientId=u67375008-99e2-4&from=paste&height=726&id=u93f71269&originHeight=1013&originWidth=1785&originalType=binary&ratio=1.3953488372093024&rotation=0&showTitle=false&size=242657&status=done&style=none&taskId=u7bcff157-b22b-45d0-810a-3dc851a0706&title=&width=1279.25)可以看到每一轮运行内存都稳定增长，肯定是有部分资源一直没有得到释放
## 第二步找到泄漏点
上 fil-profile`pip install filprofiler`
开始分析`fil-profile run plot.py`
目录下多了一个文件夹，打开html文件查看结果
![peak-memory.svg](https://cdn.nlark.com/yuque/0/2024/svg/1289688/1714095357673-3397800c-4ebf-44e4-982d-f17ccc04b21e.svg#clientId=ufe8ba211-473b-4&from=paste&height=3976&id=u63636b49&originHeight=3976&originWidth=1200&originalType=binary&ratio=1&rotation=0&showTitle=false&size=474643&status=done&style=none&taskId=ufb52fd4d-341f-49d5-aea5-12dbc097131&title=&width=1200)简单看是使用tk产生了一个窗口估计是进程未结束系统未能回收资源，由于是后台服务，不需要展示，所以想办法抑制窗口的创建
将所有创建figure的地方添加frameon参数`figure=plt.figure(figsize=(8, 6),frameon=False)  `
不行，这参数名不副实啊
发现官方更加推荐使用`from matplotlib.figure import Figure`来代替`plt.figure`，再次测试一下
因为plt.figure造成的问题没有了，但是`plt.savefig`保存图片的这一句代码依然存在问题
![peak-memory.svg](https://cdn.nlark.com/yuque/0/2024/svg/1289688/1714099199973-fe1861a4-d274-4439-8efb-5678c0318e21.svg#clientId=ufe8ba211-473b-4&from=drop&height=3976&id=ue1a39d0d&originHeight=3976&originWidth=1200&originalType=binary&ratio=1&rotation=0&showTitle=false&size=483826&status=done&style=none&taskId=u9ac64970-b6a9-4a2f-a52f-1a05750a6f4&title=&width=1200)罪魁祸首依然是由tk创建的窗口，但是从文档中没有找到能抑制窗口创建的参数或者替代方法
看一下Matplotlab的源代码，gcf获取一个figure，如果没有的话使用plt.figure创建，淦，又是这个东西
matplotlib/pyplot.py:852
```python
def gcf():
    """
    Get the current figure.

    If there is currently no figure on the pyplot figure stack, a new one is
    created using `~.pyplot.figure()`.  (To test whether there is currently a
    figure on the pyplot figure stack, check whether `~.pyplot.get_fignums()`
    is empty.)
    """
    manager = _pylab_helpers.Gcf.get_active()
    if manager is not None:
        return manager.canvas.figure
    else:
        return figure()
```
我们将`plt.savefig`修改成`figure.savefig`官方推荐用`Figure`来代替`plt.figure`，我听劝
现在`plt.savefig`的问题解决了，但是有有新的了
![peak-memory.svg](https://cdn.nlark.com/yuque/0/2024/svg/1289688/1714100539772-31d09d80-adcb-49e1-9cbe-5847c4fa3582.svg#clientId=ufe8ba211-473b-4&from=drop&height=3976&id=udb56efd3&originHeight=3976&originWidth=1200&originalType=binary&ratio=1&rotation=0&showTitle=false&size=481919&status=done&style=none&taskId=uee967a11-4a88-4b43-a200-a5fb54c4d6a&title=&width=1200)我都无语死了，删除所有的plt.clf()虽然可以解决这个问题，但是所有的问题都指向一个，那就是figure中创建的tk窗口，肯定有方法可以全局抑制窗口创建的办法。
根据网上的资料发现Matplotlib的后端分成了互动式和非互动式

- 互动式: GTK3Agg, GTK3Cairo, GTK4Agg, GTK4Cairo, MacOSX, nbAgg, QtAgg, QtCairo, TkAgg, TkCairo, WebAgg, WX, WXAgg, WXCairo, Qt5Agg, Qt5Cairo
- 非互动式: agg, cairo, pdf, pgf, ps, svg, template

`matplotlib.get_backend()`获取默认的后端：TkAgg！！
设置成agg再试试
![Figure_1.png](https://cdn.nlark.com/yuque/0/2024/png/1289688/1714109621510-9ca13c1d-ce0a-462d-ab13-2fd892a310f9.png#averageHue=%23f7f6f6&clientId=uef2157d9-40cb-4&from=paste&height=540&id=u181edcaa&originHeight=540&originWidth=1260&originalType=binary&ratio=1&rotation=0&showTitle=false&size=110383&status=done&style=none&taskId=uabf7b71e-5744-44c8-96f6-9ca0735ca81&title=&width=1260)稳得很
