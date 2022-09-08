# 1.基本配置
## 导入包和版本查询
  ```Python
  import torch
  import torch.nn as nn
  import torchvision
  print(torch.__version__)# PyTorch version
  print(torch.version.cuda)#Corresponding CUDA version
  print(torch.backends.cudnn.version())#Corresponding cuDNN version
  print(torch.cuda.get_device_name(0))#GPU type
  ```
## 可复现性
在硬件设备（CPU、GPU）不同时，完全的可复现性无法保证，即使随机种子相同。但是，在同一个设备上，应该保证可复现性。具体做法是，在程序开始的时候固定`torch`的随机种子，同时也把`numpy`的随机种子固定。
```Python
np.random.seed(0)
torch.manual_seed(0)
torch.cuda.manual_seed_all(0)
```
## 显卡设置
如果只需要一张显卡
```Python
# Device configuration
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
```
如果需要指定多张显卡，比如0，1号显卡。
```Python
import os
os.environ['CUDA_VISIBLE_DEVICES'] = '0,1'
```
也可以在命令行运行代码时设置显卡：
```
CUDA_VISIBLE_DEVICES=0,1 python train.py
```
清除显存:
```Python
torch.cuda.empty_cache()
```
也可以使用在命令行重置GPU的指令：
```
nvidia-smi --gpu-reset -i [gpu_id]
```
或在命令行可以先使用ps找到程序的PID，再使用`kill`结束该进程
```python
ps aux | grep python
kill -9 [pid]
```
## 设置为cuDNN benchmark模式
Benchmark模式会提升计算速度，但是由于计算中有随机性，每次网络前馈结果略有差异。
```python
torch.backends.cudnn.benchmark = True
```
如果想要避免这种结果波动，设置
```python
torch.backends.cudnn.deterministic = True
```
# 2.张量(Tensor)处理
## 创建张量
```python
#data只能是已知的数据，不能是数据的维度
torch.tensor(data)
#可以是data，也可以是维度
#data:
torch.Tensor(data)
#维度,维度为(1,2,3)
torch.Tensor(1,2,3)
```
在用户手动定义`Tensor`时，参数`requires_grad`默认值是`False`。而在`Module`中的层在定义时，相关`Tensor`的`requires_grad`参数默认是`True`。
## 张量转换
你可以通过访问`dtype`属性来获得张量的数据类型：
```python
short_points.dtype
torch.int16
```
您还可以使用相应的转换方法将张量创建函数的输出转换为正确的类型，例如
```python
double_points = torch.zeros(10, 2).double()
short_points = torch.ones(10, 2).short()
```
或者用更方便的`to`方法：
```python
double_points = torch.zeros(10, 2).to(torch.double)
short_points = torch.ones(10, 2).to(dtype=torch.short)
```
在实现内部，`type`和`to`执行相同的操作，即“检查类型如果需要就转换（`check-and-convert-if-needed`）”，但是`to`方法可以使用其他参数。
你始终可以使用`type`方法将一种类型的张量转换为另一种类型的张量：
```python
points = torch.randn(10, 2)
short_points = points.type(torch.short)
```
## 张量基本信息
```Python
tensor = torch.randn(3,4,5)
print(tensor.type())  # 数据类型
print(tensor.size())  # 张量的shape，是个元组，size()是size类实例
print(tensor.shape)  # 张量的shape,是属性
print(tensor.dim())   # 维度的数量
```
## 切片与索引
将图片设定为`[batch_size, channel, height, width]`的四维矩阵。
```python
a = torch.rand(4, 3, 28, 28)
```
对第一维进行索引,可以认为是第一个图片的三个维度通道的28*28的像素点。:
```python
print(a[0].size())
torch.Size([3, 28, 28])
```
第一个图片的第一个维度通道的28*28的像素点:
```python
print(a[0, 0].size())
torch.Size([28, 28])
```
具体到某一个像素点时:
```python
print(a[0, 0, 2, 3])
tensor(0.4736)
```
取连续的索引:
```python
print(a[:2].shape)
torch.Size([2, 3, 28, 28])
```
同理：
```python
# 1写在：前面，表明从1个通道开始到末尾，,不包括1
print(a[:2, 1:, :, :].shape)
```
当索引出现-1时，要提到一个知识点:
```python
# 默认索引的顺序为[0, 1, 2]，当倒着写时变为[-3, -2, -1]。由于这里取-1，因此为最后一位。
print(a[:2, -1:, :, :].shape)
torch.Size([2, 1, 28, 28])
```
当想隔点取样输出时:
```python
#输出全部batch和channel，对每个高和宽间隔2个点采样
print(a[:, :, 0:28:2, 0:28:2].shape)
torch.Size([4, 3, 14, 14])
#可简化为：
print(a[:, :, ::2, ::2].shape)
```
```python
points[1:]    # 第1行及之后所有行，（默认）所有列
points[1:, :] # 第1行及之后所有行，所有列
points[1:, 0] # 第1行及之后所有行，仅第0列
```
## 命名变量
```Python
# 在PyTorch 1.3之前，需要使用注释
# Tensor[N, C, H, W]
images = torch.randn(32, 3, 56, 56)
images.sum(dim=1)
images.select(dim=1, index=0)

# PyTorch 1.3之后
NCHW = [‘N’, ‘C’, ‘H’, ‘W’]
images = torch.randn(32, 3, 56, 56, names=NCHW)
images.sum('C')
images.select('C', index=0)
# 也可以这么设置
tensor = torch.rand(3,4,1,2,names=('C', 'N', 'H', 'W'))
# 使用align_to可以对维度方便地排序
tensor = tensor.align_to('N', 'C', 'H', 'W')
```
## 数据类型转换
```Python
# 设置默认类型，pytorch中的FloatTensor远远快于DoubleTensor
torch.set_default_tensor_type(torch.FloatTensor)

# 类型转换
tensor = tensor.cpu()
tensor = tensor.float()
tensor = tensor.long()
```
除了`dtype`之外，`PyTorch`张量还具有设备（`device`）的概念，这是在设置计算机上放张量（`tensor`）数据的位置。 通过为构造函数指定相应的参数，可以在`GPU`上创建张量：
```python
points_gpu = torch.tensor([[1.0, 4.0], [2.0, 1.0], [3.0, 4.0]], device='cuda')
```
你可以使用`to`方法将在`CPU`上创建的张量（`tensor`）复制到`GPU`：
```python
points_gpu = points.to(device='cuda')
```
这段代码返回一个具有相同数值数据的新张量，但存储在`GPU`的`RAM`中，而不是常规的系统`RAM`中。
现在数据已经存放在本地的`GPU`中，当在张量上运行数字运算时，你可以看见很好的加速效果。并且，这个新`GPU`支持的张量的类也更改为`torch.cuda`.`FloatTensor`（一开始输入的类型为`torch.FloatTensor`；`torch.cuda.DoubleTensor`等等也存在对应关系）。在大部分样例中，基于`CPU`和`GPU`的张量都公开面向用户相同的`API`，这使得与繁琐数字运算过程中无关的代码的编写更加容易。
如果你的机器拥有多个`GPU`，你可以通过传递从零开始的整数来确定张量分配给哪个GPU，该整数标志着机器上的`GPU`下标：
```python
points_gpu = points.to(device='cuda:0')
```
此时，在`GPU`上执行对张量的任何操作，例如将所有元素乘以一个常数。
```python
points = 2 * points # 在CPU上做乘法
points_gpu = 2 * points.to(device='cuda') # 在GPU上做乘法
```
请注意，当计算结果产生后，`points_gpu`的张量并不会返回到`CPU`。这里发生的是以下三个过程：
1. 将`points`张量复制到`GPU`
2. 在`GPU`上分配了一个新的张量，并用于存储乘法的结果
3. 返回该`GPU`张量的句柄
因此，如果你还想向结果加上一个常量：
```python
points_gpu = points_gpu + 4
```
加法仍然在`GPU`上执行，并且没有信息流到`CPU`（除非您打印或访问得到的张量）。 如果要将张量移回`CPU`，你需要为`to`方法提供一个`cpu`参数：
```python
points_cpu = points_gpu.to(device='cpu')
```
你可以使用速记方法`cpu`和`cuda`代替`to`方法来实现相同的目标
```python
points_gpu = points.cuda() # 默认为GPU0
points_gpu = points.cuda(0)
points_cpu = points_gpu.cpu()
```
值得一提的是，使用to方法时，可以通过提供`device`和`dtype`参数来同时更改位置和数据类型。
## 张量操作
在`torch`模块下可进行张量上和张量之间的绝大多数操作，这些操作也可以作为张量对象的方法进行调用。例如，你可以通过`torch`模块使用先前遇到的`transpose`函数：
```python
a = torch.ones(3, 2)
a_t = torch.transpose(a, 0, 1)
```
或者调用`a`张量的方法：
```python
a = torch.ones(3, 2)
a_t = a.transpose(0, 1)
```
以上两种形式之间没有区别，可以互换使用。需要注意的是：有少量的操作仅作为张量对象的方法存在。你可以通过名称中的下划线来识别它们，例如`zero_`，下划线标识表明该方法是就地（`inplace`）运行的，即直接修改输入而不是创建新的输出并返回。例如，`zero_`方法会将输入的所有元素清零。任何不带下划线的方法都将保持源张量不变并返回新的张量：
```python
a = torch.ones(3, 2)
a.zero_()
a
tensor([[0., 0.],
        [0., 0.],
        [0., 0.]])
```
`torch.tensor()`或者`tensor.clone()`总是会进行数据拷贝，新`tensor`和原来的数据不再共享内存。所以如果你想共享内存的话，建议使用`torch.from_numpy()`或者`tensor.detach()`来新建一个`tensor`, 二者共享内存。
## `torch.Tensor`与`np.ndarray`转换
除了`CharTensor`，其他所有`CPU`上的张量都支持转换为`numpy`格式然后再转换回来。
```Python
ndarray = tensor.cpu().numpy()
tensor = torch.from_numpy(ndarray).float()
tensor = torch.from_numpy(ndarray.copy()).float() # If ndarray has negative stride.
```
## 从只包含一个元素的张量中提取值
```value = torch.rand(1).item()```
## 张量形变
```Python
# 在将卷积层输入全连接层的情况下通常需要对张量做形变处理，
# 相比torch.view，torch.reshape可以自动处理输入张量不连续的情况。
tensor = torch.rand(2,3,4)
shape = (6, 4)
tensor = torch.reshape(tensor, shape)
```
## 张量复制
```Python
# Operation                 |  New/Shared memory | Still in computation graph |
tensor.clone()            # |        New         |          Yes               |
tensor.detach()           # |      Shared        |          No                |
tensor.detach.clone()()   # |        New         |          No                |
```
## 张量拼接
```Python
'''
注意torch.cat和torch.stack的区别在于torch.cat沿着给定的维度拼接，
而torch.stack会新增一维。例如当参数是3个10x5的张量，torch.cat的结果是30x5的张量，
而torch.stack的结果是3x10x5的张量。
'''
tensor = torch.cat(list_of_tensors, dim=0)
tensor = torch.stack(list_of_tensors, dim=0)
```
## 将整数标签转为one-hot编码
```python
# pytorch的标记默认从0开始
tensor = torch.tensor([0, 2, 1, 3])
N = tensor.size(0)
num_classes = 4
one_hot = torch.zeros(N, num_classes).long()
one_hot.scatter_(dim=1, index=torch.unsqueeze(tensor, dim=1), src=torch.ones(N, num_classes).long())
```
## 得到非零元素
```python
torch.nonzero(tensor)               # index of non-zero elements,包含点坐标的列表向量
torch.nonzero(tensor==0)            # index of zero elements
torch.nonzero(tensor).size(0)       # number of non-zero elements
torch.nonzero(tensor == 0).size(0)  # number of zero elements
```
## 判断两个张量相等
```Python
torch.allclose(tensor1, tensor2)  # float tensor
torch.equal(tensor1, tensor2)     # int tensor
```
## 张量扩展
```python
# Expand tensor of shape 64*512 to shape 64*512*7*7.
tensor = torch.rand(64,512)
torch.reshape(tensor, (64, 512, 1, 1)).expand(64, 512, 7, 7)
```
## 矩阵乘法
```python
# Matrix multiplcation: (m*n) * (n*p) * -> (m*p).
result = torch.mm(tensor1, tensor2)

# Batch matrix multiplication: (b*m*n) * (b*n*p) -> (b*m*p)
result = torch.bmm(tensor1, tensor2)

# Element-wise multiplication.
result = tensor1 * tensor2
```
## 计算两组数据之间的两两欧式距离
利用broadcast机制
```python
dist = torch.sqrt(torch.sum((X1[:,None,:] - X2) ** 2, dim=2))
```
## 连续张量
```
points = torch.tensor([[1.0, 4.0], [2.0, 1.0], [3.0, 5.0]])
points
tensor([[1., 4.],
        [2., 1.],
        [3., 5.]])
points_t = points.t()
points_t
tensor([[1., 2., 3.],
        [4., 1., 5.]])
```
你可以轻松地验证两个张量共享同一存储：
```python
id(points.storage()) == id(points_t.storage())
True
```
并且它们的仅仅是尺寸和步长不同：
```python
points.stride()
(2, 1)
points_t.stride()
(1, 2)
```
从最右边的维开始将其值存放在存储中的张量（例如沿着行存放在存储中的二维张量）定义为连续（`Contiguous`）张量。连续张量很方便，因为你可以高效且有序地访问它们的元素而不是在存储中四处跳跃访问。（由于现代`CPU`中内存访问的工作原理，改善数据局部性可提高性能。译者注：即连续张量满足局部性原理）
在前例中，`points`是连续的，但其转置不是：
```python
points.is_contiguous(), points_t.is_contiguous()
(True, False)
```
你可以使用`contiguous`方法从非连续张量获得新的连续张量。 张量的内容保持不变，但步长发生变化，存储也是如此：
```python
points = torch.tensor([[1.0, 4.0], [2.0, 1.0], [3.0, 5.0]])
points_t = points.t()
points_t
tensor([[1., 2., 3.],
        [4., 1., 5.]])
points_t.storage()
 1.0
 4.0
 2.0
 1.0
 3.0
 5.0
[torch.FloatStorage of size 6]
points_t.stride()(1, 2)
points_t_cont = points_t.contiguous()
points_t_cont
tensor([[1., 2., 3.],
        [4., 1., 5.]])
points_t_cont.stride()
(3, 1)
points_t_cont.storage()
 1.0
 2.0
 3.0
 4.0
 1.0
 5.0
[torch.FloatStorage of size 6]
```
请注意，新的存储对元素进行了重组以便逐行存放张量元素。步长也已改变以反映新的布局。
# 3.模型定义和操作
## 计算模型整体参数量
```python
num_parameters = sum(torch.numel(parameter) for parameter in model.parameters())
```
## 查看网络中的参数
可以通过`model.state_dict()`或者`model.named_parameters()`函数查看现在的全部可训练参数（包括通过继承得到的父类中的参数）
```python
params = list(model.named_parameters())
(name, param) = params[28]
print(name)
print(param.grad)
print('-------------------------------------------------')
(name2, param2) = params[29]
print(name2)
print(param2.grad)
print('----------------------------------------------------')
(name1, param1) = params[30]
print(name1)
print(param1.grad)
```
## 类似 Keras 的 model.summary() 输出模型信息（使用pytorch-summary ）
## 模型权重初始化
注意 `model.modules()` 和 `model.children()` 的区别：`model.modules()` 会迭代地遍历模型的所有子层，而 `model.children()` 只会遍历模型下的一层。
```python
# Common practise for initialization.
for layer in model.modules():
    if isinstance(layer, torch.nn.Conv2d):
        torch.nn.init.kaiming_normal_(layer.weight, mode='fan_out',
                                      nonlinearity='relu')
        if layer.bias is not None:
            torch.nn.init.constant_(layer.bias, val=0.0)
    elif isinstance(layer, torch.nn.BatchNorm2d):
        torch.nn.init.constant_(layer.weight, val=1.0)
        torch.nn.init.constant_(layer.bias, val=0.0)
    elif isinstance(layer, torch.nn.Linear):
        torch.nn.init.xavier_normal_(layer.weight)
        if layer.bias is not None:
            torch.nn.init.constant_(layer.bias, val=0.0)

# Initialization with given tensor.
layer.weight = torch.nn.Parameter(tensor)
```
## 提取模型中的某一层
`modules()`会返回模型中所有模块的迭代器，它能够访问到最内层，比如`self.layer1.conv1`这个模块，还有一个与它们相对应的是`name_children()`属性以及`named_modules()`,这两个不仅会返回模块的迭代器，还会返回网络层的名字。
```python
# 取模型中的前两层
new_model = nn.Sequential(*list(model.children())[:2] 
# 如果希望提取出模型中的所有卷积层，可以像下面这样操作：
for layer in model.named_modules():
    if isinstance(layer[1],nn.Conv2d):
         conv_model.add_module(layer[0],layer[1])
```
## 部分层使用预训练模型
注意如果保存的模型是`torch.nn.DataParallel`，则当前的模型也需要是
```python
model.load_state_dict(torch.load('model.pth'), strict=False)
```
## `nn.ModuleList`和`nn.Sequential`
```python
self.linears = nn.ModuleList([nn.Linear(10,10) for i in range(2)])
```
你可以把任意 `nn.Module` 的子类 (比如 `nn.Conv2d`, `nn.Linear` 之类的) 加到这个 `list` 里面，方法和 `Python` 自带的 `list` 一样，无非是 `extend`，`append` 等操作。但不同于一般的 `list`，加入到 `nn.ModuleList` 里面的 `module` 是会自动注册到整个网络上的，同时 `module` 的 `parameters` 也会自动添加到整个网络中。

