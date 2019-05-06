# layer.cpp

[back](../../index.md)
[back to caffe](caffe_index.md)

1. 这一部分的的内容主要了解一下layer是如何构建的。代码当中的解释是：构成网络的基本组件。
2. Layer必须实现的函数包括**Forward**和**Backward**函数。
3. 每一个Layer都对应两个Blob(每一个Blob当中有data\_ 和diff\_两个空间来分别保存数据本身和数据的梯度，具体见blob)。
4. 在Forward当中，从bottom blob当中读取输入数据进行当前层定义的计算（卷积、pooling、ReLu等），然后输出到top blob当中。因为caffe网络的定义是从低向上的，所以每一层都是从bottom读取想top输出。
5. Backward层当中，从top blob当中提取数据进行梯度计算，输出到bottom blob当中。

首先来看一下每个layer当中保存的成员变量和成员函数
第一部分是对于锁的定义。

```c
bool is_shared_; //表示是否是一个被多个网络共享的层
shared_ptr<boost::mutex> forward_mutex_; //如果一个层是被共享的，则需要一个互斥锁
void InitMutex();
void Lock();
void Unlock();
```
下面是对一个layer需要的基本的成员变量，我们要知道top和bottom的blob当中保存的是每一层的输入和输出的数据，而每一层当中的blob是等待学习的参数。对于一个卷积层而言下面代码中vectoer当中的每一个blob都保存的是一个卷积。对于top和bottom的blob都是保存在vector当中的，一个vector表示的是一个batch，一个元素是一个数据。所以top.size应该等于bottom.size。

```c
LayerParameter layer_param_; //protobuf结构，保存层的参数
Phase phase_; //记录当前所处的阶段，训练阶段还是测试阶段
vector<shared_ptr<Blob<Dtype> > > blobs_;//记录一系列等待学习的参数
vector<bool> param_propagate_down_; //记录每一个参数是否需要计算梯度
vector<Dtype> loss_; //每个目标函数在top 中是否有非零的权重向量
```
在每一次初始化的过程中，需要调用下面的函数对layer进行初始化：

```c
void SetUp(const vector<Blob<Dtype>*>& bottom, const vector<Blob<Dtype>*>& top) {
    InitMutex(); //初始化互斥锁
    CheckBlobCounts(bottom, top); //检查输入输出数据的数量是否匹配
    LayerSetUp(bottom, top); //对层做一些设置,目前是一个空函数
    Reshape(bottom, top);//根据输入输出对每一个训练参数进行大小调整,目前是纯虚函数
    SetLossWeights(top);//设置误差权重，对呀loss的设置还有一系列的内容，随后再看看
}
```

下面就是最重要的backward和forward,这两个函数中目前已经实现了一个框架负责调用我们自己的实现。我们可以清楚的看到这里前向和后向都分别有gpu和cpu版本，cpu版本目前全部都是纯虚函数，而GPU版本暂时直接调用cpu所以在继承时，cpu版本必须实现而gpu不一定需要实现。

对于被多个网络共享的层使用了锁，为什么反向传播没有使用锁呢？

另外在前向传播当中我注释了loss相关的部分，loss应该是涉及到了误差计算的部分。

```c
template <typename Dtype>
inline Dtype Layer<Dtype>::Forward(const vector<Blob<Dtype>*>& bottom, const vector<Blob<Dtype>*>& top) {
    Lock(); //加互斥锁
    Dtype loss = 0;
    Reshape(bottom, top);
    switch (Caffe::mode()) {
        case Caffe::CPU:
            Forward_cpu(bottom, top);
            //……loss
            break;
        case Caffe::GPU:
          	Forward_gpu(bottom, top);
    #ifndef CPU_ONLY
        //……loss
    #endif
    }
    Unlock();	//解锁
    return loss;
}

template <typename Dtype>
inline void Layer<Dtype>::Backward(const vector<Blob<Dtype>*>& top, const vector<bool>& propagate_down,
      const vector<Blob<Dtype>*>& bottom) {
    switch (Caffe::mode()) {
        case Caffe::CPU:
            Backward_cpu(top, propagate_down, bottom);
            break;
        case Caffe::GPU:
          	Backward_gpu(top, propagate_down, bottom);
    }
}
```








