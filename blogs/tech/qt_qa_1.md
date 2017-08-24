在软件开发的过程中需要对QT界面进行调整，主要使用了QSS的样式表，同时为了实现一些特别的功能需要对某些类进行重写。
在对样式进行改进的过程中遇到了两个问题：
1. signals明明已经声明了，但是提示：

```shell
error LNK2019: 无法解析的外部符号 "public: void __cdecl QMyWidget::mouseMoveSignal()
```

QMyWidget是我自己重写的QWidget，主要为了实现一些特别的功能，mouseMoveSignal是我写的信号。
在执行emit mouseMoveSignal()时，发生了错误，这个错误主要是由于，在手动书写代码的时候漏掉了Q_OBJECT的声明。


代码片段如下：

```c
class QMyWidget
    public QWidget
{
    Q_OBJECT //丢失了这一句会造成上面的错误。
public:
    QMyWidget(void);
    QMyWidget(QWidget *);
    ~QMyWidget(void);
```

2. 在我完成重写了QMyWidget之后，发现我对界面的QSS设置全部失效了。
此时需要重写新类的paintEvent方法。代码如下：

```c
void QMyWidget::paintEvent(QPaintEvent *e){
    QStyleOption opt;
    opt.init(this);
    QPainter p(this);
    style()->drawPrimitive(QStyle::PE_Widget, &opt, &p, this);
    QWidget::paintEvent(e);
}
```

直接使用这一段代码进行复制即可使用，将QMyWidget改成你对应的类名，并且包含对应的类，就可以解决这一问题。

[back to list](../../index.md)