`nn.ModuleList` 并没有定义一个网络，它只是将不同的模块储存在一起，这些模块之间并没有什么先后顺序可言。

```python
self.block = nn.Sequential(nn.Conv2d(1,20,5),
                                    nn.ReLU(),
                                    nn.Conv2d(20,64,5),
                                    nn.ReLU())
```
`nn.Sequential`，不同于 `nn.ModuleList`，它已经实现了内部的 `forward` 函数，而且里面的模块必须是按照顺序进行排列的，所以我们必须确保前一个模块的输出大小和下一个模块的输入大小是一致的。
```python
model2 = nn.Sequential(collections.OrderedDict([
          ('conv1', nn.Conv2d(1,20,5)),
          ('relu1', nn.ReLU()),
          ('conv2', nn.Conv2d(20,64,5)),
          ('relu2', nn.ReLU())
        ]))
```
一般情况下 `nn.Sequential` 的用法是来组成卷积块 (`block`)，然后像拼积木一样把不同的 `block` 拼成整个网络，让代码更简洁，更加结构化。                  
### 场景１
有的时候网络中有很多相似或者重复的层，我们一般会考虑用 `for` 循环来创建它们，而不是一行一行地写，比如：
```python
layers = [nn.Linear(10, 10) for i in range(5)]
```
这个时候，很自然而然地，我们会想到使用 `ModuleList`，像这样：
```python
class net(nn.Module):
    def __init__(self):
        super(net, self).__init__()
        self.linears = nn.ModuleList([nn.Linear(10, 10) for i in range(3)])

    def forward(self, x):
        for layer in self.linears:
            x = layer(x)
        return x
```        
这个是比较一般的方法，但如果不想这么麻烦，我们也可以用 `Sequential` 来实现，如下所示！注意 `*` 这个操作符，它可以把一个 `list` 拆开成一个个独立的元素。但是，请注意这个 `list` 里面的模块必须是按照想要的顺序来进行排列的。
```python
class net(nn.Module):
    def __init__(self):
        super(net, self).__init__()
        self.linear_list = [nn.Linear(10, 10) for i in range(3)]
        self.linears = nn.Sequential(*self.linears_list)

    def forward(self, x):
        self.x = self.linears(x)
        return x
```
### 场景二
当我们需要之前层的信息的时候，比如 `ResNets` 中的 `shortcut` 结构，或者是像 `FCN` 中用到的 `skip architecture` 之类的，当前层的结果需要和之前层中的结果进行融合，一般使用 `ModuleList` 比较方便，一个非常简单的例子如下：
```python
class net(nn.Module):
    def __init__(self):
        super(net, self).__init__()
        self.linears = nn.ModuleList([nn.Linear(10, 20), nn.Linear(20, 30), nn.Linear(30, 50)])
        self.trace = []

    def forward(self, x):
        for layer in self.linears:
            x = layer(x)
            self.trace.append(x)
        return x
net = net()
input  = torch.randn(32, 10) # input batch size: 32
output = net(input)
for each in net.trace:
    print(each.shape)
# torch.Size([32, 20])
# torch.Size([32, 30])
# torch.Size([32, 50])
```
我们使用了一个 `trace` 的列表来储存网络每层的输出结果，这样如果以后的层要用的话，就可以很方便地调用了。
# 4.模型训练和测试
## `model.eval()`和`with torch.no_grad()`的区别
在`PyTorch`中进行`validation`时，会使用`model.eval()`切换到测试模式，在该模式下，
- 主要用于通知`dropout`层和`batchnorm`层在`train`和`val`模式间切换
    在`train`模式下，`dropout`网络层会按照设定的参数`p`设置保留激活单元的概率（保留概率=`p`); `batchnorm`层会继续计算数据的`mean`和`var`等参数并更新。
