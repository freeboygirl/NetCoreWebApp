Net Core Logger 实现log写入本地文件系统

.net core 自带一个基础的logger框架Microsoft.Extensions.Logging。

微软默认实现了Microsoft.Extensions.Logging.Console.dll。控制台的日志输出和Microsoft.Extensions.Logging.Debug.dll调试输出。

下面我们写一个我们自己的本地文件输出模块demo，简单理解一下自带的这个logger系统。

logger框架主要几个类：LoggerFactory，Logger，LoggerProvider。

看名字就很好理解，都不需要解释。

实现我们自己的file logger只需要实现logger，loggerProvider即可。

第一步：入口。

loggerFactory.AddFile(this.Configuration.GetSection("FileLogging"));

为LoggerFactory扩张一个方法，提供增加日志写文件方式的入口。相关的配置来自appsettings.json

 View Code
第二步：实现我们的logger提供程序，实现ILoggerProvider接口

public class FileLoggerProvider : ILoggerProvider, Idisposable

关键方法CreateLogger，创建真正写日志的logger。对当前的logger可以做适当的缓存，配置logger

 View Code
第三步：实现我们的logger，实现ILogger接口。真正将log写入file

public class FileLogger : Ilogger


复制代码
  1     public class FileLogger : ILogger
  2     {
  3         static protected string delimiter = new string(new char[] { (char)1 });
  4         public FileLogger(string categoryName)
  5         {
  6             this.Name = categoryName;
  7         }
  8         class Disposable : IDisposable
  9         {
 10             public void Dispose()
 11             {
 12             }
 13         }
 14         Disposable _DisposableInstance = new Disposable();
 15         public IDisposable BeginScope<TState>(TState state)
 16         {
 17             return _DisposableInstance;
 18         }
 19         public bool IsEnabled(LogLevel logLevel)
 20         {
 21             return this.MinLevel <= logLevel;
 22         }
 23         public void Reload()
 24         {
 25             _Expires = true;
 26         }
 27 
 28         public string Name { get; private set; }
 29 
 30         public LogLevel MinLevel { get; set; }
 31         public string FileDiretoryPath { get; set; }
 32         public string FileNameTemplate { get; set; }
 33         public void Log<TState>(LogLevel logLevel, EventId eventId, TState state, Exception exception, Func<TState, Exception, string> formatter)
 34         {
 35             if (!this.IsEnabled(logLevel))
 36                 return;
 37             var msg = formatter(state, exception);
 38             this.Write(logLevel, eventId, msg, exception);
 39         }
 40         void Write(LogLevel logLevel, EventId eventId, string message, Exception ex)
 41         {
 42             EnsureInitFile();
 43 
 44             //TODO 提高效率 队列写！！！
 45             var log = String.Concat(DateTime.Now.ToString("HH:mm:ss"), '[', logLevel.ToString(), ']', '[',
 46                   Thread.CurrentThread.ManagedThreadId.ToString(), ',', eventId.Id.ToString(), ',', eventId.Name, ']',
 47                   delimiter, message, delimiter, ex?.ToString());
 48             lock (this)
 49             {
 50                 this._sw.WriteLine(log);
 51             }
 52         }
 53 
 54         bool _Expires = true;
 55         string _FileName;
 56         protected StreamWriter _sw;
 57         void EnsureInitFile()
 58         {
 59             if (CheckNeedCreateNewFile())
 60             {
 61                 lock (this)
 62                 {
 63                     if (CheckNeedCreateNewFile())
 64                     {
 65                         InitFile();
 66                         _Expires = false;
 67                     }
 68                 }
 69             }
 70         }
 71         bool CheckNeedCreateNewFile()
 72         {
 73             if (_Expires)
 74             {
 75                 return true;
 76             }
 77             //TODO 使用 RollingType判断是否需要创建文件。提高效率！！！
 78             if (_FileName != DateTime.Now.ToString(this.FileNameTemplate))
 79             {
 80                 return true;
 81             }
 82             return false;
 83         }
 84         void InitFile()
 85         {
 86             if (!Directory.Exists(this.FileDiretoryPath))
 87             {
 88                 Directory.CreateDirectory(this.FileDiretoryPath);
 89             }
 90             var path = "";
 91             int i = 0;
 92             do
 93             {
 94                 _FileName = DateTime.Now.ToString(this.FileNameTemplate);
 95                 path = Path.Combine(this.FileDiretoryPath, _FileName + "_" + i + ".log");
 96                 i++;
 97             } while (System.IO.File.Exists(path));
 98             var oldsw = _sw;
 99             _sw = new StreamWriter(new FileStream(path, FileMode.CreateNew, FileAccess.ReadWrite, FileShare.Read), Encoding.UTF8);
100             _sw.AutoFlush = true;
101             if (oldsw != null)
102             {
103                 try
104                 {
105                     _sw.Flush();
106                     _sw.Dispose();
107                 }
108                 catch
109                 {
110                 }
111             }
112         }
113     }
复制代码
代码：https://github.com/czd890/NetCoreWebApp

