<!--
.. title: Qt的Impl
.. slug: qtde-impl
.. date: 2021-02-23 20:56:26 UTC+08:00
.. tags: 
.. category: 
.. link: 
.. description: 
.. type: text
-->

[TOC]

在Qt中，大多数的类在设计的时候都把具体的数据处理放在一个Private类中，这样做有几个好处：
- 能保证二进制兼容性（关于二进制兼容型,可以参考[Policies/Binary Compatibility Issues With C++](https://community.kde.org/Policies/Binary_Compatibility_Issues_With_C%2B%2B)），就是发布库的时候可以尽量保持跟以前版本的兼容，不至于因为添加了某些成员变量导致原来调用库的程序奔溃；
- 在编译的时候不用因为修改了一点东西，而影响太多的文件
- d指针的实现方法并不限于QObject的子类

## 注意事项
D-Pointer这样的写法好处是显而易见的，但是也带来了一些问题，因此，我们在设计的时候需要注意

- 如果在D Class中需要使用到QTimer, QTcpSocket之类的类，同时Q Class又可能被应用在多线程环境，那么QTimer等类应该在Q Class中初始化为Q Class的子类，这样moveToThread后QTimer等对象就能跟随Q Class转移了。例如：
```c
class ParserPrivate;
class Parser : public QObject
{
    Q_OBJECT
    // ...
private:
    //! Private class data pointer
    ParserPrivate *d;
};

class ParserPrivate
{
public:
    //! timer
    QPointer<QTimer> timer;
    //! Constructor
    ParserPrivate() : flatMode(false) {}
    //...
};

Parser::Parser(QObject *parent)
    : QObject(parent),
    d(new ParserPrivate())
{
    d->timer = new QTimer(this);
    d->timer->setObjectName(QLatin1String("ClassViewParser::timer"));
    d->timer->setSingleShot(true);

    // timer for emitting changes
    connect(d->timer.data(), &QTimer::timeout, this, &Parser::requestCurrentState, Qt::QueuedConnection);
}

Parser::~Parser()
{
    delete d;
}
```

## 实例说明

下面的例子因为`StaServerPrivate`不需要使用QObject的功能，因此没有继承QObject；如果需要用到信号槽或者其他的QObject功能，添加了Q_OBJECT宏，那么就需要将`StaServerPrivate`类定义到一个新的头文件`staserver_p.h`中，以便moc处理。

### staserver.h
```c
#ifndef STASERVER_H
#define STASERVER_H

#include <QObject>
#include <QTcpSocket>
#include <QTcpServer>

class StaServerPrivate;

class StaServer : public QTcpServer
{
    Q_OBJECT
    Q_DECLARE_PRIVATE(StaServer)
public:
    explicit StaServer(QObject *parent = Q_NULLPTR);
    ~StaServer();
    StaServer(const StaServer &other);
    StaServer &operator=(const StaServer &other);

    int port() const;

protected:
    StaServer(StaServerPrivate &dd, QObject *parent = Q_NULLPTR);

private:
    StaServerPrivate * const d_ptr;
    // If not defined copy and operator= funtion, enable Q_DISABLE_COPY
    //Q_DISABLE_COPY(StaServer)
};

#endif // STASERVER_H

```

### staserver.cpp

```c
class StaServerPrivate
{
    Q_DECLARE_PUBLIC(StaServer)
public:
    StaServerPrivate(StaServer *parent);
    virtual ~StaServerPrivate();
    StaServerPrivate &operator=(const StaServerPrivate &other);

    StaServer * const q_ptr;
    int m_port;
};

StaServerPrivate::StaServerPrivate(StaServer *parent)
    : q_ptr(parent)
{
    Q_Q(StaServer);

}

StaServerPrivate::~StaServerPrivate()
{

}

StaServerPrivate &StaServerPrivate::operator=(const StaServerPrivate &other)
{
    m_port = other.m_port;

    return *this;
}

StaServer::StaServer(QObject *parent)
    : QTcpServer(parent)
    , d_ptr(new StaServerPrivate(this))
{
    Q_D(StaServer);
}

StaServer::~StaServer()
{
    delete d_ptr;
}

StaServer &StaServer::operator=(const StaServer &other)
{
    *d_ptr = *(other.d_ptr);
    return *this;
}

StaServer::StaServer(const StaServer &other)
    : d_ptr(new StaServerPrivate(this))
{
    *d_ptr = *(other.d_ptr);
}

// this function is used for child class
StaServer::StaServer(StaServerPrivate &dd, QObject *parent)
    : d_ptr(&dd)
{

}

int StaServer::port()
{
    Q_D(const StaServer);

    return d->m_port;
}
```

如果某子类继承以上的类，则子类的写法如下：
```c
class StaServerChildPrivate;
class StaServerChild : public StaServer
{
    Q_DECLARE_PRIVATE(StaServerChild)
public:
    StaServerChild(QObject *parent);
}

class StaServerChildPrivate : public StaServerPrivate
{
    Q_DECLARE_PUBLIC(StaServerChild)
public:

};

StaServerChild::StaServerChild(QObject *parent)
    : StaServer(* new StaServerChildPrivate)
{

}

```

## QObject

通过上面的分析，我们大致知道了写法（原理的部分参考下面的Qt Wiki可以更深入了解）。回头来看看Qt的框架。Qt最基础的类就是QObject，所以从QObject开始，就使用了d pointer的写法，QObject是所有类最底层的基类。所有继承QObject的类都可以使用QObject提供的q_ptr（通过函数`QObject(QObjectPrivate &dd, QObject *parent = nullptr);`）。


## 参考
- [Qt Wiki: D-Pointer](https://wiki.qt.io/D-Pointer)
- [KDE Community Wiki: Policies/Library Code Policy](https://community.kde.org/Policies/Library_Code_Policy)
- [How to use the Qt's PIMPL idiom?](https://stackoverflow.com/questions/25250171/how-to-use-the-qts-pimpl-idiom)