在`val`模式下，`dropout`层会让所有的激活单元都通过，而`batchnorm`层会停止计算和更新`mean`和`var`，直接使用在训练阶段已经学出的`mean`和`var`值。
- 该模式不会影响各层的`gradient`计算行为，即`gradient`计算和存储与`training`模式一样，只是不进行反传(`backprobagation`)
而`with torch.zero_grad()`则主要是用于停止autograd模块的工作，以起到加速和节省显存的作用，具体行为就是停止`gradient`计算，从而节省了GPU算力和显存，但是并不会影响`dropout`和`batchnorm`层的行为。
### 使用场景
如果不在意显存大小和计算时间的话，仅仅使用`model.eval()`已足够得到正确的`validation`的结果；而`with torch.zero_grad()`则是更进一步加速和节省gpu空间（因为不用计算和存储`gradient`），从而可以更快计算，也可以跑更大的`batch`来测试。
## 自定义loss
继承`torch.nn.Module`类写自己的`loss`。
```python
class MyLoss(torch.nn.Moudle):
    def __init__(self):
        super(MyLoss, self).__init__()
        
    def forward(self, x, y):
        loss = torch.mean((x - y) ** 2)
        return loss
```
## 解决样本不平衡问题
在分类任务中，常常出现样本不平衡问题，这个问题会导致模型收敛到局部最优解，也就是广泛地将样本全部预测为某一类，因此在这里总结带权重的交叉熵损失的使用方法：
```python
    print('use wegithed cross entropy.... ')
    label_type = np.unique(label.reshape(-1))
    alpha = np.array([ np.sum(label == x) for x in label_type])
    alpha = np.max(alpha) / alpha
    alpha = np.clip(alpha,1,50)
    alpha = alpha/ np.sum(alpha)
    loss_fn = t.nn.CrossEntropyLoss(weight = t.tensor(alpha).float()) # 平衡权重
    loss_fn = loss_fn.to(device)
```
## 标签平滑（label smoothing）
写一个`label_smoothing.py`的文件，然后在训练代码里引用，用LSR代替交叉熵损失即可。`label_smoothing.py`内容如下：
```python
import torch
import torch.nn as nn


class LSR(nn.Module):

    def __init__(self, e=0.1, reduction='mean'):
        super().__init__()

        self.log_softmax = nn.LogSoftmax(dim=1)
        self.e = e
        self.reduction = reduction
    
    def _one_hot(self, labels, classes, value=1):
        """
            Convert labels to one hot vectors
        
        Args:
            labels: torch tensor in format [label1, label2, label3, ...]
            classes: int, number of classes
            value: label value in one hot vector, default to 1
        
        Returns:
            return one hot format labels in shape [batchsize, classes]
        """

        one_hot = torch.zeros(labels.size(0), classes)

        #labels and value_added  size must match
        labels = labels.view(labels.size(0), -1)
        value_added = torch.Tensor(labels.size(0), 1).fill_(value)

        value_added = value_added.to(labels.device)
        one_hot = one_hot.to(labels.device)

        one_hot.scatter_add_(1, labels, value_added)

        return one_hot

    def _smooth_label(self, target, length, smooth_factor):
        """convert targets to one-hot format, and smooth
        them.
        Args:
            target: target in form with [label1, label2, label_batchsize]
            length: length of one-hot format(number of classes)
            smooth_factor: smooth factor for label smooth
        
        Returns:
            smoothed labels in one hot format
        """
        one_hot = self._one_hot(target, length, value=1 - smooth_factor)
        one_hot += smooth_factor / (length - 1)

        return one_hot.to(target.device)

    def forward(self, x, target):

        if x.size(0) != target.size(0):
            raise ValueError('Expected input batchsize ({}) to match target batch_size({})'
                    .format(x.size(0), target.size(0)))

        if x.dim() < 2:
            raise ValueError('Expected input tensor to have least 2 dimensions(got {})'
                    .format(x.size(0)))

        if x.dim() != 2:
            raise ValueError('Only 2 dimension tensor are implemented, (got {})'
                    .format(x.size()))


        smoothed_target = self._smooth_label(target, x.size(1), self.e)
        x = self.log_softmax(x)
        loss = torch.sum(- x * smoothed_target, dim=1)

        if self.reduction == 'none':
            return loss
        
        elif self.reduction == 'sum':
            return torch.sum(loss)
        
        elif self.reduction == 'mean':
            return torch.mean(loss)
        
        else:
            raise ValueError('unrecognized option, expect reduction to be one of none, mean, sum')
```
或者直接在训练文件里做`label smoothing`
```python
for images, labels in train_loader:
    images, labels = images.cuda(), labels.cuda()
    N = labels.size(0)
    # C is the number of classes.
    smoothed_labels = torch.full(size=(N, C), fill_value=0.1 / (C - 1)).cuda()
    smoothed_labels.scatter_(dim=1, index=torch.unsqueeze(labels, dim=1), value=0.9)

    score = model(images)
    log_prob = torch.nn.functional.log_softmax(score, dim=1)
    loss = -torch.sum(log_prob * smoothed_labels) / N
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()    
```
## L1 正则化
```python
l1_regularization = torch.nn.L1Loss(reduction='sum')
loss = ...  # Standard cross-entropy loss
for param in model.parameters():
    loss += torch.sum(torch.abs(param))
loss.backward()
```
## 不对偏置项进行权重衰减（weight decay）
`PyTorch`里的`weight decay`相当于`l2`正则
```python
bias_list = (param for name, param in model.named_parameters() if name[-4:] == 'bias')
others_list = (param for name, param in model.named_parameters() if name[-4:] != 'bias')
parameters = [{'parameters': bias_list, 'weight_decay': 0},                
              {'parameters': others_list}]
optimizer = torch.optim.SGD(parameters, lr=1e-2, momentum=0.9, weight_decay=1e-4)
```
## 梯度裁剪(`gradient clipping`)
```python
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=20)
```
## 得到当前学习率
```python
# If there is one global learning rate (which is the common case).
lr = next(iter(optimizer.param_groups))['lr']

# If there are multiple learning rates for different layers.
all_lr = []
for param_group in optimizer.param_groups:
    all_lr.append(param_group['lr'])
```
另一种方法，在一个`batch`训练代码里，当前的`lr`是
```python
optimizer.param_groups[0]['lr']
```
## 学习率衰减
```python
# Reduce learning rate when validation accuarcy plateau.
scheduler = torch.optim.lr_scheduler.ReduceLROnPlateau(optimizer, mode='max', patience=5, verbose=True)
for t in range(0, 80):
    train(...)
    val(...)
    scheduler.step(val_acc)

# Cosine annealing learning rate.
scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=80)
# Reduce learning rate by 10 at given epochs.
scheduler = torch.optim.lr_scheduler.MultiStepLR(optimizer, milestones=[50, 70], gamma=0.1)
for t in range(0, 80):
    scheduler.step()    
    train(...)
    val(...)

# Learning rate warmup by 10 epochs.
scheduler = torch.optim.lr_scheduler.LambdaLR(optimizer, lr_lambda=lambda t: t / 10)
for t in range(0, 10):
    scheduler.step()
    train(...)
    val(...)
```
## 优化器链式更新
从1.4版本开始，`torch.optim.lr_scheduler` 支持链式更新(chaining)，即用户可以定义两个 `schedulers`，并交替在训练中使用。
```python
import torch
from torch.optim import SGD
from torch.optim.lr_scheduler import ExponentialLR, StepLR
model = [torch.nn.Parameter(torch.randn(2, 2, requires_grad=True))]
optimizer = SGD(model, 0.1)
scheduler1 = ExponentialLR(optimizer, gamma=0.9)
scheduler2 = StepLR(optimizer, step_size=3, gamma=0.1)
for epoch in range(4):
    print(epoch, scheduler2.get_last_lr()[0])
    optimizer.step()
    scheduler1.step()
    scheduler2.step()
```
## 模型训练可视化
`PyTorch`可以使用1tensorboard`来可视化训练过程。
安装和运行`TensorBoard`。
```python
pip install tensorboard
tensorboard --logdir=runs
```
使用`SummaryWriter`类来收集和可视化相应的数据，放了方便查看，可以使用不同的文件夹，比如`Loss/train`和`Loss/test`。
```python
from torch.utils.tensorboard import SummaryWriter
import numpy as np

