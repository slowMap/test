# [QT QObject分析](https://www.cnblogs.com/WushiShengFei/p/9820835.html)

看了上面大佬写的东西，自己也总结一下吧，元对象系统中实现了很多功能，有信号槽机制，将信号，槽，qt 的一些宏转化为 moc\_\*.h文件，而且其中的信号和槽连接，是通过字符串连接的，然后通过存储的数据，在函数中判断所类型，和取得的序列号，然后通过switch case到达相应的槽。

值得注意的点：

1. 而槽所对应的参数比信号要少，会自动获得较少的。

2. 对于sender()而言，尽量不要使用，因为在采用queue连接时，会获得一个空的对象。

3. 对于一般的连接采用postevent()的方式，或者回调的方式。当跨线程时，尽量采用`QueuedConnection`而，是在要阻塞运行，采用`BlockingQueuedConnection`的方式。以前是从来没有发现过`BlockingQueuedConnection`和`DirectConnection`两者的区别，现在发现只是前者只能用在线程中。

4. 对于emit而言 是一个空的宏，而qt将其转化为moc\_\*.h文件，而qt而言，首先将参数，转化为void，然后讲通过一个自己的元对象存储系统中的序号的方式，交给对方的元对象系统解决或者有存储到那个槽函数则调用。期间还有通过连接方式的不同，调用方式也不同。


     void *_a[] = { Q_NULLPTR, const_cast<void*>(reinterpret_cast<const void*>(&_t1)) };
     QMetaObject::activate(this, &staticMetaObject, 0, _a);

5. 对于signgal和slot的定义，可以明显发现signals前面其实是有一个public的，所以不能加限定的词语，而slots是可以的


    //qobjectdefs.h
    define  slots  Q_SLOTS
    define  signals  Q_SIGNALS
     define  Q_SLOTS  QT_ANNOTATE_ACCESS_SPECIFIER(qt_slot)
     define  Q_SIGNALS  public  QT_ANNOTATE_ACCESS_SPECIFIER(qt_signal)

6. 我们在来看看`Q_Object`


    #define  Q_OBJECT  \
    public:  \
      Q_OBJECT_CHECK  \   //qcast_object宏转化，
      QT_WARNING_PUSH  \   // 一些警告
      Q_OBJECT_NO_OVERRIDE_WARNING  \
      static  const  QMetaObject  staticMetaObject;  \  //静态的对象
      virtual  const  QMetaObject  *metaObject()  const;  \  //调用上面的对象
      virtual  void  *qt_metacast(const  char  *);  \       //将字符串转化为相应的方法
      virtual  int  qt_metacall(QMetaObject::Call,  int,  void  **);  \ //通过方式和序号和参数，调用相应的槽函数
      QT_TR_FUNCTIONS  \ //国际化处理
    private:  \
      Q_OBJECT_NO_ATTRIBUTES_WARNING  \
      Q_DECL_HIDDEN_STATIC_METACALL  static  void  qt_static_metacall(QObject  *,  QMetaObject::Call,  int,  void  **);  \
      QT_WARNING_POP  \
      struct  QPrivateSignal  {};  \
      QT_ANNOTATE_CLASS(qt_qobject,  "")

7. `Q_PROPERTY(QString objectName READ objectName WRITE setObjectName NOTIFY objectNameChanged)`
   定义一个属性，将变量暴露在外部出去，可以适当的减少属性
   
8. `Q_DECLARE_PRIVATE(QObject)`创建了一个私有的对象，然后通过d指针去访问他们，

    #define  Q_DECLARE_PRIVATE(Class)  \
      inline  Class##Private*  d_func()  {  return  reinterpret_cast<Class##Private  *>(qGetPtrHelper(d_ptr));  }  \
      inline  const  Class##Private*  d_func()  const  {  return  reinterpret_cast<const  Class##Private  *>(qGetPtrHelper(d_ptr));  }  \
      friend  class  Class##Private;

    #define  Q_DECLARE_PRIVATE_D(Dptr,  Class)  \
      inline  Class##Private*  d_func()  {  return  reinterpret_cast<Class##Private  *>(Dptr);  }  \
      inline  const  Class##Private*  d_func()  const  {  return  reinterpret_cast<const  Class##Private  *>(Dptr);  }  \
      friend  class  Class##Private;

    #define  Q_DECLARE_PUBLIC(Class)  \
      inline  Class*  q_func()  {  return  static_cast<Class  *>(q_ptr);  }  \
      inline  const  Class*  q_func()  const  {  return  static_cast<const  Class  *>(q_ptr);  }  \
      friend  class  Class;

    #define  Q_D(Class)  Class##Private  *  const  d  =  d_func()
    #define  Q_Q(Class)  Class  *  const  q  =  q_func()

而对于qobject的构造函数，首相将q指针，指向this，将d指针指向threadData，然后在初始化threadData，设置父节点
`Q_D(QObject);`
```d_ptr->q_ptr = ``this``;```
```d->threadData = (parent && !parent->``thread``()) ? parent->d_func()->threadData : QThreadData::current();```
`d->threadData->ref();`
`if` `(parent) {`
`QT_TRY {`
`if` `(!check_parent_thread(parent, parent ? parent->d_func()->threadData : 0, d->threadData))`
`parent = 0;`
`setParent(parent);`
`} QT_CATCH(...) {`
`d->threadData->deref();`
`QT_RETHROW;`
`}`
`}`
`#if QT_VERSION < 0x60000`
```qt_addObject(``this``);```
`#endif`
`if` `(Q_UNLIKELY(qtHookData[QHooks::AddQObject]))`
```reinterpret_cast``<QHooks::AddQObjectCallback>(qtHookData[QHooks::AddQObject])(``this``);```

9. moveToThread函数
   处于同一线程，parent为空，并且不为窗口部件
```
  d->moveToThread_helper(); // this and child send ThreadChange
      if  (!targetData)
      targetData  =  new  QThreadData(0);
      QOrderedMutexLocker  locker(&currentData->postEventList.mutex
      &targetData->postEventList.mutex);
      //  keep  currentData  alive  (since  we've  got  it  locked)
      currentData->ref();
      
      //  move  the  object 
      //  move post event, and set  new  thread  data
      d_func()->setThreadData_helper(currentData,  targetData);
      
      locker.unlock();
      //  now  currentData  can  commit  suicide  if  it  wants  to
      currentData->deref();
```
    
