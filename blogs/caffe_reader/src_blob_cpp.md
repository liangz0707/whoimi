# blob.cpp

[back](../../index.md)

[back to caffe](caffe_index.md)

注释当中的解释是：同步内存的包装器，基本的计算单元。

这部分内容主要来自两个文件： **blob.cpp** 和 **blob.hpp**

先来看看Bolb的初始化，除了默认的初始化构造函数之外，就是需要通过尺寸定义一个内存空间。

1. num: 每个batch的尺寸
2. channels: 图像的通道数
3. height、weight: 图像的大小

```c
Blob(): data_(), diff_(), count_(0), capacity_(0) {}
Blob(const int num, const int channels, const int height, const int width);
Blob(const vector<int>& shape);
```

除了通过构造函数能够决定内存空间的大小外，也可以通过成员函数

```c
void Reshape(const int num, const int channels, const int height, const int width);
void Reshape(const vector<int>& shape);
void Reshape(const BlobShape& shape);
```

需要保存的内容主要是三种：前向计算的输入，后向计算的梯度和尺寸大小。这些内容保存在SynceMemory当中，通过智能指针shared_ptr管理。同时count_表示总大小

```c
shared_ptr<SyncedMemory> data_;
shared_ptr<SyncedMemory> diff_;
shared_ptr<SyncedMemory> shape_data_;
int count_;//data_或diff_需要占用的大小；
int capacity_;//表示当前所申请空间的大小，如果count_>capacity_则需要重新申请空间
```

在reshape当中如果尺寸变大，则需要重新申请空间

```c
//shape_ 是vector<int>成员变量,shape
if (!shape_data_ || shape_data_->size() < shape.size() * sizeof(int)) {
    shape_data_.reset(new SyncedMemory(shape.size() * sizeof(int)));
}
//如果count_>capacity_则需要重新申请空间
if (count_ > capacity_) {
    capacity_ = count_;
    data_.reset(new SyncedMemory(capacity_ * sizeof(Dtype)));
	diff_.reset(new SyncedMemory(capacity_ * sizeof(Dtype)));
}
```

上面的内容主要是申请空间，然后通过以下方法访问数据。

```c
data_->gpu_data();
data_->cpu_data();
data_->mutable_gpu_data();
data_->mutable_cpu_data();
```

其中还有两个重要的方法，使用了google的格式.proto，这个随后再看。

```c
template <typename Dtype> void Blob<Dtype>::FromProto(const BlobProto& proto, bool reshape)
template <> void Blob<double>::ToProto(BlobProto* proto, bool write_diff)
```
进行不同格式的转换。