writer = SummaryWriter()

for n_iter in range(100):
    writer.add_scalar('Loss/train', np.random.random(), n_iter)
    writer.add_scalar('Loss/test', np.random.random(), n_iter)
    writer.add_scalar('Accuracy/train', np.random.random(), n_iter)
    writer.add_scalar('Accuracy/test', np.random.random(), n_iter)
```
## 保存与加载断点
注意为了能够恢复训练，我们需要同时保存模型和优化器的状态，以及当前的训练轮数。
```python 
start_epoch = 0
# Load checkpoint.
if resume: # resume为参数，第一次训练时设为0，中断再训练时设为1
    model_path = os.path.join('model', 'best_checkpoint.pth.tar')
    assert os.path.isfile(model_path)
    checkpoint = torch.load(model_path)
    best_acc = checkpoint['best_acc']
    start_epoch = checkpoint['epoch']
    model.load_state_dict(checkpoint['model'])
    optimizer.load_state_dict(checkpoint['optimizer'])
    print('Load checkpoint at epoch {}.'.format(start_epoch))
    print('Best accuracy so far {}.'.format(best_acc))

# Train the model
for epoch in range(start_epoch, num_epochs): 
    ... 

    # Test the model
    ...
        
    # save checkpoint
    is_best = current_acc > best_acc
    best_acc = max(current_acc, best_acc)
    checkpoint = {
        'best_acc': best_acc,
        'epoch': epoch + 1,
        'model': model.state_dict(),
        'optimizer': optimizer.state_dict(),
    }
    model_path = os.path.join('model', 'checkpoint.pth.tar')
    best_model_path = os.path.join('model', 'best_checkpoint.pth.tar')
    torch.save(checkpoint, model_path)
    if is_best:
        shutil.copy(model_path, best_model_path)
