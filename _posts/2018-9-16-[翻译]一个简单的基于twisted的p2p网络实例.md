---
layout: post
title: "[翻译]一个简单的基于twisted的p2p网络实例"
date: 2018-9-16
comments: true
---
> 这是一篇翻译自```https://benediktkr.github.io/dev/2016/02/04/p2p-with-twisted.html```的文章
> 注意到文章中的代码是使用python2写成的，所以如果要在python3运行需要做出一些修改。

让我们来用python构建一个简单的p2p网络吧！它可以发现其他的节点并且通过这个网络连接它们。我们将会使用[twisted](https://twistedmatrix.com/)来搭建这个网络。

我受到[Bitcoin Developer Documentation](https://bitcoin.org/en/developer-documentation)(比特币开发者文档)的启发，知道了要怎么搭建一个p2p网络。我写了这个网络的具体实现，已经放在了这里了:[benediktkr](https://github.com/benediktkr)/[ncpoc](https://github.com/benediktkr/ncpoc)

既然是在一个p2p网络当中，每个成员都得同时作为服务端和客户端。而第一个主机需要些做些事情去找到对方，并且也要意识到自己连接自己的情况的发生。
## 使用UUID确认成员的身份
因为对外的TCP连接会被分配随机的端口，我们就不能够依靠```ip:端口```这样的方式来作为对每个在p2p网络中的成员的认证了。所以我们要给每一个成员随机分配一个UUID来进行会话控制。一个UUID有128比特的大小，这也算的上是合理的不大的开销吧。
```python
>>> from uuid import uuid4
>>> generate_nodeid = lambda: str(uuid4())
'a46de8d6-177e-4644-a711-63d182fdbade'
```
## 定义一个简单的基于Twisted的协议
我们只会在真正要用到Twisted的场合里谈及它。如果你想了解更多关于如何使用Twisted搭建服务端及构建通讯协议的知识，[Writing Servers in the Twisted documenation](https://twistedmatrix.com/documents/current/core/howto/servers.html)(Twisted服务端编写文档)是一个好去处。(译者注:如果项目的文档和项目同样是遵循MIT协议的话，迟些时候我将会翻译这篇文章)
类```Factory```的实例是会要持续的存在于连接当中的，所以它的实例是我们存储像成员列表和会话时需要的UUID这类信息的地方。在这里，一个新的```MyFactory```的实例将会在每次连接发生后被创建。
```python
from twisted.internet.endpoints import TCP4ServerEndpoint
from twisted.internet.protocol import Protocol, Factory
from twisted.internet import reactor

class MyProtocol(Protocol):
    def __init__(self, factory):
        self.factory = factory
        self.nodeid = self.factory.nodeid

class MyFactory(Factory):
    def startFactory(self):
        self.peers = {}
        self.nodeid = generate_nodeid()

    def buildProtocol(self, addr):
        return NCProtocol(self)

endpoint = TCP4ServerEndpoint(reactor, 5999)
endpoint.listen(MyFactory())
```
这将会为一个在```Localhost:5999```尚为空的通讯协议定义一个监听器。至今为止它还不是那么有用。因为我们希望各个节点之间可以相互传输信息来进行对话。为此，我们用JSON流来创建这些发送的信息。这是一种非常容易传播并且十分具有可读性的方式。

那么，从为每一个节点创建一个相互介绍的```hello```信息开始吧。信息的形式被包含在这里面，这样每个主机就很容易知道要怎么去处理它。
```json
{
    "nodeid" : "a46de8d6-177e-4644-a711-63d182fdbade",
    "msgtype": "hello",
}
```
(译者注: 这里面的nodeid就是上面生成的UUID)
现在我们需要修改我们的代码以使它可以处理来自新的链接的```hello```信息并且保存连接的UUID的列表。（就是上文代码框中类```MyFactory```的```peers```属性。
```python
import json

class MyProtocol(Protocol):
    def __init__(self, factory):
        self.factory = factory
        self.state = "HELLO"
        self.remote_nodeid = None
        self.nodeid = self.factory.nodeid

    def connectionMade(self):
        print "Connection from", self.transport.getPeer()

        def connectionLost(self, reason):
        if self.remote_nodeid in self.factory.peers:
            self.factory.peers.pop(self.remote_nodeid)
        print self.nodeid, "disconnected"

    def dataReceived(self, data):
        for line in data.splitlines():
            line = line.strip()
            if self.state == "HELLO":
                self.handle_hello(line)
                self.state = "READY"

    def send_hello(self):
        hello = json.puts({'nodeid': self.nodeid, 'msgtype': 'hello'})
        self.transport.write(hello + "\n")

    def handle_hello(self, hello):
        hello = json.loads(hello)
        self.remote_nodeid = hello["nodeid"]
        if self.remote_nodeid == self.nodeid:
            print "Connected to myself."
            self.transport.loseConnection()
        else:
            self.factory.peers[self.remote_nodeid] = self
```
这样，程序就可以发送、解读```hello```信息并且持续跟踪p2p网络里的成员了。
## 协议通讯
现在我们有了一个看起来像是可以充当处理```hello```信息的“服务端”的东西了。不过我们还要让它同时充当“客户端”的角色。好在Twisted是一个包含大量库的一系列工具，所以我们几乎可以随心所欲地使用Twisted的回调函数来实现我们的目的。
```python
from twisted.internet.endpoints import TCP4ClientEndpoint, connectProtocol

def gotProtocol(p):
    """The callback to start the protocol exchange. We let connecting
    nodes start the hello handshake"""
    """这一个回调函数是用于开启通讯协议的交换的。
       通过它，我们可以让连接了的节点发送hello的握手通讯。"""
    p.send_hello()

point = TCP4ClientEndpoint(reactor, "localhost", 5999)
d = connectProtocol(point, MyProtocol())
d.addCallback(gotProtocol)
```
（译者注:根据TCP协议，两台主机在连接时会有三次握手的过程。在这里原作者可能有所简化，只是通过这样一个```hello```信息的通讯代替这个过程。）
## 网络引导
如果我们同时运行这些部分，一次成功的握手通讯将有希望被建立，并且用于监听的```factory```会输出这样的信息:
```
Connection from 127.0.0.1:58790
Connected to myself.
a46de8d6-177e-4644-a711-63d182fdbade disconnected.
```
要有一个p2p网络，我们需要不只一个的实例。最简单的让网络中最初的成员找到彼此的办法只需要拥有一个包含```host:prot```格式的列表，然后遍历这个列表的同时尝试连接列表中的成员，并且对于这当中每一个实例都分配一个不同的监听端口。
```python
BOOTSTRAP_LIST = [ "localhost:5999"
                 , "localhost:5998"
                 , "localhost:5997" ]

for bootstrap in BOOTSTRAP_LIST:
    host, port = bootstrap.split(":")
    point = TCP4ClientEndpoint(reactor, host, int(port))
    d = connectProtocol(point, MyProtocol())
    d.addCallback(gotProtocol)
reactor.run()
```
好了，现在我们程序用当中的实例引导了这个网络。
## Ping
那我们就通过增加```ping```信息的方式来让它们做些有用的事情吧——就通过一个空的有效负载信息。
(译者注:此为payload。实际上是一种安全测试数据。通常在信息的传输过程中，会在每一批数据的头和尾增加一些辅助信息，比如数据量的大小、校验位等等，以形成一个数据包。这些被附以辅助信息的原始信息称为payload。它可以帮助我们实现更可靠的数据传输。在这里原作者没有添加这样的辅助信息，仅用ping信息来标识连接，所以说是"空"的有效负载)

相应地，我们也同样需要一个```pong```信息作为回应。
```json
{
    "msgtype": "ping"
}
```
```json
{
    "msgtype": "pong"
}
```
当我们通过发送一个```ping```信息来连接节点的时候，注意到我们会收到```pong```作为回复。这对于发现离线客户端有实际意义，为我们简简单单的```ping```/```pong```的信息流赋予了真正的作用。我们同样要使用Twisted的```LoopingCall```来向节点发送周期性的```ping```。
因为对于每一次连接都会创建一个```MyProtocol```的实例，我们就不需要对所有的连接循环地操作，我们可以在已经抽象化的连接（译者注:指的是每个被创建的```MyProtocol```的实例）中实现。

我再次强调，Twisted已经为我们做了许多像抬高重物般的粗活了。我们要做的，就是修改```MyProtocol```类来操作网络当中结构性的工作。
```python
from time import time

from twisted.internet.task import LoopingCall

class MyProtocol(Protocol):
    def __init__(self, factory):
        self.factory = factory
        self.state = "HELLO"
        self.remote_nodeid = None
        self.nodeid = self.factory.nodeid
        self.lc_ping = LoopingCall(self.send_ping)
        self.lastping = None

    def connectionMade(self):
        print "Connection from", self.transport.getPeer()

    def connectionLost(self, reason):
        if self.remote_nodeid in self.factory.peers:
            self.factory.peers.pop(self.remote_nodeid)
            self.lc_ping.stop()
        print self.nodeid, "disconnected"

    def dataReceived(self, data):
        for line in data.splitlines():
            line = line.strip()
            msgtype = json.loads(line)['msgtype']
            if self.state == "HELLO" or msgtype == "hello":
                self.handle_hello(line)
                self.state = "READY"
            elif msgtype == "ping":
                self.handle_ping()
            elif msgtype == "pong":
                self.handle_pong()

    def send_hello(self):
        hello = json.puts({'nodeid': self.nodeid, 'msgtype': 'hello'})
        self.transport.write(hello + "\n")

    def send_ping(self):
        ping = json.puts({'msgtype': 'ping'})
        print "Pinging", self.remote_nodeid
        self.transport.write(ping + "\n")

    def send_pong(self):
        ping = json.puts({'msgtype': 'pong'})
        self.transport.write(pong + "\n")

    def handle_ping(self, ping):
        self.send_pong()

   def handle_pong(self, pong):
        print "Got pong from", self.remote_nodeid
        ###Update the timestamp
        ###更新时间戳
        self.lastping = time()

    def handle_hello(self, hello):
        hello = json.loads(hello)
        self.remote_nodeid = hello["nodeid"]
        if self.remote_nodeid == self.nodeid:
            print "Connected to myself."
            self.transport.loseConnection()
        else:
            self.factory.peers[self.remote_nodeid] = self
            self.lc_ping.start(60)
```
现在我们引导了一个成员间每分钟进行```ping```的操作的对等网络了。但是成员间只能通过引导来发现对方。
## 发现节点
当前的版本只具有连接已经事先定义的列表中的节点的能力。我们还需要添加额外的函数来使之称为真正的p2p网络。
* 我们需要区分我们将要连接的成员和已经和我们连接的成员。对外的TCP连接被分配随机的端口以至于我们不能连接。使用字符串类型的名字来描述这两种成员将会带来不可避免的难以理解的错误，所以我们使用整形数据```1```来描述前者，```2```来描述后者。
* 我们需要公布我们自己正在监听的```ip:port```的信息对。（但事实是，NAT会在运行时大部分时间里对此有所影响，但我们在这里要先忽略这一点）（译者注: NAT,Network Address Translation,网络地址转换。使用较少的公有ip来代表较多私有ip的方式以实现与英特网的连接。由于以“较少”代替“较多”，我们损失了一部分信息，所以影响了ip对的交换）
* 定义一个信息```addr```来公布成员的信息，和一个```getaddr```信息来请求新成员的信息。

完成这些改变之后，我们的```Protocol```的实例就会像这个样子
```python
class MyProtocol(Protocol):
    def __init__(self, factory, peertype):
        self.factory = factory
        self.state = "HELLO"
        self.remote_nodeid = None
        self.nodeid = self.factory.nodeid
        self.lc_ping = LoopingCall(self.send_ping)
        self.peertype = peertype
        self.lastping = None

    def connectionMade(self):
        remote_ip = self.transport.getPeer()
        host_ip = self.transport.getHost()
        self.remote_ip = remote_ip.host + ":" + str(remote_ip.port)
        self.host_ip = host_ip.host + ":" + str(host_ip.port)
        print "Connection from", self.transport.getPeer()

    def connectionLost(self, reason):
        if self.remote_nodeid in self.factory.peers:
            self.factory.peers.pop(self.remote_nodeid)
            self.lc_ping.stop()
        print self.nodeid, "disconnected"

    def dataReceived(self, data):
        for line in data.splitlines():
            line = line.strip()
            msgtype = json.loads(line)['msgtype']
            if self.state == "HELLO" or msgtype == "hello":
                self.handle_hello(line)
                self.state = "READY"
            elif msgtype == "ping":
                self.handle_ping()
            elif msgtype == "pong":
                self.handle_pong()
            elif msg_type == "getaddr":
                self.handle_getaddr()


    ###The methods for ping and pong remain unchanged and are omitted
    ###for brevity
    ###ping和pong的方法没有发生改变。为了简洁，在这里省略它们的实现。

    def send_addr(self, mine=False):
        now = time()
        if mine:
            peers = [self.host_ip]
        else:
            peers = [(peer.remote_ip, peer.remote_nodeid)
                     for peer in self.factory.peers
                     if peer.peertype == 1 and peer.lastping > now-240]
        addr = json.puts({'msgtype': 'addr', 'peers': peers})
        self.transport.write(peers + "\n")

    def send_addr(self, mine=False):
        now = time()
        if mine:
            peers = [self.host_ip]
        else:
            peers = [(peer.remote_ip, peer.remote_nodeid)
                     for peer in self.factory.peers
                     if peer.peertype == 1 and peer.lastping > now-240]
        addr = json.puts({'msgtype': 'addr', 'peers': peers})
        self.transport.write(peers + "\n")

    def handle_addr(self, addr):
        json = json.loads(addr)
        for remote_ip, remote_nodeid in json["peers"]:
            if remote_node not in self.factory.peers:
                host, port = remote_ip.split(":")
                point = TCP4ClientEndpoint(reactor, host, int(port))
                d = connectProtocol(point, MyProtocol(2))
                d.addCallback(gotProtocol)

    def handle_getaddr(self, getaddr):
        self.send_addr()

    def handle_hello(self, hello):
        hello = json.loads(hello)
        self.remote_nodeid = hello["nodeid"]
        if self.remote_nodeid == self.nodeid:
            print "Connected to myself."
            self.transport.loseConnection()
        else:
            self.factory.peers[self.remote_nodeid] = self
            self.lc_ping.start(60)
            ###inform our new peer about us
            ###将我们自己的信息通知给新成员
            self.send_addr(mine=True)
            ###and ask them for more peers
            ###向他们所要更多的成员
            self.send_getaddr()
```
## 结语
实现了一个有以下功能的简单的对等网络:
  * 通过事先声明的列表引导
  * 与其他节点连接并进行握手通讯
  * 向成员索取节点信息并连接
  * 对外公布自己连接的节点及它自身节点的信息
  * ping对等网络的成员
