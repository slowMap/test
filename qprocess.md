# QProcess启动线程的内存问题

在使用QProcess的过程中，采用了

    start(const  QString  &program,  const  QStringList  &arguments,  OpenMode  mode  =  ReadWrite)

启动与项目相关的_exe文件_，然后在采用`QIODevice 中 write(const  char  *data)`进行写入，本来发现挺好的，但是突然出现了一个bug，在进行打开QProcess后，居然视频的流没有办法正常的关闭，于是猜测有QProcess启动另一个子线程采用了同一个空间。而采用

    static  bool  startDetached(const  QString  &program,  const  QStringList  &arguments);

构建了两个没有子父关系的进程时，QProcess中的各种函数都不起作用，例如判断是否当前状态的函数`state()`或者写入进程函数`write(char*data)`。

查看QT的源代码后，在qprocess_win.cpp 找到了源代码`startProcess ()`函数中

    struct  CreateProcessArguments
      {
      const  wchar_t  *applicationName;
      wchar_t  *arguments;
      Q_SECURITY_ATTRIBUTES  *processAttributes;
      Q_SECURITY_ATTRIBUTES  *threadAttributes;
      bool  inheritHandles;
      unsigned  long  flags;
      void  *environment;
      const  wchar_t  *currentDirectory;
      Q_STARTUPINFO  *startupInfo;
      Q_PID  processInformation;
      };

        QProcess::CreateProcessArguments cpargs = {
            0, (wchar_t*)args.utf16(),
            0, 0, TRUE, dwCreationFlags,
            environment.isEmpty() ? 0 : envlist.data(),
            nativeWorkingDirectory.isEmpty() ? Q_NULLPTR : (wchar_t*)nativeWorkingDirectory.utf16(),
            &startupInfo, pid
        };
        if (modifyCreateProcessArgs)
            modifyCreateProcessArgs(&cpargs);
        success = CreateProcess(cpargs.applicationName, cpargs.arguments, cpargs.processAttributes,
                                cpargs.threadAttributes, cpargs.inheritHandles, cpargs.flags,
                                cpargs.environment, cpargs.currentDirectory, cpargs.startupInfo,
                                cpargs.processInformation);

而windows [CreateProcess的参数命令](https://baike.baidu.com/item/CreateProcess/11050419?fr=aladdin)

值：**CREATE_SEPARATE_WOW_VDM**

如果被设置，新进程将会在一个私有的虚拟DOS机（VDM）中运行。另外，默认情况下所有的16位Windows应用程序都会在同一个共享的VDM中以[线程](https://baike.baidu.com/item/%E7%BA%BF%E7%A8%8B)的方式运行。单独运行一个16位程序的优点是一个应用程序的崩溃只会结束这一个VDM的运行；其他那些在不同VDM中运行的程序会继续正常的运行。同样的，在不同VDM中运行的16位Windows应用程序拥有不同的输入队列，这意味着如果一个程序暂时失去响应，在独立的VDM中的应用程序能够继续获得输入。所以可以看到为0，所以以线程的方式运行，公用同一段空间。

### 解决方式

既然，不行，然后采用`startDetached`的方式启动，然后使用socket的方式两者进行通信。而对于qt而言，在文档中**On Windows this is a named pipe and on Unix this is a local domain socket.**表明采用了管道的方式，所以不会造成资源的浪费。而QLocalSocket采用的是全双工的模式，可以相互之间通信\~~