```
# 5.其他注意事项
## 数据读取
  使用`read_csv`读取一次原始文件，将`dataframe`存储为`HDF`或者`feather`格式。一般情况下`HDF`的读取比读取`csv`文件快几十倍，但`HDF`文件在大小上会稍微大一些。
## Tricks
  1. 建议有参数的层和汇合（pooling）层使用`torch.nn`模块定义，激活函数直接使用`torch.nn.functional。torch.nn`模块和`torch.nn.functional`的区别在于，`torch.nn`模块在计算时底层调用了`torch.nn.functional`，但`torch.nn`模块包括该层参数，还可以应对训练和测试两种网络状态。使用`torch.nn.functional`时要注意网络状态，如
  ```python
  def forward(self, x):
    ...
    x = torch.nn.functional.dropout(x, p=0.5, training=self.training)
  ```
  2. 不要使用太大的线性层。因为nn.Linear(m,n)使用的是O(mn)的内存，线性层太大很容易超出现有显存。
  3. 不要在太长的序列上使用RNN。因为RNN反向传播使用的是BPTT算法，其需要的内存和输入序列的长度呈线性关系。
  4. model(x) 前用 `model.train()` 和 `model.eval()` 切换网络状态。
  5. 不需要计算梯度的代码块用 with torch.no_grad() 包含起来。
  6. `model.eval()` 和 `torch.no_grad()` 的区别在于，`model.eval()` 是将网络切换为测试状态，例如 BN 和dropout在训练和测试阶段使用不同的计算方法。`torch.no_grad() `是关闭 PyTorch 张量的自动求导机制，以减少存储使用和加速计算，得到的结果无法进行 `loss.backward()`。
  7. `model.zero_grad()`会把整个模型的参数的梯度都归零, 而`optimizer.zero_grad()`只会把传入其中的参数的梯度归零。
  8. torch.nn.CrossEntropyLoss 的输入不需要经过 Softmax。torch.nn.CrossEntropyLoss 等价于 torch.nn.functional.log_softmax + torch.nn.NLLLoss。
  9. `loss.backward()`前用 `optimizer.zero_grad()` 清除累积梯度。
  10. `torch.utils.data.DataLoader` 中尽量设置 `pin_memory=True`，对特别小的数据集如 MNIST 设置 `pin_memory=False` 反而更快一些。`num_workers` 的设置需要在实验中找到最快的取值。
  11. 用 `del `及时删除不用的中间变量，节约 GPU 存储。
  12. 使用 `inplace` 操作可节约 GPU 存储，如
```python
x = torch.nn.functional.relu(x, inplace=True)
```
  13. 减少 CPU 和 GPU 之间的数据传输。例如如果你想知道一个 epoch 中每个 mini-batch 的 loss 和准确率，先将它们累积在 GPU 中等一个 epoch 结束之后一起传输回 CPU 会比每个 `mini-batch` 都进行一次 GPU 到 CPU 的传输更快。
  14. 使用半精度浮点数 `half()`会有一定的速度提升，具体效率依赖于 GPU 型号。需要小心数值精度过低带来的稳定性问题。
  15. 时常使用 `assert tensor.size() == (N, D, H, W)` 作为调试手段，确保张量维度和你设想中一致。
  16. 除了标记 y 外，尽量少使用一维张量，使用 n*1 的二维张量代替，可以避免一些意想不到的一维张量计算结果。
  17. 统计代码各部分耗时
```python
with torch.autograd.profiler.profile(enabled=True, use_cuda=False) as profile:
    ...
