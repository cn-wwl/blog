### 序言

​		最近在做抄表项目时，需要增加一个点抄功能，考虑到用户体验性、及时性及服务器压力等问题，使用`ajax轮询`的方式请求服务端资源的方式显然已经out了 ，以此查看当前下发的抄表指令的结果，因此查了一下资料，决定使用[websocket](https://www.runoob.com/html/html5-websocket.html)，以此实现点抄功能。

### 抄表介绍

​		以前居民用水使用的水表是以机械表来计量的，因此自来水公司每月收费之前只能派员工去走抄，然后进行开票收费，此种收费方式比较费时费力 且采集到的用水量信息可能出错，随着嵌入式技术的发展，无线远传水表诞生了，此类水表可以将用户的用水量信息通过无线传输技术传递到自来水公司的抄表平台，而我们现在就在做这个平台。
<img title="机械水表" src="https://images.cnblogs.com/cnblogs_com/cn-wwl/1764499/o_200513135651机械水表.jpg" alt="https://images.cnblogs.com/cnblogs_com/cn-wwl/1764499/o_200513135651%E6%9C%BA%E6%A2%B0%E6%B0%B4%E8%A1%A8.jpg" style="zoom:50%;" /> <img title="抄表流程图" src="https://images.cnblogs.com/cnblogs_com/cn-wwl/1764499/o_200513135700抄表流程图.png" alt="https://images.cnblogs.com/cnblogs_com/cn-wwl/1764499/o_200513135700%E6%8A%84%E8%A1%A8%E6%B5%81%E7%A8%8B%E5%9B%BE.png" style="zoom:62%;" />

​							



题外话了哈，回归正题，聊一聊三端是如何通信的。

------

### 通信流程

​		三端通信是指`webclient`、`webserver`、`clientserver`之间的通信；一般来讲，其实仅需两端就足以面对大部分场景需要，即`webclient→webserver` 、 `webclient→clientserver` 、 `webserver →clientserver` 。使用三端通信`webclient→webserver→clientserver`目的是在于避免 webclient→clientserver 通信，理论上webclient的所有请求应该由webserver进行管理，而非依赖于请求第三方应用，webserver无法处理的事情可以通过请求其第三方应用获取到信息再给到webclient。这种方式相较于webclient→clientserver 来讲优点是加强了webserver的权利，降低了webclient与第三方应用clientserver的沟通；缺点是多占用了一个端口资源。

<img  title="三端通信流程图" src="https://images.cnblogs.com/cnblogs_com/cn-wwl/1764499/o_200513144227三端通信流程图.png" alt="https://images.cnblogs.com/cnblogs_com/cn-wwl/1764499/o_200513144227%E4%B8%89%E7%AB%AF%E9%80%9A%E4%BF%A1%E6%B5%81%E7%A8%8B%E5%9B%BE.png" style="zoom:80%;" />





从上图可以看出，三端通信实际上是可以分为两个部分，webclient→clientserver之间通信和webserver →clientserver之间通信。

#### 1. webclient→webserver

​	webclient→clientserver的连接实际上有很多种，比如轮询、长轮询、websocket。有兴趣的可以去看一下这三种连接方式的优缺点，此处推荐一篇[博客](https://blog.csdn.net/pacosonswjtu/article/details/52035252)讲的还是蛮详细的，当然了websocket也是需要浏览器支持才能用的，如果说在实际项目中可能客户用的浏览器版本不一，可能无法使用websocket。

本文使用了[Fleck](https://github.com/statianzo/Fleck)作为webserver监听webseocket连接的工具。可通过NuGet安装Fleck，具体使用方式请看前面的链接。

​	webclient相关代码

```json
//测试数据：
{"ModuleName":"ClickToCopy","Content":{"ConcentratorNo":"101","WaterMtrAddress":"101001"}}
```



```html
<!DOCTYPE HTML>
<html>
<head>
    <meta charset="utf-8">
    <title>websockt</title>
    <style>
        .container {  
        background-color:#00ffff;
        border:thick solid #808080;
        margin:20px;
        padding:20px;
        }

    </style>

    <script src="Scripts/jquery-3.4.1.min.js"></script>
    <script src="Scripts/WebsocketManager.js"></script>
    <script type="text/javascript">
        $(function () {
            $("#send").click(function () {
                SendMessage($("#msg").val()); 
                $("#messagehistory").append('<li><strong>发送：</strong>&nbsp;&nbsp;' + $("#msg").val()+'</li>');
            })
        });
    </script>

</head>
<body> 
    <div id="sse">
        <input type="text" id="msg" name="msg" value="" />
        <input type="button" id="send" name="send" value="Send" />
    </div> 

    <div class="container">
        <ul id="messagehistory"></ul>
    </div>
</body>
</html>
```

```javascript
/// <reference path="jquery-3.4.1.min.js" />
var ws = null;
$(function () {
    if ("WebSocket" in window) {
        // 打开一个 web socket
        ws = new WebSocket("ws://localhost:7759");
        ws.onopen = function () {
        };

        ws.onmessage = function (evt) {
            var received_msg = evt.data;
            console.log(received_msg)
            //alert("数据已接收...");
            $("#messagehistory").append('<li><strong>接收：</strong>&nbsp;&nbsp;' + received_msg + '</li>');
        };

    }

    else {
        // 浏览器不支持 WebSocket
        alert("您的浏览器不支持 WebSocket!");
    }

})

function SendMessage(data) {
    ws.send(data);
}

function CloseWebsocket() {
    ws.onclose = function () {
        // 关闭 websocket
        alert("连接已关闭...");
    };
}

```

webserver相关代码

```c#
public class Fleckwebsockt
{
    /// <summary>
    /// 消息服务器
    /// </summary>
    public static WebSocketServer ChartSocket { get; set; }

    /// <summary>
    /// websocket连接管理器
    /// </summary>
    public static ConcurrentDictionary<Guid, websocketModel> Sockets { get; set; }

    public string msg = string.Empty;
    public static void ServerStart()
    {

        var _port = 7759;
        if (ChartSocket != null)
        {
            ChartSocket.Dispose();
        }

        ChartSocket = new WebSocketServer("ws://0.0.0.0:" + _port);

        ChartSocket.Start(socket =>
        {

            socket.OnOpen = () =>
            {
                socket.Send("连接成功");
                if (Sockets == null)
                {
                    Sockets = new ConcurrentDictionary<Guid, websocketModel>();
                }
                Sockets.TryAdd(socket.ConnectionInfo.Id, new websocketModel(socket));
            };
            socket.OnMessage = (msg) =>
            { 
                if (!msg.Contains("ConcentratorNo"))
                {
                    socket.Send($"请发送点抄指令数据");
                    return;
                }
                var message = JsonConvert.DeserializeObject<MessageModel>(msg);
                   
                var msglist = Sockets[socket.ConnectionInfo.Id].msgdic;
                if (msglist.Any(s => s.ConcentratorNo.Equals(message.Content.ConcentratorNo) && s.WaterMtrAddress.Equals(message.Content.WaterMtrAddress)))
                {
                    socket.Send($"不要重复发送，{ msg }");
                    return;
                }
                Sockets[socket.ConnectionInfo.Id].msgdic.Add(message.Content); 
            };
        });
    }
} 

public class websocketModel
{
     public websocketModel(IWebSocketConnection socket)
     {
         Socket = socket;
         SocketID = socket.ConnectionInfo.Id;
         msgdic = new ConcurrentBag<ClickToCopyModel>();
     }

     public Guid SocketID { get; set; }

     public IWebSocketConnection Socket { get; set; }


     /// <summary>
     /// 消息集合
     /// </summary>
     public ConcurrentBag<ClickToCopyModel> msgdic { get; set; }
}
```

此时，webclient→clientserver之间通信已经畅通无碍了，不论是webclient向webserver发消息还是webserver向webclient发消息都是ok的了，在上述代码中，每个连接发过来的消息都通过`msgdic`集合进行存储了，流程①已实现，接下来就可以看看webserver →clientserver之间的通信了。

#### 2. webserver→clientserver

webserver与clientserver之间就用普通的socket连接就好了，将webserver作为`socketclient`，clientserver作为`socketserver` ，啥也不说，贴代码了

webserver相关代码

```c#
public class ClientAsynSocket
{
    public static Socket ClientSocket;

    public static ConcurrentQueue<ClickToCopyModel> msglis = new ConcurrentQueue<ClickToCopyModel>();



    /// <summary>
    /// 接收的单个报文的最大长度 
    /// </summary> 
    public const int ReceiveBufferSize = 1166;


    public static void Init()
    {
        String IP = "127.0.0.1"; 
        var port = 9092;

        IPAddress ip = IPAddress.Parse(IP);  //将IP地址字符串转换成IPAddress实例
        ClientSocket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);//使用指定的地址簇协议、套接字类型和通信协议
        IPEndPoint endPoint = new IPEndPoint(ip, port); // 用指定的ip和端口号初始化IPEndPoint实例

        ClientSocket.BeginConnect(endPoint, new AsyncCallback(ConnectCallBack), ClientSocket);
    }

    private static void ConnectCallBack(IAsyncResult iar)
    {
        Socket client = (Socket)iar.AsyncState;
        try
        {
            client.EndConnect(iar);
            Recive();
        }
        catch (SocketException e)
        {
            Console.WriteLine("服务器程序未运行或服务器端口未开放");

        }
    }

    public static void Recive()
    {
        byte[] data = new byte[1024];
        try
        {
            ClientSocket.BeginReceive(data, 0, data.Length, SocketFlags.None,
            asyncResult =>
            {
                try
                {
                    int length = ClientSocket.EndReceive(asyncResult);
                    var msg=JsonConvert.DeserializeObject<ClickToCopyModel>(Encoding.GetEncoding("GB2312").GetString(data));
                    Console.WriteLine("接收到消息：" + Encoding.GetEncoding("GB2312").GetString(data));
                    msglis.Enqueue(msg);
                    Recive();
                }
                catch (SocketException e)
                {
                    if (e.ErrorCode == 10054)
                    {
                        Console.WriteLine("服务器已断线");
                    }
                    else
                    {
                        Console.WriteLine(e.Message);
                    }
                }
            }, null);
        }
        catch (Exception ex)
        {
            Console.WriteLine(ex.Message);
        }
    }

    public static void Send(string messagestr)
    {
        byte[] message = Encoding.Default.GetBytes(messagestr);  //通信时实际发送的是字节数组，所以要将发送消息转换字节
         ClientSocket.Send(message);
        Console.WriteLine("发送消息为:" + messagestr);
    } 
}
```

clientserver相关代码

```c#
public class ServerAsynSocket
{
    static Socket ServerSocket;

    /// <summary>
    /// 连接管理器
    /// </summary>
    public static ConcurrentDictionary<string, Connection> dic_conn = new ConcurrentDictionary<string, Connection>();


    /// <summary>
    /// 消息集合
    /// </summary>
    public ConcurrentQueue<ClickToCopyModel> msgdic = new ConcurrentQueue<ClickToCopyModel>();


    /// <summary>
    /// 监听队列长度
    /// </summary>
    public int ListenCount = 1000;


    /// <summary>
    /// 接收的单个报文的最大长度 
    /// </summary> 
    public const int ReceiveBufferSize = 1166;



    public void Init()
    {
        String IP = "127.0.0.1";
        var port = Convert.ToInt32(ConfigurationManager.AppSettings["ReadingMtrPort"]);

        IPAddress ip = IPAddress.Parse(IP);  //将IP地址字符串转换成IPAddress实例
        IPEndPoint endPoint = new IPEndPoint(ip, port);
        ServerSocket = new Socket(endPoint.AddressFamily, SocketType.Stream, ProtocolType.Tcp);
        ServerSocket.SetSocketOption(SocketOptionLevel.Tcp, SocketOptionName.NoDelay, true);
        ServerSocket.Bind(endPoint);
        ServerSocket.Listen(ListenCount);
        Console.WriteLine($"{endPoint.Address}:{endPoint.Port}已开启监听");
        Accept();
    }

    /// <summary>
    /// 接入请求
    /// </summary>
    void Accept()
    {
        //开启异步监听
        ServerSocket.BeginAccept(AcceptDone, null);
    }

    /// <summary>
    /// 完成接入请求事件
    /// </summary>
    /// <param name="result"></param>
    void AcceptDone(IAsyncResult result)
    {
        try
        {
            var clientSocket = ServerSocket.EndAccept(result);
            var conn = new Connection(clientSocket, ReceiveBufferSize);
            Receive(conn);
            Accept();//接着进行接收
        }
        catch (Exception ex)
        {
            Console.WriteLine(DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss") + "AcceptDone出错");
        }
    }

    /// <summary>
    /// 开始接收
    /// </summary>
    /// <param name="socket"></param>
    public void Receive(Connection connection)
    {
        try
        {
            connection.Socket.BeginReceive(
                   connection.Buffer,
                   0,
                   ReceiveBufferSize,
                   SocketFlags.None,
                   ReceiveDone,
                   connection
               );
        }
        catch (Exception ex)
        {
            Console.WriteLine(DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss") + "Receive出错");
        }
    }

    /// <summary>
    /// 完成接收
    /// </summary>
    /// <param name="result"></param>
    void ReceiveDone(IAsyncResult result)
    {
        try
        {
            var connection = (Connection)result.AsyncState;

            EndPoint iPEnd = connection.Socket.RemoteEndPoint;


            //通过获取本次回传的数据长度来截取数据，故可以重复利用该缓冲区而不需要清空
            var bytesTransferred = 0;
            try
            {
                bytesTransferred = connection.Socket.EndReceive(result);
            }
            catch (Exception ex)
            {
                if (dic_conn.Values.Any(s => s.Socket.Equals(connection.Socket)))
                {
                    Connection par = null;
                    dic_conn.TryRemove(iPEnd.ToString(), out par);
                }
                Console.WriteLine(DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss") + "远程客户端主动关闭");
                connection.Release();
                return;
            }

            if (bytesTransferred == 0)
            {
                //TCP返回0至少可以肯定对方关闭了写方向的socket；所以这里直接关闭socket,进行资源释放
                connection.Release();
                return;
            }

            //将接受到的报文拷贝出来
            byte[] info = new byte[bytesTransferred];
            Buffer.BlockCopy(
                                    connection.Buffer,
                                    0,
                                    info,
                                    0,
                                    bytesTransferred
                                );

            if (!dic_conn.Values.Any(s => s.Socket.Equals(connection.Socket)))
            {
                dic_conn.TryAdd(iPEnd.ToString(), connection);
            }
            var commandmsg = Encoding.GetEncoding("GB2312").GetString(info);

            Console.WriteLine($"{iPEnd}发送消息：" + commandmsg);

            //简单判断了一下是不是web网站服务端的连接发过来的数据
            if (commandmsg.Contains("ConcentratorNo"))
            {
                var model = JsonConvert.DeserializeObject<ClickToCopyModel>(commandmsg);

                Random random = new Random();
                var acc = Convert.ToDouble(random.Next() * 10);

                model.AccumVal = acc;
                msgdic.Enqueue(model);
            }
           


            //继续接收
            Receive(connection);

        }
        catch (Exception ex)
        {
            Console.WriteLine(DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss") + "ReceiveDone出错");
        }
    }

    public void Send(string ipaddress, string messagestr)
    {
        byte[] send = Encoding.Default.GetBytes(messagestr);

        if (dic_conn.Keys.Any(s => s.Equals(ipaddress)))
        {
            dic_conn.FirstOrDefault(s => s.Key.Equals(ipaddress)).Value.Socket.Send(send);
            Console.WriteLine("发送消息为：" + messagestr);
        } 
    }  
}
```

现在webclient→webserver和webserver→clientserver之间的通信就打通了，按之前三端通信流程图来讲，我们在webclient发送一个消息向webclient，webserver并没有转发给clientserver，流程②、③、④似乎都没有实现，只是简单的建立通信基础条件。

流程②的实现

```c#
public static void ServerStart()
{
	Task sendclientServer = Task.Run(() =>
    {
        while (true)
        {
            SendMessage();
        }
    });        
}
  		
  		
  		
/// <summary>
/// 发送给clientserver监听程序
/// </summary>
public static void SendMessage()
{
    if (Sockets!=null && Sockets.Count>0)
    {
        foreach (var item in Sockets.Values)
        { 
            foreach (var command in item.msgdic)
            {
                if (!command.IsDownCommand)
                {
                    var msg = JsonConvert.SerializeObject(command);
                    ClientAsynSocket.Send(msg);
                    command.IsDownCommand = true;
                }

            }
        }
    }
}
```

流程①将消息全部存到每个websocket连接的消息集合中，此处我们开启一个线程循环检查消息集合，将需要发送给clientserver，这样就可以啦。

流程③的实现

```c#
static void Main(string[] args)
{ 
    ServerAsynSocket server = new ServerAsynSocket();
    server.Init();

    Task task = Task.Run(() =>
    {
        while (true)
        {
            server.SendData();
        }
    });
    Console.ReadKey(); 
}

/// <summary>
/// 回复网站服务端
/// </summary>
public void SendData()
{
    while (true)
    {
        if (msgdic.IsEmpty)
        {
            return;
        }
        ClickToCopyModel data = new ClickToCopyModel();
        msgdic.TryDequeue(out data);

        var msg = JsonConvert.SerializeObject(data);
        byte[] send = Encoding.Default.GetBytes(msg);
        if (!dic_conn.IsEmpty)
        {
            dic_conn.First().Value.Socket.Send(send);
            Console.WriteLine($"抄表服务端发送指令：{ msg }");
        }
    }
}
```

webserver向clientserver发消息时 做了一些业务处理，然后存入`msgdic`中，而`SendData`就只需要将需要转发给webserver的消息转发出去就完成了流程③的任务了(clientserver回复webserver)。

流程④的实现

```c#
public static void ServerStart()
{
	Task sendclientServer = Task.Run(() =>
    {
        while (true)
        {
            Reply();
        }
    });        
}


/// <summary>
/// 回复webClient
/// </summary>
public static void Reply()
{
    if (Sockets != null && Sockets.Count > 0 && ClientAsynSocket.msglis != null && ClientAsynSocket.msglis.Count > 0)
    {
        ClickToCopyModel msg = null;
        ClientAsynSocket.msglis.TryDequeue(out msg);


        foreach (var item in Sockets.Values)
        {
            if (item.msgdic.Any(s => s.WaterMtrAddress.Equals(msg.WaterMtrAddress)))
            {
                ClickToCopyModel msgr = null;
                item.msgdic.TryTake(out msgr);

                string sendmsg = JsonConvert.SerializeObject(msg);
                item.Socket.Send(sendmsg);
            }
        }
    }
}   
```

最后webserver回复webclient了 就结束了整个通信流程。



### 源码

[下载](https://github.com/cn-wwl/Sockets/tree/sanduantongxin)