print(profile)

# 或者在命令行运行
python -m torch.utils.bottleneck main.py
```
  18. 使用`TorchSnooper`来调试`PyTorch`代码，程序在执行的时候，就会自动 `print` 出来每一行的执行结果的 `tensor` 的形状、数据类型、设备、是否需要梯度的信息。
```python
# pip install torchsnooper
import torchsnooper

# 对于函数，使用修饰器
@torchsnooper.snoop()

# 如果不是函数，使用 with 语句来激活 TorchSnooper，把训练的那个循环装进 with 语句中去。
with torchsnooper.snoop():
    原本的代码
```
  19. 训练过程记录及可视化:
  - `wandb`
  - `comet_ml`
  - `mlflow`
  20. 一些扩展包：
  - `torch-optimizer`:实现了最新的一些优化器.
  - `fastai`:有一些评价指标
  - `numba`:`import numba as nb`,纯`python`或`numpy`函数加装饰器,加速计算，加@nb.njit或@nb.jit(nopython=True)
  - `swifter`:`df.apply()`→`df.swifter.apply()`，加速`pandas`
  - `captum`:可解释性
  - `cupy`:加速`pandas`,1000万以上数据更快
  - `modin`:`import modin.pandas as mdpd`,用`mdpd`代替`pd`即可，加速`pandas`,加载数据和查询数据更快,统计方法`pandas`更快
  
