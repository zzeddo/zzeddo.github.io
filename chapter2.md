# 2장-소켓 및 패턴(Sockets and Patterns)
;In Chapter 1 - Basics we took ØMQ for a drive, with some basic examples of the main ØMQ patterns: request-reply, pub-sub, and pipeline. In this chapter, we're going to get our hands dirty and start to learn how to use these tools in real programs.

1장 - 기본에서 몇 가지 ØMQ 패턴의 기본 예제로 ØMQ의 세계로 뛰어들었습니다 : 요청-응답, 발행-구독, 파이프라인. 이장에서는 우리의 손을 더럽히면서 실제 프로그램에서 이러한 도구들을 어떻게 사용하는지 배우게 될 것입니다.

;We'll cover:

다루는 내용은 다음과 같습니다.

;* How to create and work with ØMQ sockets.
;* How to send and receive messages on sockets.
;* How to build your apps around ØMQ's asynchronous I/O model.
;* How to handle multiple sockets in one thread.
;* How to handle fatal and nonfatal errors properly.
;* How to handle interrupt signals like Ctrl-C.
;* How to shut down a ØMQ application cleanly.
;* How to check a ØMQ application for memory leaks.
;* How to send and receive multipart messages.
;* How to forward messages across networks.
;* How to build a simple message queuing broker.
;* How to write multithreaded applications with ØMQ.
;* How to use ØMQ to signal between threads.
;* How to use ØMQ to coordinate a network of nodes.
;* How to create and use message envelopes for pub-sub.
;* Using the HWM (high-water mark) to protect against memory overflows.

* ØMQ 소켓을 만들고 사용하는 방법
* 소켓상에서 메시지 송/수신 방법
* ØMQ 비동기 I/O 모델상에서 응용프로그램 개발하기
* 하나의 스레드상에서 다중 소켓 처리하는 방법
* 치명적인 및 비치명적인 오류에 적절하게 대응하기
* Ctrl-C와 같은 인터럽트 처리하기
* ØMQ 응용프로그램을 깨끗하게 종료하기
* ØMQ 응용프로그램의 메모리 누수 여부를 확인하기
* 멀티파트 메시지 송/수신 방법
* 네트워크상에서 메시지 전달하기
* 단순 메시지 대기열 브로커 개발하기
* ØMQ에서 멀티스레드 응용프로그램 개발하기
* ØMQ를 스레드 간 신호에 사용하기
* ØMQ를 네트워크상 노드들 간 협업에 사용하기
* 발행-구독에서 메시지 봉투 생성하고 사용하기
* 메모리 오버플로우에 대비하여 최고수위 표시(HWM,high-water mark) 사용하기

## 소켓 API(The Socket API)
;To be perfectly honest, ØMQ does a kind of switch-and-bait on you, for which we don't apologize. It's for your own good and it hurts us more than it hurts you. ØMQ presents a familiar socket-based API, which requires great effort for us to hide a bunch of message-processing engines. However, the result will slowly fix your world view about how to design and write distributed software.

솔직히 말하자면, ØMQ는 당신에게 일종의 유인 상술을 쓰는데, 그것에 대하여 사과하지 않겠습니다.
ØMQ는 익숙한 소켓 기반 API를 제공하고 내부에 일련의 메시지 처리 엔진을 가지고 있으며 당신이 어떻게 분산 소프트웨어를 설계하고 개발하는지에 따라 점차 당신의 세계관으로 만들 수 있습니다.

> [옮긴이] 유인 상술(switch and bait)은  값싼 상품을 광고해서 소비자를 끌어들인 뒤 비싼 상품을 사게 하는 상술입니다.

;Sockets are the de facto standard API for network programming, as well as being useful for stopping your eyes from falling onto your cheeks. One thing that makes ØMQ especially tasty to developers is that it uses sockets and messages instead of some other arbitrary set of concepts. Kudos to Martin Sustrik for pulling this off. It turns "Message Oriented Middleware", a phrase guaranteed to send the whole room off to Catatonia, into "Extra Spicy Sockets!", which leaves us with a strange craving for pizza and a desire to know more.

소켓은 네트워크 프로그래밍에서는 표준 API이며 눈꺼풀처럼 눈알이 빰에 떨어지는 것을 막을 정도로 유용합니다. 
특히 개발자에게 ØMQ  매력적인 점은 다른 개념 대신 소켓과 메시지를 사용한다란 것이며, 이러한 개념을 이끌어낸 [Martin Sustrik](http://250bpm.com/)에게 감사하고 싶습니다. 
 "메시지 기반 미들웨어"로 방 전체를 긴장감으로 채울 것 문구를 "특별히 매운 소켓!"으로 변경하여 피자에 대한 이상한 갈망과 더 많은 것을 알고 싶게 합니다. 

;Like a favorite dish, ØMQ sockets are easy to digest. Sockets have a life in four parts, just like BSD sockets:

좋아하는 요리처럼, ØMQ 소켓도 쉽게 이해할 수 있습니다. 
ØMQ 소켓도 BSD 소켓과 같이 4개의 생명 주기가 있습니다.

;* Creating and destroying sockets, which go together to form a karmic circle of socket life (see zmq_socket(), zmq_close()).
;* Configuring sockets by setting options on them and checking them if necessary (see zmq_setsockopt(), zmq_getsockopt()).
;* Plugging sockets into the network topology by creating ØMQ connections to and from them (see zmq_bind(), zmq_connect()).
;* Using the sockets to carry data by writing and receiving messages on them (see zmq_send(), zmq_recv()).

* 소켓 생성 및 파괴를 합니다.
 - 소켓의 운명의 생명 주기를 만듭니다(`zmq_socket()`, `zmq_close()` 참조).
* 소켓 구성 옵션 설정합니다.
 - 필요할 경우 설정 필요합니다(`zmq_setsockopt()` 참조).
* 소켓 연결을 합니다.
 - ØMQ 연결을 생성하여 소켓을 네트워크에 참여시킵니다(`zmq_bind()`, `zmq_connect()` 참조).
* 소켓을 통한 메시지 송/수신합니다.
 - 다양한 통신 패턴에서 메세지 송신과 수신에 사용됩니다.(`zmq_msg_send()`, `zme_msg_recv()` 참조)

;Note that sockets are always void pointers, and messages (which we'll come to very soon) are structures. So in C you pass sockets as-such, but you pass addresses of messages in all functions that work with messages, like zmq_send() and zmq_recv(). As a mnemonic, realize that "in ØMQ, all your sockets are belong to us", but messages are things you actually own in your code.

소켓은 향상 void 포인터이며, 메시지들은 구조체이며 C 언어에서 `zmq_msg_send()`에 메시지의 주소를 전달하거나 `zmq_msg_recv()` 메시지의 주소를 반환받습니다.
기억하기 쉽게 ØMQ에서 모든 소켓은 우리에게 속해 있지만 메시지들은 소스 코드상에 있습니다.

;Creating, destroying, and configuring sockets works as you'd expect for any object. But remember that ØMQ is an asynchronous, elastic fabric. This has some impact on how we plug sockets into the network topology and how we use the sockets after that.

소켓 생성, 소멸 및 구성은 모든 객체에 대해 예상대로 동작하지만, ØMQ는 비동기식이며 소켓을 네트워크 토폴로지에 연결하는 방법에 따라 소켓을 사용하는데 영향을 미칩니다.

### 네트워크상에 소켓 넣기(Plugging Sockets into the Topology)
;To create a connection between two nodes, you use zmq_bind() in one node and zmq_connect() in the other. As a general rule of thumb, the node that does zmq_bind() is a "server", sitting on a well-known network address, and the node which does zmq_connect() is a "client", with unknown or arbitrary network addresses. Thus we say that we "bind a socket to an endpoint" and "connect a socket to an endpoint", the endpoint being that well-known network address.

2개의 노드 간의 연결을 생성하기 위하여, 한쪽 노드에 `zmq_bind()`를  그리고 다른 쪽 노드에 `zmq_connect()`를 사용할 수 있습니다. 일반적으로 `zmq_bind()`를 수행하는 노드를 서버(네트워크 주소가 고정됨)라고 하며 `zmq_connect()`를 수행하는 노드를 클라이언트(네트워크 주소가 모르거나 임시)라고 합니다.
우리는 "단말(endpoint)에 소켓을 바인딩"과 "단말에 소켓을 연결"라고 말하며, 단말은 네트워크 주소를 알 수 있어야 합니다.

;ØMQ connections are somewhat different from classic TCP connections. The main notable differences are:

ØMQ 연결은 전통적인 TCP 연결과 다소 차이가 있으며 주요한 차이는 다음과 같습니다.

;* They go across an arbitrary transport (inproc, ipc, tcp, pgm, or epgm). See zmq_inproc(), zmq_ipc(), zmq_tcp(), zmq_pgm(), and zmq_epgm().
;* One socket may have many outgoing and many incoming connections.
;* There is no zmq_accept() method. When a socket is bound to an endpoint it automatically starts accepting connections.
;* The network connection itself happens in the background, and ØMQ will automatically reconnect if the network connection is broken (e.g., if the peer disappears and then comes back).
;* Your application code cannot work with these connections directly; they are encapsulated under the socket.

* ØMQ는 임의의 전송방식(inproc, ipc, tcp, pgm, epgm)을 교차하여 사용할 수 있습니다.
 - `zmq_inproc()`, `zmq_ipc()`, `zmq_tcp()`, `zmq_pgm()`, `zmq_epgm()` 참조
* 하나의 소켓에서 여러 개의 송/수신 연결들(connections)을 가질 수 있습니다.
* `zmq_accept()`가 같은 함수가 없습니다.
 - 소켓이 단말에 바인딩되면 자동으로 연결을 수락합니다.
* ØMQ는 네트워크 연결을 백그라운드로 수행하며, 네트워크 연결이 끊기면 자동으로 재연결합니다(예를 들어 통신 대상이 사라졌다가 다시 돌아올 경우).
* 응용프로그램 코드가 이러한 연결에 대해 직접 작업하지 않습니다. : ØMQ 소켓에 내장되어 있습니다.

;Many architectures follow some kind of client/server model, where the server is the component that is most static, and the clients are the components that are most dynamic, i.e., they come and go the most. There are sometimes issues of addressing: servers will be visible to clients, but not necessarily vice versa. So mostly it's obvious which node should be doing zmq_bind() (the server) and which should be doing zmq_connect() (the client). It also depends on the kind of sockets you're using, with some exceptions for unusual network architectures. We'll look at socket types later.

일반적인 네트워크 아키텍처상에서 클라이언트/서버 구성할 때, 서버는 정적 형태, 클라이언트는 동적 형태로, 각 노드에서 서버의 경우 `zmq_bind()`, 클라이언트에서는 `zmq_connect()`을 수행하지만 어떤 종류의 소켓(예 : REQ-REP, PUB-SUB, PUSH-PULL, ROUTER-DEALER)을 사용하는지에 연관되며, 특이한 네트워크 아키텍처에서는 예외가 있습니다. 이러한 소켓 유형을 나중에 살펴보도록 하겠습니다.

;Now, imagine we start the client before we start the server. In traditional networking, we get a big red Fail flag. But ØMQ lets us start and stop pieces arbitrarily. As soon as the client node does zmq_connect(), the connection exists and that node can start to write messages to the socket. At some stage (hopefully before messages queue up so much that they start to get discarded, or the client blocks), the server comes alive, does a zmq_bind(), and ØMQ starts to deliver messages.

서버를 시작하기 전에 클라이언트를 수행하는 경우, 전통적인 네트워킹에서는 오류가 발생하지만, ØMQ의 경우 순서에 상관없이 시작과 중단을 할 수 있습니다. 클라이언트 노드에서 `zmq_connect()`를 수행하여 성공하면 소켓에 메시지를 전달하기 시작합니다. 일정 단계(희망하건대 메시지가 너무 많이 대기열에 쌓이게 되어 버리거나 차단되기 전까지)까지, 서버가 기동 되어 `zmq_bind()`를 하게 되면 ØMQ는 메시지를 전송하게 됩니다.

;A server node can bind to many endpoints (that is, a combination of protocol and address) and it can do this using a single socket. This means it will accept connections across different transports:

서버 노드는 다수의 단말들(통신규약과 네트워크 주소의 조합)을 바인딩할 수 있으며 이것도 하나의 소켓을 통해 가능합니다. 즉 서로 다른 전송방식(inproc, ipc, pgm, tcp 등)으로 연결을 수락한다.

```cpp
zmq_bind (socket, "tcp://*:5555");
zmq_bind (socket, "tcp://*:9999");
zmq_bind (socket, "inproc://somename");
```

;With most transports, you cannot bind to the same endpoint twice, unlike for example in UDP. The ipc transport does, however, let one process bind to an endpoint already used by a first process. It's meant to allow a process to recover after a crash.

UDP를 제외하고 대부분의 전송방식에서 동일한 단말을 두 번 바인딩할 수 없습니다.
그러나 ipc(프로세스 간 통신) 전송계층은 하나의 프로세스가 첫 번째 프로세스에서 이미 사용된 단말에 바인딩하게 합니다. 이것은 프로세스 간의 충돌에 대비하여 복구하기 위한 목적입니다.

;Although ØMQ tries to be neutral about which side binds and which side connects, there are differences. We'll see these in more detail later. The upshot is that you should usually think in terms of "servers" as static parts of your topology that bind to more or less fixed endpoints, and "clients" as dynamic parts that come and go and connect to these endpoints. Then, design your application around this model. The chances that it will "just work" are much better like that.

ØMQ는 어느 쪽이 바인딩되고 어느 쪽이 연결되는지에 대해 중립을 유지하려 하지만 차이점이 있습니다. 나중에 더 자세히 살펴보겠습니다. 일반적으로 "서버"는 다소 고정된 단말에 바인딩하고  "클라이언트"는 동적 요소인 단말에 연결하는 것으로 생각합니다. 동적/정적 사용 모델에 따라 응용프로그램을 설계하십시오. 그러면 "그냥 작동"할 가능성이 높습니다.

;Sockets have types. The socket type defines the semantics of the socket, its policies for routing messages inwards and outwards, queuing, etc. You can connect certain types of socket together, e.g., a publisher socket and a subscriber socket. Sockets work together in "messaging patterns". We'll look at this in more detail later.

소켓은 유형이 있고 소켓 유형에 따라 정의되는 의미는 네트워크 상에서 내부/외부 메시지 라우팅 정책이나 대기열 저장 등입니다. 특정 소켓 유형은 함께 사용할 수 없으며 예를 들면 PUB 소켓과 SUB 소켓. 


;It's the ability to connect sockets in these different ways that gives ØMQ its basic power as a message queuing system. There are layers on top of this, such as proxies, which we'll get to later. But essentially, with ØMQ you define your network architecture by plugging pieces together like a child's construction toy.

ØMQ는 다양한 형태로 소켓을 연결할 수 있으며 메시징 대기열 시스템으로써 기본 역량을 제공합니다. 근본적으로 ØMQ는 다양한 구성 요소(소켓, 연결, 프록시 등)들을 필요에 따라 끼워 맞추어 필요한 네트워크 아키텍처를 정의할 수 있게 합니다.

### 메시지 송/수신(Sending and Receiving Messages)
;To send and receive messages you use the zmq_msg_send() and zmq_msg_recv() methods. The names are conventional, but ØMQ's I/O model is different enough from the classic TCP model that you will need time to get your head around it.

메시지 송/수신을 위하여 `zmq_msg_send()`와 `zmq_msg_recv()` 함수를 사용할 수 있습니다. 전통적인 방식의 명칭이지만 ØMQ I/O 모델은 기존 TCP 모델과는 확연히 차이가 있습니다.

Figure 9 - 1:1 TCP 소켓(TCP sockets are 1 to 1)

![1:1 TCP 소켓](images/fig9.svg)

;Let's look at the main differences between TCP sockets and ØMQ sockets when it comes to working with data:

TCP 소켓과 ØMQ 소켓의 주요 차이는 데이터 처리 방식입니다.

;* ØMQ sockets carry messages, like UDP, rather than a stream of bytes as TCP does. A ØMQ message is length-specified binary data. We'll come to messages shortly; their design is optimized for performance and so a little tricky.
;* ØMQ sockets do their I/O in a background thread. This means that messages arrive in local input queues and are sent from local output queues, no matter what your application is busy doing.
;* ØMQ sockets have one-to-N routing behavior built-in, according to the socket type.

* ØMQ 소켓은 메시지들을 전달합니다.
마치 UDP처럼. 다소 TCP의 바이트 스트림처럼 동작한다. ØMQ 메시지는 길이가 지정된 바이너리 데이터로 성능 최적화를 위해 다소 까다롭게 설계되었습니다.
* ØMQ 소켓의 I/O는 백그라운드 스레드로 동작합니다.
응용프로그램이 무엇을 바쁘게 처리하더라도 I/O 스래드는 메시지들을 입력 대기열로부터 수신하고, 출력 대기열로에서 송신하게 합니다.
* ØMQ 소켓들은 소켓 유형에 따라 1:N 라우팅이 내장되어 있다.

;The zmq_send() method does not actually send the message to the socket connection(s). It queues the message so that the I/O thread can send it asynchronously. It does not block except in some exception cases. So the message is not necessarily sent when zmq_send() returns to your application.

`zmq_send()` 함수는 실제 메시지를 소켓 연결에서 전송하지 않고 메시지를 대기열에 쌓아 I/O 스레드가 비동기로 전송할 수 있게 합니다. 이러한 동작은 일부 예외 상황을 제외하고는 차단되지 않습니다. 그래서 메시지를 `zmq_send()`로 전송할 때 반환값이 필요 없습니다.

### 유니케스트 전송방식(Unicast Transports)
;ØMQ provides a set of unicast transports (inproc, ipc, and tcp) and multicast transports (epgm, pgm). Multicast is an advanced technique that we'll come to later. Don't even start using it unless you know that your fan-out ratios will make 1-to-N unicast impossible.

ØMQ는 일련의 유니케스트 전송방식(inproc, ipc, tcp)과 멀티캐스트 통신방식(epgm, pgm)을 제공합니다. 멀티캐스트는 진보된 기술로 나중에 다루지만 전개 비율에 따라 1:N 유니케스트가 불가하다는 것을 모른다면 시작조차 하지 마시기 바랍니다.

;For most common cases, use tcp, which is a disconnected TCP transport. It is elastic, portable, and fast enough for most cases. We call this disconnected because ØMQ's tcp transport doesn't require that the endpoint exists before you connect to it. Clients and servers can connect and bind at any time, can go and come back, and it remains transparent to applications.

대부분의 경우 TCP 사용하며 "비연결성 TCP 전송계층"라고 하며, 대부분의 경우 충분히 빠르고 유연하고 간편합니다. 우리가 "비연결성"라고 부르는 것은 ØMQ의 TCP 전송방식은 단말이 연결을 하기전에 존재할 필요가 없기 때문입니다. 클라이언트들과 서버들은 언제든지 연결하거나 바인딩할 수 있으며, 나가고 들어오는 것이 가능하며 응용프로그램에 투명하게 유지됩니다.

;The inter-process ipc transport is disconnected, like tcp. It has one limitation: it does not yet work on Windows. By convention we use endpoint names with an ".ipc" extension to avoid potential conflict with other file names. On UNIX systems, if you use ipc endpoints you need to create these with appropriate permissions otherwise they may not be shareable between processes running under different user IDs. You must also make sure all processes can access the files, e.g., by running in the same working directory.

프로세스 간 ipc 전송방식도 TCP처럼 "비연결성"이지만 한 가지 제약 사항은 원도우 환경에서는 아직 동작하지 않습니다. 편의상 단말의 명칭들을 ".ipc" 확장명으로 사용하여 다른 파일명과 잠재적인 충돌을 피하도록 하겠습니다. 유닉스 시스템에서 ipc 단말을 사용하려 한다면 적절한 접근 권한을 생성이 필요하며 그렇지 않다면 다른 사용자 식별자(ID)하에 구동되는 프로세스 간에 공유는 허용되지 않을 것입니다. 그리고 모든 프로세스들은 파일들(예를 들면 동일 디렉터리에서 존재하는 파일들)에 대하서 접근할 수 있어야 합니다.

;The inter-thread transport, inproc, is a connected signaling transport. It is much faster than tcp or ipc. This transport has a specific limitation compared to tpc and icp: the server must issue a bind before any client issues a connect. This is something future versions of ØMQ may fix, but at present this defines how you use inproc sockets. We create and bind one socket and start the child threads, which create and connect the other sockets.

스레드 간 inproc 전송방식은  "연결된 신호 전송계층"이며 tcp 혹은 ipc 보다 훨씬 빠르지만 tcp나 ipc와 비교했을 때 특정 제약을 가지고 있습니다 : 서버는 반드시 클라이언트가 연결을 요청하기 전에 바인딩이 되어야 한다. 이것은 미래의 ØMQ 버전에서 개선되겠지만 현재는 어떻게 inproc 소켓을 사용해야 하는지를 정의합니다. 우리는 부모 스레드에서 소켓을 생성하고 바인딩하고 자식 스레드를 시작하여 소켓을 생성하고 연결하게 합니다.

### ØMQ는 중립적인 전송수단이 아니다(ØMQ is Not a Neutral Carrier).
;A common question that newcomers to ØMQ ask (it's one I've asked myself) is, "how do I write an XYZ server in ØMQ?" For example, "how do I write an HTTP server in ØMQ?" The implication is that if we use normal sockets to carry HTTP requests and responses, we should be able to use ØMQ sockets to do the same, only much faster and better.

ØMQ을 처음 접하는 사람들의 공통적인 질문(이것은 나 스스로에게도 한 것임)은 "ØMQ로 XYZ 서버를 개발하는 방법은 무엇이지?"입니다. 예를 들면 "ØMQ로 HTTP 서버를 개발하는 방법은 무엇이지?"와 같으며 이는 HTTP 요청과 응답을 일반 소켓을 사용하는 것처럼, ØMQ 소켓을 통하여 더 빠르고 좋게 동일한 작업을 수행할 수 있어야 합니다.

;The answer used to be "this is not how it works". ØMQ is not a neutral carrier: it imposes a framing on the transport protocols it uses. This framing is not compatible with existing protocols, which tend to use their own framing. For example, compare an HTTP request and a ØMQ request, both over TCP/IP.

예전에는 ØMQ는 "그렇게 동작하지 않는다"라며 ØMQ는 중립적인 전송수단(Neutral carrier)이 아니다고 했습니다 : 이것은 전송방식 통신규약(OSI 7 계층)에 프레이임을 부가하기 때문입니다. ØMQ의 프레이임은 기존의 통신규약들과 호환되지 않습니다. 예를 들어 HTTP 요청과 ØMQ 요청을 비교하면 둘 다 TCP/IP상에서 동작하지만 자신의 프레이임을 사용하는 경향이 있습니다.

> [옮긴이] 프레이임(framing)은 OSI 7 계층에서 데이터링크에서 정의하고 있으며 메시지 데이터 구성 방식으로 다양한 통신규약들에 의해 교환되고 운반되는 데이터 단위입니다

그림 10 - 네트워크상 HTTP

![네트워크상 HTTP](images/fig10.svg)

;The HTTP request uses CR-LF as its simplest framing delimiter, whereas ØMQ uses a length-specified frame. So you could write an HTTP-like protocol using ØMQ, using for example the request-reply socket pattern. But it would not be HTTP.

HTTP 요청은 CR(0x0D)-LF(0x0A)을 프레이임 구분자로 사용하지만 ØMQ는 길이가 지정된 프레임을 사용합니다. 그래서 ØMQ를 사용하여 요청-응답 소켓 패턴으로 HTTP와 유사한 통신규약을 작성할 수 있지만 HTTP에서는 할 수 없습니다.

그림 11 - 네트워크상 ØMQ

![네트워크상 ØMQ](images/fig11.svg)

;Since v3.3, however, ØMQ has a socket option called ZMQ_ROUTER_RAW that lets you read and write data without the ØMQ framing. You could use this to read and write proper HTTP requests and responses. Hardeep Singh contributed this change so that he could connect to Telnet servers from his ØMQ application. At time of writing this is still somewhat experimental, but it shows how ØMQ keeps evolving to solve new problems. Maybe the next patch will be yours.


v3.3 버전 이후로 ØMQ는 "ZMQ_ROUTER_RAW"라는 새로운 소켓 옵션을 가지고 ØMQ 프레이밍이 없이도 데이터를 읽고 쓸 수 있게 되었습니다. 즉 적절한 HTTP 요청과 응답을 읽고 쓸수 있게 되었다. "하딥 싱(Hardeep Singh)"은 자신의 ØMQ 응용프로그램에서 텔넷 서버에 연결할 수 있도록 이러한 변경에 기여했습니다. 글을 쓰는 시점에는 이것은 다소 실험적이지만 ØMQ가 어떻게 새로운 문제를 해결하기 위해 계속 발전하고 있는지 보여줍니다. 아마도 다음 패치에서는 사용 가능할 것으로 예상됩니다.

> [옮긴이] 현재(2020년 8월) ØMQ 4.3.2의 `zmq_setsockopt()`에서도 `ZMQ_ROUTER_RAW` 사용 가능합니다.

### I/O 스레드들(I/O Threads)
;We said that ØMQ does I/O in a background thread. One I/O thread (for all sockets) is sufficient for all but the most extreme applications. When you create a new context, it starts with one I/O thread. The general rule of thumb is to allow one I/O thread per gigabyte of data in or out per second. To raise the number of I/O threads, use the zmq_ctx_set() call before creating any sockets:

이전에 ØMQ는 백그라운드 스레드를 통하여 I/O을 수행한다고 하였으며, 하나의 I/O 스레드(모든 소켓들에 대하여)에서 메시지 처리가 가능합니다(일부 극단적으로 대량의 I/O를 처리하는 응용프로그램의 경우 I/O 스레드 개수를 조정 가능함). 새로운 컨텍스트를 만들 때, 하나의 I/O 스레드가 시작되며, 일반적으로 하나의 I/O 스레드는 초당 1 기가바이트의 데이트를 입/출력할 수 있습니다. I/O 스레드의 개수를 늘리기 위해서는 소켓을 생성하기 전에 `zmq_ctx_set()` 함수를 사용할 수 있습니다.

```cpp
int io_threads = 4;
void *context = zmq_ctx_new ();
zmq_ctx_set (context, ZMQ_IO_THREADS, io_threads);
assert (zmq_ctx_get (context, ZMQ_IO_THREADS) == io_threads);
```

;We've seen that one socket can handle dozens, even thousands of connections at once. This has a fundamental impact on how you write applications. A traditional networked application has one process or one thread per remote connection, and that process or thread handles one socket. ØMQ lets you collapse this entire structure into a single process and then break it up as necessary for scaling.

하나의 소켓에서 한 번에 수십 개 혹은 수천 개의 연결을 처리할 수 있으며 이것이 ØMQ 기반 응용프로그램 작성에 근본적인 영향을 줍니다. 기존의 네트워크 응용프로그램에서는 원격 연결당 하나의 프로세스 혹은 하나의 스레드가 필요하였으며, 해당 프로세스 혹은 스레드에서 하나의 소켓으로 처리하였습니다. ØMQ를 사용하면 전체 구조를 단일 프로세스로 축소하고 필요에 따라 확장할 수 있습니다.

> [옮긴이] ØMQ 응용프로그램 작성 시 클라이언트/서버를 각각의 스레드로 작성하여 inproc를 통하여 테스트를 수행하고, 정상적인 경우 각 스레드는 프로세스로 분리하여 IPC나 TCP로 연결할 수 있습니다.

;If you are using ØMQ for inter-thread communications only (i.e., a multithreaded application that does no external socket I/O) you can set the I/O threads to zero. It's not a significant optimization though, more of a curiosity.

ØMQ를 스레드 간 통신(inproc)에 사용할 경우(예를 들어 멀티스레드 응용프로그램에서 외부 소켓 I/O가 없을 경우)에 한하여 I/O 스레드의 개수를 0으로 설정할 수 있습니다. 흥미롭지만 중요한 최적화는 아닙니다.

## 메시징 패턴(Messaging Patterns)
;Underneath the brown paper wrapping of ØMQ's socket API lies the world of messaging patterns. If you have a background in enterprise messaging, or know UDP well, these will be vaguely familiar. But to most ØMQ newcomers, they are a surprise. We're so used to the TCP paradigm where a socket maps one-to-one to another node.

ØMQ 소켓 API를 감싸고 있는 갈색 포장지를 걷어내면 메시징 패턴의 세계가 놓여 있습니다. 만약 기업 메시징 시스템에 대한 배경이 있거나 UDP을 아신다면 다소 막연하지만 익숙할 것입니다. 그러나 ØMQ를 새로 접하시는 분에게는 놀라울 것입니다. 우리는 TCP 패러디엄을 사용하여 다른 노드와 1:1 매핑을 하였습니다.

;Let's recap briefly what ØMQ does for you. It delivers blobs of data (messages) to nodes, quickly and efficiently. You can map nodes to threads, processes, or nodes. ØMQ gives your applications a single socket API to work with, no matter what the actual transport (like in-process, inter-process, TCP, or multicast). It automatically reconnects to peers as they come and go. It queues messages at both sender and receiver, as needed. It manages these queues carefully to ensure processes don't run out of memory, overflowing to disk when appropriate. It handles socket errors. It does all I/O in background threads. It uses lock-free techniques for talking between nodes, so there are never locks, waits, semaphores, or deadlocks.

ØMQ가 하는 일을 간단히 다음과 같습니다. 
* 데이터의 블록(메시지, 대용량 데이터(blobs))을 노드 간에 빠르고 효율적으로 전달합니다.
* 노드를 스레드(inproc), 프로세스(ipc) 혹은 머신(tcp, pgm)으로 메핑 할 수 있습니다.
* ØMQ는 응용프로그램에서 전송방식(in-process, inter-process, TCP, multicast)에 관계없이 단일 소켓 API를 제공합니다.
* 노드들 간 연결이 끊기면 자동으로 재연결하게 합니다.
* 송/수신 측에서 메시지들을 대기열에 관리하며, 이러한 대기열을 신중하게 관리하여 프로세스에 메모리가 부족하지 않도록 하며, 필요하다면 디스크에 저장합니다.
* 소켓 오류를 처리합니다.
* 모든 I/O 처리를 백그라운드 스레드가 처리합니다(기본 1개이며, 수량 조정 가능).
* ØMQ는 노드 간의 통신에 잠금 없는 기술을 사용하여 잠금, 대기, 세마포어, 교착이 발생하지 않습니다.

> [옮긴이] 대기열에 설정한 HMW(최고수위 표시)를 초과할 경우 디스크에 저장하기 위해서는 `zmq_setsockopt()`에서 `ZMQ_SWAP`설정을 통해 수행합니다.

;But cutting through that, it routes and queues messages according to precise recipes called patterns. It is these patterns that provide ØMQ's intelligence. They encapsulate our hard-earned experience of the best ways to distribute data and work. ØMQ's patterns are hard-coded but future versions may allow user-definable patterns.

패턴이란 불리는 정확한 레시피에 따라 메시지들을 경유하고 대기열에 쌓을 수 있으며, 이러한 패턴들을 통하여 ØMQ에 지능을 제공합니다.
패턴들은 우리가 힘겹게 얻은 경험을 통하여 데이터와 작업을 분산하기 위한 최선의 방법을 제공합니다. ØMQ의 패턴은 하드 코딩되어 있지만 향후 버전에서는 사용자 정의 가능 패턴이 허용될 수 있습니다.

;ØMQ patterns are implemented by pairs of sockets with matching types. In other words, to understand ØMQ patterns you need to understand socket types and how they work together. Mostly, this just takes study; there is little that is obvious at this level.

ØMQ 패턴들은 소켓과 일치하는 유형의 쌍으로 구현됩니다. ØMQ 패턴을 이해하기 위해서 소켓 유형의 이해가 필요하며 어떻게 함께 동작(ØMQ 패턴, 소켓 유형)하는지 알아야 합니다.

;The built-in core ØMQ patterns are:

내장된 핵심 ØMQ 패턴들은 다음과 같습니다.

;* Request-reply, which connects a set of clients to a set of services. This is a remote procedure call and task distribution pattern.
;* Pub-sub, which connects a set of publishers to a set of subscribers. This is a data distribution pattern.
;* Pipeline, which connects nodes in a fan-out/fan-in pattern that can have multiple steps and loops. This is a parallel task distribution and collection pattern.
;* Exclusive pair, which connects two sockets exclusively. This is a pattern for connecting two threads in a process, not to be confused with "normal" pairs of sockets.

* 요청-응답(REQ-REP)
 - 일련의 클라이언트들에게 서비스들 연결을 하게 하며, 이것은 RPC(remote procedure call)와 작업 분배 패턴입니다.
* 발행-구독(PUSH-PULL)
 - 일련의 발행자들과 구독자들 연결을 하게 하며, 데이터 분배 패턴입니다.
* 파이프라인(Pipeline)
 - 노드들을 팬아웃/팬인(fan-out/fan-in) 패턴으로 연결하게 하며 여러 개의 절차들과 루프를 가질 수 있습니다. 이것은 병렬 작업 분배와 수집 패턴입니다.
* 독점적인 쌍(PAIR-PAIR)
 - 2개의 소켓들을 상호 독점적으로 연결합니다. 이것은 하나의 프로세스에서 2개의 스레드 간에 연결하는 패턴이며 정상적인 소켓의 쌍과 혼동이 없어야 합니다.

;We looked at the first three of these in Chapter 1 - Basics, and we'll see the exclusive pair pattern later in this chapter. The zmq_socket() man page is fairly clear about the patterns — it's worth reading several times until it starts to make sense. These are the socket combinations that are valid for a connect-bind pair (either side can bind):

1장 기본에서 3개의 패턴을 보았으며 "독점적인 쌍"은 다음 장에서 보도록 하겠습니다. `zmq_socket()` 매뉴얼에 패턴에 대한 설명이 되어 있으며 이해될 때까지 숙지할 필요가 있습니다.
다음은 연결-바인드 쌍에 대하여 유효한 소켓 조합입니다(어느쪽이든 바인딩을 수행할 수 있습니다.).

; * PUB and SUB
; * REQ and REP
; * REQ and ROUTER
; * DEALER and REP
; * DEALER and ROUTER
; * DEALER and DEALER
; * ROUTER and ROUTER
; * PUSH and PULL
; * PAIR and PAIR

* PUB와 SUB
* REQ와 REP
* REQ와 ROUTER(주의 필요, REQ는 추가로 1개의 널(\0) 프레임을 넣는다.)
* DEALER와 REP(주의 필요, REP는 1개의 널(\0) 프레임으로 가정한다.)
* DEALER와 ROUTER
* DEALER와 DEALER
* ROUTER와 ROUTER
* PUSH와 PULL
* PAIR와 PAIR

;You'll also see references to XPUB and XSUB sockets, which we'll come to later (they're like raw versions of PUB and SUB). Any other combination will produce undocumented and unreliable results, and future versions of ØMQ will probably return errors if you try them. You can and will, of course, bridge other socket types via code, i.e., read from one socket type and write to another.

XPUB와 XSUB 소켓도 있으며, 다음장에서 다루도록 하겠습니다(PUB와 SUB의 원시(raw) 버전과 유사). 다른 소켓 조합들은 신뢰할 수 없는 결과를 낳을 수 있으며, 미래의 ØMQ에서는 오류를 생성하게 할 예정입니다. 물론 코드상에서 다른 소켓 유형을 연결할 수 있습니다(즉, 한 소켓 유형에서 읽고 다른 소켓 유형에 쓰기).

### 고수준의 메시지 패턴들(High Level Messaging Patterns)
;These four core patterns are cooked into ØMQ. They are part of the ØMQ API, implemented in the core C++ library, and are guaranteed to be available in all fine retail stores.

ØMQ 에는 내장된 4개의 핵심 패턴이 있으며, 그들은 ØMQ API의 일부이며 C++ 라이브러리로 구현되어 있으며 모든 용도에 적절하게 동작함을 보장합니다.

> [옮긴이] 핵심 패턴들은 ØMQ RFC사이트(https://rfc.zeromq.org)에서 정의되어 있습니다.
- MDP(Majordome Protocol), TSP(Titanic Service Protocol), FLP(Freelance Protocol), CHP(Clustered Hashmap Protocol) 등이 있습니다.

;On top of those, we add high-level messaging patterns. We build these high-level patterns on top of ØMQ and implement them in whatever language we're using for our application. They are not part of the core library, do not come with the ØMQ package, and exist in their own space as part of the ØMQ community. For example the Majordomo pattern, which we explore in Chapter 4 - Reliable Request-Reply Patterns, sits in the GitHub Majordomo project in the ØMQ organization.

위의 제목에서 ØMQ의 핵심 라이브러리가 아니고 ØMQ 패키지에 포함되지도 않았음에도 고수준의 메시지 패턴들을 넣은 것은,  ØMQ 커뮤니티에서 자신만의 영역으로 존재하기 때문이다.
예를 들면 집사(Majordome) 패턴(4장-신뢰성 있는 요청-응답 패턴에서 소개)은 ØMQ 조직에서 깃허브 프로젝트(https://github.com/ØMQ/majordomo)로 자리 잡았습니다.

;One of the things we aim to provide you with in this book are a set of such high-level patterns, both small (how to handle messages sanely) and large (how to make a reliable pub-sub architecture).

이 책에서 우리가 제공하고자 하는 일련의 고수준 패턴들은 작을 수도 있고(정상적으로 메시지 처리하는 방법) 거대(신뢰할 수 있는 PUB-SUB 아키텍처를 만드는 방법)할 수도 있습니다.

### 메시지와 작업하기(Working with Messages)
;The libzmq core library has in fact two APIs to send and receive messages. The zmq_send() and zmq_recv() methods that we've already seen and used are simple one-liners. We will use these often, but zmq_recv() is bad at dealing with arbitrary message sizes: it truncates messages to whatever buffer size you provide. So there's a second API that works with zmq_msg_t structures, with a richer but more difficult API:

libzmq 핵심 라이브러리는 사실 메시지를 송/수신하는 2개의 API를 가지고 있습니다. `zmq_send()`와 `zmq_recv()` 함수이며, 이미 한줄짜리 코드로 보았습니다. `zmq_recv()`는 임의의 메시지 크기를 가지는 데이터를 처리하는 데는 좋지 않습니다. `zmq_recv()`는 응용프로그램의 코드 내에서 정의한 버퍼의 크기를 넘을 경우 크기 이상 메시지 데이터는 버리게 됩니다. 그래서 `zmq_msg_t` 구조체를 다룰 수 있도록 추가적인 API로 작업해야 합니다.

;* Initialise a message: zmq_msg_init(), zmq_msg_init_size(), zmq_msg_init_data().
;* Sending and receiving a message: zmq_msg_send(), zmq_msg_recv().
;* Release a message: zmq_msg_close().
;* Access message content: zmq_msg_data(), zmq_msg_size(), zmq_msg_more().
;* Work with message properties: zmq_msg_get(), zmq_msg_set().
;* Message manipulation: zmq_msg_copy(), zmq_msg_move().

* 메시지 초기화 : `zmq_msg_init()`,  `zmq_msg_init_size()`,  `zmq_msg_init_data() `
* 메시지 송/수신 : `zmq_msg_send()`,  `zmq_msg_recv()` 
* 메시지 해제 : `zmq_msg_close()`
* 메시지 내용 접근 : `zmq_msg_data()`,  `zmq_msg_size()`,  `zmq_msg_more()`
* 메시지 속성 작업 : `zmq_msg_get()` ,  `zmq_msg_set()`
* 메시지 조작 : `zmq_msg_copy()` ,  `zmq_msg_move()`

;On the wire, ØMQ messages are blobs of any size from zero upwards that fit in memory. You do your own serialization using protocol buffers, msgpack, JSON, or whatever else your applications need to speak. It's wise to choose a data representation that is portable, but you can make your own decisions about trade-offs.

네트워크상에서, ØMQ 메시지들은 메모리상 0에서 임의의 크기의 덩어리(blob)입니다. 응용프로그램에서 해당 메시지를 처리하기 위해서는 통신규약 버퍼, msgpack, JSON 등으로 직렬화를 수행해야 합니다.
이식 가능한 데이터 표현을 현명하게 선택하기 위하여 장/단점을 이해해야 결정할 수 있습니다.

;In memory, ØMQ messages are zmq_msg_t structures (or classes depending on your language). Here are the basic ground rules for using ØMQ messages in C:

메모리상에서 ØMQ 메시지는 `zmq_msg_t` 구조체(개발 언어에 따라 클래스)로 존재하며 C 언어에서 ØMQ 메시지들을 사용하는 기본 규칙이 있습니다.

; * You create and pass around zmq_msg_t objects, not blocks of data.
; * To read a message, you use zmq_msg_init() to create an empty message, and then you pass that to zmq_msg_recv().
; * To write a message from new data, you use zmq_msg_init_size() to create a message and at the same time allocate a block of data of some size. You then fill that data using memcpy, and pass the message to zmq_msg_send().
; * To release (not destroy) a message, you call zmq_msg_close(). This drops a reference, and eventually ØMQ will destroy the message.
; * To access the message content, you use zmq_msg_data(). To know how much data the message contains, use zmq_msg_size().
; * Do not use zmq_msg_move(), zmq_msg_copy(), or zmq_msg_init_data() unless you read the man pages and know precisely why you need these.
; * After you pass a message to zmq_msg_send(), ØMQ will clear the message, i.e., set the size to zero. You cannot send the same message twice, and you cannot access the message data after sending it.
; * These rules don't apply if you use zmq_send() and zmq_recv(), to which you pass byte arrays, not message structures.

* `zmq_msg_t` 객체(데이터의 블록 아님)를 생성하고 전달할 수 있습니다.
* 메시지를 읽기 위해서 `zmq_msg_init()`을 호출하여 하나의 빈 메시지를 만들고 `zmq_msg_recv()`을 통하여 수신합니다.
* 새로운 데이터(구조체)에 메시지를 쓰기 위해서 `zmq_msg_init_size()`을 호출하여 메시지를 생성하고 동시에 구조체 크기의 데이터 블록을 할당합니다. 그리고 `memcpy()`을 사용하여 데이터를 채워 넣고 `zmq_msg_send()`를 통하여 메시지를 송신합니다.
* 메시지를 해제(파괴가 아님)를 위하여, `zmq_msg_close()`을 호출하여 참조를 없애고 결국 ØMQ가 메시지를 삭제하게 합니다.
* 메시지 내용을 접근하기 위하여 `zmq_msg_data()`을 사용하며, 메시지 내용에 얼마나 많은 데이터가 있는지 알기 위하여 `zmq_msg_size()`을 호출합니다.
* `zmq_msg_move()`, `zmq_msg_copy()`, `zmq_msg_init_data()`에 대하여 정확하게 사용법을 확인하여 사용해야 합니다.
* 메시지가 `zmq_msg_send()`을 통해 전송된 이후, ØMQ는 메시지를 청소합니다. 예를 들어 크기를 0으로 설정하여 동일 메시지를 두 번 보낼 수 없으며 데이터 전송 이후 메시지에 접근할 수 없습니다.
* 이러한 규칙은 `zmq_send()`와 `zmq_recv()`을 사용한 경우 적용되지 않으며, 이 함수들은 바이트의 배열을 전달하지 메시지 구조체를 전달하는 것은 아니기 때문입니다.

;If you want to send the same message more than once, and it's sizable, create a second message, initialize it using zmq_msg_init(), and then use zmq_msg_copy() to create a copy of the first message. This does not copy the data but copies a reference. You can then send the message twice (or more, if you create more copies) and the message will only be finally destroyed when the last copy is sent or closed.

만약 동일 메시지를 한번 이상 보내고 싶을 경우, 추가적인 메시지를 만들어 `zmq_msg_init()`으로 초기화하고 `zmq_msg_copy()`을 사용하고 첫 번째 메시지를 복사하여 생성할 수 있습니다. 이것은 데이터를 복사하는 것이 아니라 참조를 복사합니다. 그런 다음 메시지를 두 번(혹은 이상으로 더 많은 복사를 작성하는 경우) 보내며, 마지막 사본을 보내거나 닫을 때 메시지가 삭제됩니다.

;ØMQ also supports multipart messages, which let you send or receive a list of frames as a single on-the-wire message. This is widely used in real applications and we'll look at that later in this chapter and in Chapter 3 - Advanced Request-Reply Patterns.

ØMQ는 멀티파트 메시지들을 지원하며 일련의 데이터 프레임들을 하나의 네트워크상 메시지로 송/수신하게 하며 실제 응용프로그램에서 광범위하게 사용됩니다. "3장-개선된 요청-응답 패턴"에서 살펴보도록 하겠습니다.

;Frames (also called "message parts" in the ØMQ reference manual pages) are the basic wire format for ØMQ messages. A frame is a length-specified block of data. The length can be zero upwards. If you've done any TCP programming you'll appreciate why frames are a useful answer to the question "how much data am I supposed to read of this network socket now?"

프레임들(ØMQ 참조 매뉴얼에서는 메시지 파트들이고도 함)은 기본적인 ØMQ 메시지 형태이며 하나의 프레임은 일정 길이의 데이터의 블록이며 길이는 0 이상입니다. TCP 프로그래밍을 하셨다면 "네트워크 소켓에서 얼마나 많은 데이터를 읽어야 하는지?"란 물음에 대하여 ØMQ 프레임이 유용한 해답임을 알 수 있습니다.

;There is a wire-level protocol called ZMTP that defines how ØMQ reads and writes frames on a TCP connection. If you're interested in how this works, the spec is quite short.

[ZMTP](https://rfc.ØMQ.org/spec/37/)라고 불리는 와이어 레벨(wire-level) 통신규약이 존재하며 ØMQ가 TCP 연결상에서 프레임들을 어떻게 읽고 쓰는지를 정의합니다. 관심 있으면 한번 참조하기 바라며 사양서는 단순합니다.

;Originally, a ØMQ message was one frame, like UDP. We later extended this with multipart messages, which are quite simply series of frames with a "more" bit set to one, followed by one with that bit set to zero. The ØMQ API then lets you write messages with a "more" flag and when you read messages, it lets you check if there's "more".

원래는 하나의 ØMQ 메시지는 UDP처럼 하나의 프레임이었지만 나중에 멀티파트 메시지들로 확장하였으며, 단순히 일련의 프레임들이 있으며 "more" 비트을 1로 설정하고, 하나의 프레임 있으면 "more" 비트를 0으로 설정합니다.
ØMQ API에서 "more" 플레그를 통해 메시지를 쓸 수 있으며, 메시지를 읽을 때 "more"을 체크하여 추가적인 동작을 할 수 있습니다.

;In the low-level ØMQ API and the reference manual, therefore, there's some fuzziness about messages versus frames. So here's a useful lexicon:

저수준의 ØMQ API와 참조 매뉴얼에서는 메시지들과 프레임들 간에 모호함이 있어 용어를 정의하겠습니다.

;* A message can be one or more parts.
;* These parts are also called "frames".
;* Each part is a zmq_msg_t object.
;* You send and receive each part separately, in the low-level API.
;* Higher-level APIs provide wrappers to send entire multipart messages.

* 하나의 메시지는 하나 이상의 파트(part)들이 될 수 있습니다.
* 여기서 파트는 프레임입니다.
* 각 파트는 `zmq_msg_t` 객체입니다.
* 저수준 ØMQ API를 통하여 각 파트를 개별적으로 송/수신할 수 있습니다.
* 고수준 ØMQ API에서는 전체 멀티파트 메시지를 전송하는 래퍼를 제공합니다.

;Some other things that are worth knowing about messages:

메시지에 대하여 알아두면 좋을 것들이 있습니다.

; * You may send zero-length messages, e.g., for sending a signal from one thread to another.
; * ØMQ guarantees to deliver all the parts (one or more) for a message, or none of them.
; * ØMQ does not send the message (single or multipart) right away, but at some indeterminate later time. A multipart message must therefore fit in memory.
; * A message (single or multipart) must fit in memory. If you want to send files of arbitrary sizes, you should break them into pieces and send each piece as separate single-part messages. Using multipart data will not reduce memory consumption.
; * You must call zmq_msg_close() when finished with a received message, in languages that don't automatically destroy objects when a scope closes. You don't call this method after sending a message.

* 길이가 0인 메시지를 전송할 수 있습니다. 예를 들어 하나의 스레드에서 다른 스레드로 신호를 전송하기 위한 목적입니다.
* ØMQ는 메시지의 모든 파트들(하나 이상)의 전송을 보증하거나, 하지 않을 수 있습니다.
* ØMQ는 메시지(1개 혹은 멀티파트)를 바로 전송하지 않고, 일정 시간 후에 보냅니다.  그래서 하나의 멀티파트 메시지는 설정한 메모리의 크기에 맞아야 합니다.
* 하나의 메시지(1개 혹은  멀티파트)는 반드시 메모리 크기에 맞아야 하며 파일이나 임의의 크기를 전송하려면 하나의 파트의 메시지로 쪼개어서 각 조각을 전송해야 합니다. 멀티파트 데이터를 사용하여 메모리 사용을 줄일 수는 없습니다.
* 메시지 수신이 끝나면 `zmq_msg_close()`을 반드시 호출하여야 하며, 일부 개발 언어에서는 변수의 사용 범위를 벋어나도 자동으로 삭제하지 않습니다. 메시지를 전송할 때는 `zmq_msg_close()`을 호출하지 마시기 바랍니다.

> [옮긴이]  zhelpers.h에 정의된 s_dump() 함수를 통하여 멀티파트 메시지(multipart message) 출력이 가능하게 합니다.

```cpp
s_dump (void *socket)
{
    int rc;

    zmq_msg_t message;
    rc = zmq_msg_init (&message);
    assert (rc == 0);

    puts ("----------------------------------------");
    //  Process all parts of the message
    do {
        int size = zmq_msg_recv (&message, socket, 0);
        assert (size >= 0);

        //  Dump the message as text or binary
        char *data = (char*)zmq_msg_data (&message);
        assert (data != 0);
        int is_text = 1;
        int char_nbr;
        for (char_nbr = 0; char_nbr < size; char_nbr++) {
            if ((unsigned char) data [char_nbr] < 32
                || (unsigned char) data [char_nbr] > 126) {
                is_text = 0;
            }
        }

        printf ("[%03d] ", size);
        for (char_nbr = 0; char_nbr < size; char_nbr++) {
            if (is_text) {
                printf ("%c", data [char_nbr]);
            } else {
                printf ("%02X", (unsigned char) data [char_nbr]);
            }
        }
        printf ("\n");
    } while (zmq_msg_more (&message));

    rc = zmq_msg_close (&message);
    assert (rc == 0);
}
```

;And to be repetitive, do not use zmq_msg_init_data() yet. This is a zero-copy method and is guaranteed to create trouble for you. There are far more important things to learn about ØMQ before you start to worry about shaving off microseconds.

아직 `zmq_msg_init_data()`을 언급하지 않았지만 이것은 "zero-copy"며 문제를 일으킬 수 있습니다. 마이크로초 단축에 대해 걱정하기보다는 ØMQ에 대해 알아야 할 중요한 사항들이 많이 있습니다.

;This rich API can be tiresome to work with. The methods are optimized for performance, not simplicity. If you start using these you will almost definitely get them wrong until you've read the man pages with some care. So one of the main jobs of a good language binding is to wrap this API up in classes that are easier to use.

많은 API는 작업하기 피곤하며 각종 함수들을 성능을 위해 최적화하는 것도 단순하지 않습니다. [ØMQ 매뉴얼](http://api.ØMQ.org/)에 익숙하기 전에 API를 잘못 사용하는 경우 오류가 발생할 수 있습니다. 특정 개발 언어에 대한 바인딩을 통하여 API를 쉽게 사용하는 것도 하나의 방법입니다.

### 다중 소켓 다루기(Handling Multiple Sockets)
;In all the examples so far, the main loop of most examples has been:

지금까지의 예제들은 아래 항목의 루프를 수행하였습니다.

;1. Wait for message on socket.
;2. Process message.
;3. Repeat.

1. 소켓의 메시지를 기다림
2. 메시지를 처리
3. 반복

;What if we want to read from multiple endpoints at the same time? The simplest way is to connect one socket to all the endpoints and get ØMQ to do the fan-in for us. This is legal if the remote endpoints are in the same pattern, but it would be wrong to connect a PULL socket to a PUB endpoint.

만약 다중 단말로부터 동시에 메시지를 받아 처리해야 한다면 하나의 소켓에 모든 단말들을 연결하여 ØMQ가 팬아웃 형태로 처리하는 것이 간단한 방법이며, 원격 단말들에도 동일한 패턴을 적용할 수 있습니다. 하지만 패턴이 다른 경우 즉 PULL 소켓을 PUB 단말에 연결하는 것은 잘못된 방법이다.

> [옮긴이] 가능한 패턴 : PUB/SUB, REQ/REP, REQ/ROUTER, DEALER/REP, DEALER/ROUTER, DEALER/DEALER, ROUTER/ROUTER, PUSH/PULL, PAIR/PAIR

;To actually read from multiple sockets all at once, use zmq_poll(). An even better way might be to wrap zmq_poll() in a framework that turns it into a nice event-driven reactor, but it's significantly more work than we want to cover here.

실제로 한 번에 여러 소켓에서 읽으려면 `zmq_poll()`을 사용해야 합니다. 더 좋은 방법은 `zmq_poll()`을 프레임워크로 감싸서 이벤트 중심으로 반응하도록 변형하는 것이지만, 이것은 여기서 다루고 싶은 것보다 훨씬 더 많은 작업이 필요합니다.

> [옮긴이] zloop 리엑터(reactor)를 통하여 이벤트 중심으로 반응하도록 zmq_poll() 대체 가능합니다.

;Let's start with a dirty hack, partly for the fun of not doing it right, but mainly because it lets me show you how to do nonblocking socket reads. Here is a simple example of reading from two sockets using nonblocking reads. This rather confused program acts both as a subscriber to weather updates, and a worker for parallel tasks:

그럼 그렇게 하지 말라는 예로, 비차단(nonblocking) 소켓 읽기를 수행하는 방법을 보여 드리겠습니다.
다음 예제는  2개의 소켓으로 메시지를 비차단 읽기로 수신하고 있으며 프로그램은 날씨 변경정보의 구독자와 병렬 작업의 작업자 역할을 수행하여 다소 혼란스럽습니다.

msreader.c: 다중 소켓 읽기

```cpp
//  Reading from multiple sockets
//  This version uses a simple recv loop

#include "zhelpers.h"

int main (void) 
{
    //  Connect to task ventilator
    void *context = zmq_ctx_new ();
    void *receiver = zmq_socket (context, ZMQ_PULL);
    zmq_connect (receiver, "tcp://localhost:5557");

    //  Connect to weather server
    void *subscriber = zmq_socket (context, ZMQ_SUB);
    zmq_connect (subscriber, "tcp://localhost:5556");
    zmq_setsockopt (subscriber, ZMQ_SUBSCRIBE, "10001 ", 5);

    //  Process messages from both sockets
    //  We prioritize traffic from the task ventilator
    while (1) {
        char msg [256];
        while (1) {
            int size = zmq_recv (receiver, msg, 255, ZMQ_DONTWAIT);
            if (size != -1) {
                //  Process task
                printf("P");
            }
            else
                break;
        }
        while (1) {
            int size = zmq_recv (subscriber, msg, 255, ZMQ_DONTWAIT);
            if (size != -1) {
                //  Process weather update
                printf("S");
            }
            else
                break;
        }
        //  No activity, so sleep for 1 msec
        s_sleep (1);
    }
    zmq_close (receiver);
    zmq_close (subscriber);
    zmq_ctx_destroy (context);
    return 0;
}
```

;The cost of this approach is some additional latency on the first message (the sleep at the end of the loop, when there are no waiting messages to process). This would be a problem in applications where submillisecond latency was vital. Also, you need to check the documentation for nanosleep() or whatever function you use to make sure it does not busy-loop.

이런 접근에 대한 비용은 첫 번째 메시지에 대한 추가적인 지연이 발생한다(메시지 처리하기에도 바쁜데 루푸의 마지막에 `sleep()`). 이러한 접근은 고속 처리가 필요한 응용프로그램에서 치명적인 지연을 발생합니다. `nanosleep()`를 사용할 경우 바쁘게 반복(busy-loop)되지 않는지 확인해야 합니다.

> [옮긴이] msreader.c에서 사용된 s_sleep()는 zhelpers.h에 정의되어 있음

```cpp
//  Sleep for a number of milliseconds
static void
s_sleep (int msecs)
{
#if (defined (WIN32))
    Sleep (msecs);
#else
    struct timespec t;
    t.tv_sec  =  msecs / 1000;
    t.tv_nsec = (msecs % 1000) * 1000000;
    nanosleep (&t, NULL);
#endif
}
```

;You can treat the sockets fairly by reading first from one, then the second rather than prioritizing them as we did in this example.

예제에서 2개의 루프에서 첫 번쨰 소켓(ZMQ_PULL)이 두 번쨰 소켓(ZMQ_SUB)보다 먼저 처리하게 하였다.

;Now let's see the same senseless little application done right, using zmq_poll():

대안으로 이제 `zmq_poll()`을 사용하는 응용프로그램을 보도록 하자.

mspoller.c: 다중 소켓 폴러

```cpp
//  Reading from multiple sockets
//  This version uses zmq_poll()

#include "zhelpers.h"

int main (void) 
{
    //  Connect to task ventilator
    void *context = zmq_ctx_new ();
    void *receiver = zmq_socket (context, ZMQ_PULL);
    zmq_connect (receiver, "tcp://localhost:5557");

    //  Connect to weather server
    void *subscriber = zmq_socket (context, ZMQ_SUB);
    zmq_connect (subscriber, "tcp://localhost:5556");
    zmq_setsockopt (subscriber, ZMQ_SUBSCRIBE, "10000 ", 5);

    //  Process messages from both sockets
    while (1) {
        char msg [256];
        zmq_pollitem_t items [] = {
            { receiver,   0, ZMQ_POLLIN, 0 },
            { subscriber, 0, ZMQ_POLLIN, 0 }
        };
        zmq_poll (items, 2, -1);
        if (items [0].revents & ZMQ_POLLIN) {
            int size = zmq_recv (receiver, msg, 255, 0);
            if (size != -1) {
                //  Process task
                printf("P");
            }
        }
        if (items [1].revents & ZMQ_POLLIN) {
            int size = zmq_recv (subscriber, msg, 255, 0);
            if (size != -1) {
                //  Process weather update
                printf("S");
            }
        }
    }
    zmq_close (subscriber);
    zmq_ctx_destroy (context);
    return 0;
}
```

;The items structure has these four members:

`zmq_pollitem_t`는 4개의 맴버가 있으며 구조체이다(ØMQ 4.3.2).

```cpp
typedef struct zmq_pollitem_t
{
    void *socket;
#if defined _WIN32
    SOCKET fd;
#else
    int fd;
#endif
    short events;
    short revents;
} zmq_pollitem_t;
```

> [옮긴이] msreader와 mspoller 테스트를 위해서는 taskvent(호흡기 서버)와 wuserver(기상 정보 변경 서버) 수행이 필요함

taskvent.c : 호흡기 서버 프로그램

```cpp
//  Task ventilator
//  Binds PUSH socket to tcp://localhost:5557
//  Sends batch of tasks to workers via that socket

#include "zhelpers.h"

int main (void) 
{
    void *context = zmq_ctx_new ();

    //  Socket to send messages on
    void *sender = zmq_socket (context, ZMQ_PUSH);
    zmq_bind (sender, "tcp://*:5557");

    //  Socket to send start of batch message on
    void *sink = zmq_socket (context, ZMQ_PUSH);
    zmq_connect (sink, "tcp://localhost:5558");

    printf ("Press Enter when the workers are ready: ");
    getchar ();
    printf ("Sending tasks to workers...\n");

    //  The first message is "0" and signals start of batch
    s_send (sink, "0");

    //  Initialize random number generator
    srandom ((unsigned) time (NULL));

    //  Send 100 tasks
    int task_nbr;
    int total_msec = 0;     //  Total expected cost in msecs
    for (task_nbr = 0; task_nbr < 100; task_nbr++) {
        int workload;
        //  Random workload from 1 to 100msecs
        workload = randof (100) + 1;
        total_msec += workload;
        char string [10];
        sprintf (string, "%d", workload);
        s_send (sender, string);
    }
    printf ("Total expected cost: %d msec\n", total_msec);

    zmq_close (sink);
    zmq_close (sender);
    zmq_ctx_destroy (context);
    return 0;
}
```
wuserver.c : 기상 정보 변경 서버
```cpp
//  Weather update server
//  Binds PUB socket to tcp://*:5556
//  Publishes random weather updates

#include "zhelpers.h"

int main (void)
{
    //  Prepare our context and publisher
    void *context = zmq_ctx_new ();
    void *publisher = zmq_socket (context, ZMQ_PUB);
    int rc = zmq_bind (publisher, "tcp://*:5556");
    assert (rc == 0);

    //  Initialize random number generator
    srandom ((unsigned) time (NULL));
    while (1) {
        //  Get values that will fool the boss
        int zipcode, temperature, relhumidity;
        zipcode     = randof (99999);
        temperature = randof (215) - 80;
        relhumidity = randof (50) + 10;

        //  Send message to all subscribers
        char update [20];
        sprintf (update, "%05d %d %d", zipcode, temperature, relhumidity);
        s_send (publisher, update);
    }
    zmq_close (publisher);
    zmq_ctx_destroy (context);
    return 0;
}
```

> [옮긴이] 빌드 및 테스트

~~~ {.bash}
PS D:\git_store\zguide-kr\examples\C> cl -EHsc mspoller.c libzmq.lib
PS D:\git_store\zguide-kr\examples\C> cl -EHsc msreader.c libzmq.lib
PS D:\git_store\zguide-kr\examples\C> cl -EHsc taskvent.c libzmq.lib
PS D:\git_store\zguide-kr\examples\C> cl -EHsc wuserver.c libzmq.lib

PS D:\git_store\zguide-kr\examples\C> ./taskvent
Press Enter when the workers are ready:
Sending tasks to workers...
Total expected cost: 4856 msec

PS D:\git_store\zguide-kr\examples\C> ./wuserver

PS D:\git_store\zguide-kr\examples\C> ./mspoller
SSSSSSSSSSSSSSSPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPSPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSS

PS D:\git_store\zguide-kr\examples\C> ./msreader
SSSSSSSSSSSSSSSSSSSPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPPSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSS
~~~

### 멀티파트 메시지들(Multipart Messages)
;ØMQ lets us compose a message out of several frames, giving us a "multipart message". Realistic applications use multipart messages heavily, both for wrapping messages with address information and for simple serialization. We'll look at reply envelopes later.

ØMQ는 여러 개의 프레임들로 하나의 메시지를 구성하게 하며 멀티파트 메시지를 제공합니다. 현실에서는 응용프로그램은 멀티파트 메시지를 많이 사용하며 IP 주소 정보를 메시지에 내포하여 직열화 용도로 사용합니다. 우리는 이것을 응답 봉투에서 나중에 다루겠습니다.

;What we'll learn now is simply how to blindly and safely read and write multipart messages in any application (such as a proxy) that needs to forward messages without inspecting them.

이제부터 어떤 응용프로그램(프록시와 같은)에서나 멀티파트 메시지를 맹목적으로 안전하게 읽고 쓰는 방법을 설명하며, 프록시는 메시지를 별도의 검사 없이 전달하도록 합니다.

;When you work with multipart messages, each part is a zmq_msg item. E.g., if you are sending a message with five parts, you must construct, send, and destroy five zmq_msg items. You can do this in advance (and store the zmq_msg items in an array or other structure), or as you send them, one-by-one.

멀티파트 메시지 작업 시, 개별 파트는 `zmq_msg` 항목이며, 메시지를 5개의 파트들로 전송한다면 반드시 5개의 zmq_msg 항목들을 생성, 전송, 파괴를 해야 한다. 이것을 미리 하거나(`zmq_msg`를 배열이나 구조체에 저장하여) 혹은 전송 이후 차례로 할 수 있다.

;Here is how we send the frames in a multipart message (we receive each frame into a message object):

`zmq_msg_send()`를 통하여 멀티파트 메시지에 있는 프레임을 전송하는 방법이다(각각의 프레임은 메시지 객체로 수신합니다).

```cpp
zmq_msg_send (&message, socket, ZMQ_SNDMORE);
…
zmq_msg_send (&message, socket, ZMQ_SNDMORE);
…
zmq_msg_send (&message, socket, 0);
```

;Here is how we receive and process all the parts in a message, be it single part or multipart:

`zmq_msg_recv()`을 통하여 메시지에 있는 모든 파트들을 수신하고 처리하는 예제이다, 파트는 1개이거나 멀티파트로 가능함
```cpp
while (1) {
    zmq_msg_t message;
    zmq_msg_init (&message);
    zmq_msg_recv (&message, socket, 0);
    // Process the message frame
    …
    zmq_msg_close (&message);
    if (!zmq_msg_more (&message))
        break; // Last message frame
}
```

;Some things to know about multipart messages:

멀티파트 메시지들에 대하여 알아야 될 몇 가지 사항은 다음과 같습니다.

; * When you send a multipart message, the first part (and all following parts) are only actually sent on the wire when you send the final part.
; * If you are using zmq_poll(), when you receive the first part of a message, all the rest has also arrived.
; * You will receive all parts of a message, or none at all.
; * Each part of a message is a separate zmq_msg item.
; * You will receive all parts of a message whether or not you check the more property.
; * On sending, ØMQ queues message frames in memory until the last is received, then sends them all.
; * There is no way to cancel a partially sent message, except by closing the socket.

* 멀티파트 메시지를 전송할 때, 첫 번째 파트(와 후속되는 파트들)는 마지막 파트를 전송하기 전에 네트워크상에서 실제 전송되어야 합니다.
* `zmq_poll()`을 사용한다면, 메시지의 첫 번째 파트가 수신될 때, 나머지 모든 파트들이 도착되어야 합니다.
* 메시지의 모든 파트들을 수신하거나, 하나도 받지 않을 수 있습니다.
* 메시지의 각각의 파트는 개별적인 `zmq_msg` 항목입니다.
* `zmq_msg_more()`을 통한 `more` 속성을 점검 여부에 관계없이 메시지의 모든 파트들을 수신할 수 있습니다.
* 데이터 전송 시, ØMQ는 메시지 대기열에 마지막 프레임이 올 때까지 쌓이게 하여, 마지막 프레임이 도착하면 대기열의 모든 프레임들을 전송합니다.
* 소켓을 닫는 경우를 제외하고는 메시지 전송 시 프레임들을 부분적으로 취소할 방법은 없습니다.

> [옮긴이] mbroker.c : zmq_poll()을 통한 송/수신 브로커에서 `zmq_msg_t`을 통해 메시지를 정의하고 멀티파트 메시지를 처리합니다.

```cpp
//  Simple request-reply broker

#include "zhelpers.h"

int main (void) 
{
    //  Prepare our context and sockets
    void *context = zmq_ctx_new ();
    void *frontend = zmq_socket (context, ZMQ_ROUTER);
    void *backend  = zmq_socket (context, ZMQ_DEALER);
    zmq_bind (frontend, "tcp://*:5559");
    zmq_bind (backend,  "tcp://*:5560");

    //  Initialize poll set
    zmq_pollitem_t items [] = {
        { frontend, 0, ZMQ_POLLIN, 0 },
        { backend,  0, ZMQ_POLLIN, 0 }
    };
    //  Switch messages between sockets
    while (1) {
        zmq_msg_t message;
        zmq_poll (items, 2, -1);
        if (items [0].revents & ZMQ_POLLIN) {
            while (1) {
                //  Process all parts of the message
                zmq_msg_init (&message);
                zmq_msg_recv (&message, frontend, 0);
                int more = zmq_msg_more (&message);
                zmq_msg_send (&message, backend, more? ZMQ_SNDMORE: 0);
                zmq_msg_close (&message);
                if (!more)
                    break;      //  Last message part
            }
        }
        if (items [1].revents & ZMQ_POLLIN) {
            while (1) {
                //  Process all parts of the message
                zmq_msg_init (&message);
                zmq_msg_recv (&message, backend, 0);
                int more = zmq_msg_more (&message);
                zmq_msg_send (&message, frontend, more? ZMQ_SNDMORE: 0);
                zmq_msg_close (&message);
                if (!more)
                    break;      //  Last message part
            }
        }
    }
    //  We never get here, but clean up anyhow
    zmq_close (frontend);
    zmq_close (backend);
    zmq_ctx_destroy (context);
    return 0;
}
```

### 중개자와 프록시(Intermediaryies and Proxies)
;ØMQ aims for decentralized intelligence, but that doesn't mean your network is empty space in the middle. It's filled with message-aware infrastructure and quite often, we build that infrastructure with ØMQ. The ØMQ plumbing can range from tiny pipes to full-blown service-oriented brokers. The messaging industry calls this intermediation, meaning that the stuff in the middle deals with either side. In ØMQ, we call these proxies, queues, forwarders, device, or brokers, depending on the context.

ØMQ는 지능의 탈중앙화를 목표로 만들었지만 네트워크상에서 중간이 텅 비었다는 것은 아니며, ØMQ를 통한 메시지 인식 가능한 인프라구조를 만들었습니다.
ØMQ의 배관은 조그만 파이프에서 완전한 서비스 지향 브로커까지 확장될 수 있습니다. 
메시지 업계에서는 이것을 중개자로 부르며 양쪽의 중앙에서 중개하는 역할을 수행하며, ØMQ에서는 문맥에 따라 프록시, 대기열, 포워더, 디바이스, 브로커라 부르겠습니다.

;This pattern is extremely common in the real world and is why our societies and economies are filled with intermediaries who have no other real function than to reduce the complexity and scaling costs of larger networks. Real-world intermediaries are typically called wholesalers, distributors, managers, and so on.

이러한 패턴은 현실세계에서 일반적이지만 왜 우리의 사회와 경제에서 실제 아무 기능도 없는 중개자로 채워져 있는 것은 거대한 네트워크 상에서 복잡성을 줄이고 비용을 낮추기 위함입니다. 현실세계에서 중계자는 보통 도매상, 판매 대리점, 관리자 등으로 불립니다.

### 동적 발견 문제(The Dynamic Discovery Problem)
;One of the problems you will hit as you design larger distributed architectures is discovery. That is, how do pieces know about each other? It's especially difficult if pieces come and go, so we call this the "dynamic discovery problem".

거대한 분산 아키텍처를 설계할 때 머리를 강타하는 문제 중에 하나가 상대방을 발견하는 것입니다. 어떻게 각 노드들이 상호 간을 인식할 수 있을지, 이것은 노드들이 네트워크상에서 들어오고 나가고를 반복하면서 특히 어렵게 되며 이러한 문제를 "동적 발견 문제"로 부르기로 하겠습니다.

> [옮긴이] 클라이언트에서 서비스(서버) 제공자를 찾기 위한 방법이 분산 시스템 구성에서 주요한 화두이며, 일부 상용 제품에서는 디렉터리 서비스(Directory Service)를 통해 문제를 해결합니다. 대표적인 제품으로 Apache LDAP이 있으며, NASA의 GMSEC의 경우 LDAP을 사용합니다.

;There are several solutions to dynamic discovery. The simplest is to entirely avoid it by hard-coding (or configuring) the network architecture so discovery is done by hand. That is, when you add a new piece, you reconfigure the network to know about it.

동적 발견에 대한 몇 가지 해결 방법이 있지만, 가장 쉬운 해결 방법은 네트워크 아키텍처를 하드 코딩(혹은 구성 파일 사용)하여 사전에 정의하는 것이지만, 새로운 노드가 추가되면 네트워크 구성을 다시 하드 코딩(혹은 구성 파일 변경) 해야 합니다.

그림 12 - 소규모 Pub-Sub 네트워크

![Small-Scale Pub-Sub Network](images/fig12.svg)

;In practice, this leads to increasingly fragile and unwieldy architectures. Let's say you have one publisher and a hundred subscribers. You connect each subscriber to the publisher by configuring a publisher endpoint in each subscriber. That's easy. Subscribers are dynamic; the publisher is static. Now say you add more publishers. Suddenly, it's not so easy any more. If you continue to connect each subscriber to each publisher, the cost of avoiding dynamic discovery gets higher and higher.

실제 이런 해결법(하드 코딩(혹은 구성 파일 변경))은 현명하지 못하며 무너지기 쉬운 아키텍터로 이끌어 갑니다.
하나의 발행자에 대하여 100여 개의 구독자가 있다면 각 구독자가 연결하려는 발행자 단말 정보를 구성 정보화하여 발행자에 연결하려 할 것입니다. 발행자가가 고정되어 있고 구독자가 동적인 경우는 쉽지만, 새로운 발행자들을 추가할 경우 더 이상 이전 구성 정보를 사용할 수 없습니다. 만약 구성 정보 변경을 통하여 각 구독자를 발행자에 연결하게 한다면 동적 발견 회피 비용(구성 정보 변경 + 구독자 재기동)은 점점 더 늘어갈 것입니다.

그림 13 - 프록시를 통한 발행-구독 네트워크

![Pub-Sub Network with a Proxy](images/fig14.svg)

;There are quite a few answers to this, but the very simplest answer is to add an intermediary; that is, a static point in the network to which all other nodes connect. In classic messaging, this is the job of the message broker. ØMQ doesn't come with a message broker as such, but it lets us build intermediaries quite easily.

이것에 대한 해결책은 중개자를 추가하는 것이며, 네트워크상 정적 지점에 다른 모든 노드들이 연결하게 합니다. 전통적인 메시징에서 이러한 역할은 메시지 브로커가 수행합니다. ØMQ는 이와 같은 메시지 브로커가 없지만 중개자들을 쉽게 만들 수 있습니다.

;You might wonder, if all networks eventually get large enough to need intermediaries, why don't we simply have a message broker in place for all applications? For beginners, it's a fair compromise. Just always use a star topology, forget about performance, and things will usually work. However, message brokers are greedy things; in their role as central intermediaries, they become too complex, too stateful, and eventually a problem.

그럼 왜 ØMQ에서 메시지 브로커를 만들지 않았는지 궁금해질 것입니다. 초보자가 응용프로그램을 작성할 때 메시지 브로커가 있다면 상당히 편리할 수 있습니다. 마치 네트워크 상에서 성능을 고려하지 않은 스타 토폴로지가 편리한 것처럼 말입니다.
하지만 메시지 브로커는 탐욕스러운 놈입니다 : 특히 중앙 중개자 역할을 할 경우 그들은 너무 복잡하고 다양한 상태를 가지고, 결국 문제를 일으킵니다.

;It's better to think of intermediaries as simple stateless message switches. A good analogy is an HTTP proxy; it's there, but doesn't have any special role. Adding a pub-sub proxy solves the dynamic discovery problem in our example. We set the proxy in the "middle" of the network. The proxy opens an XSUB socket, an XPUB socket, and binds each to well-known IP addresses and ports. Then, all other processes connect to the proxy, instead of to each other. It becomes trivial to add more subscribers or publishers.

중개자를 단순히 상태가 없는 메시지 스위치들로 생각하면 좋을 것 같으며, 유사한 사례는 HTTP 프록시이며 클라이언트의 요청을 서버에 전달하고 서버의 응답을 클라이언트에 전달하는 역할 외에는 수행하지 않습니다.
PUB-SUB 프록시를 추가하여 우리의 예제에서 동적 발견 문제를 해결할 수 있습니다. 프록시를 네트워크상 중앙에 두고 프록시를 발행자에게는 XSUB 소켓으로 구독자에게는 XPUB 소켓으로 오픈하고 각각(XSUB, XPUB)에 대하여 잘 알려진 IP 주소와 포트로 바인딩하게 합니다. 그러면 다른 프로세스들(발행자, 구독자)는 개별적으로 연결하는 것 대신에 프록시에 연결됩니다. 이런 구성에서 추가적인 발행자나 구독자를 구성하는 것은 쉬운 일입니다.

그림 14 - 확장된 발행-구독

![Extended Pub-Sub](images/fig13.svg)

;We need XPUB and XSUB sockets because ØMQ does subscription forwarding from subscribers to publishers. XSUB and XPUB are exactly like SUB and PUB except they expose subscriptions as special messages. The proxy has to forward these subscription messages from subscriber side to publisher side, by reading them from the XSUB socket and writing them to the XPUB socket. This is the main use case for XSUB and XPUB.

ØMQ가 구독 정보를 구독자들로부터 발행자들로 전달할 수 있는 XSUB과 XPUB 소켓에 대하여 살펴보도록 하겠습니다. XSUB와 XPUB은 특별한 메시지들에 대한 구독 정보를 공개하는 것 외에는 SUB와 PUB과 정확히 일치합니다. 프록시는 구독된 메시지들을 발행자들로부터 구독자들로 전달하며, 구독된 정보를 XSUB 소켓으로 읽어서 XPUB 소켓에 쓰도록 합니다. 이것이 XSUB와 XPUB의 주요 사용법입니다.

> [옮긴이] XSUB는 eXtended subscriber, XPUB은 eXtended publisher이며, 다수의 발행자들과 구독자들을 연결하는 프록시(`zmq_proxy()`)를 통해 브로커 역할을 수행합니다.

> [옮긴이] XSUB와 XPUB을 테스트하기 위하여 기상 변경 정보를 전달하는 `wuserver`의 `bind()`를 `connect()`로 변경하여 테스트를 하겠습니다.

wuserver.c : 기상 변경 정보 발행
```cpp
//  Weather update server
//  Binds PUB socket to tcp://*:5556
//  Publishes random weather updates

#include "zhelpers.h"

int main (void)
{
    //  Prepare our context and publisher
    void *context = zmq_ctx_new ();
    void *publisher = zmq_socket (context, ZMQ_PUB);
    int rc = zmq_connect (publisher, "tcp://localhost:5556");
    assert (rc == 0);

    //  Initialize random number generator
    srandom ((unsigned) time (NULL));
    while (1) {
        //  Get values that will fool the boss
        int zipcode, temperature, relhumidity;
        zipcode     = randof (99999);
        temperature = randof (215) - 80;
        relhumidity = randof (50) + 10;

        //  Send message to all subscribers
        char update [20];
        sprintf (update, "%05d %d %d", zipcode, temperature, relhumidity);
        s_send (publisher, update);
    }
    zmq_close (publisher);
    zmq_ctx_destroy (context);
    return 0;
}
```
wuclient.c : 기상 변경 정보 구독
```cpp
//  Weather update client
//  Connects SUB socket to tcp://localhost:5556
//  Collects weather updates and finds avg temp in zipcode

#include "zhelpers.h"

int main (int argc, char *argv [])
{
    //  Socket to talk to server
    printf ("Collecting updates from weather server...\n");
    void *context = zmq_ctx_new ();
    void *subscriber = zmq_socket (context, ZMQ_SUB);
    int rc = zmq_connect (subscriber, "tcp://localhost:5557");
    assert (rc == 0);

    //  Subscribe to zipcode, default is NYC, 10001
    const char *filter = (argc > 1)? argv [1]: "10000";
    rc = zmq_setsockopt (subscriber, ZMQ_SUBSCRIBE,
                         filter, strlen (filter));
    assert (rc == 0);

    //  Process 100 updates
    int update_nbr;
    long total_temp = 0;
    for (update_nbr = 0; update_nbr < 100; update_nbr++) {
        char *string = s_recv (subscriber);

        int zipcode, temperature, relhumidity;
        sscanf (string, "%d %d %d",
            &zipcode, &temperature, &relhumidity);
        total_temp += temperature;
        free (string);
    }
    printf ("Average temperature for zipcode '%s' was %dF\n",
        filter, (int) (total_temp / update_nbr));

    zmq_close (subscriber);
    zmq_ctx_destroy (context);
    return 0;
}
```
wuproxy.c : XSUB와 XPUB를 사용하여 중개자 역할 수행
```cpp
//  Weather proxy device

#include "zhelpers.h"

int main (void)
{
    void *context = zmq_ctx_new ();

    //  This is where the weather server sits
    void *frontend = zmq_socket (context, ZMQ_XSUB);
    zmq_bind (frontend, "tcp://*:5556");

    //  This is our public endpoint for subscribers
    void *backend = zmq_socket (context, ZMQ_XPUB);
    zmq_bind (backend, "tcp://*:5557");

    //  Run the proxy until the user interrupts us
    zmq_proxy (frontend, backend, NULL);
    
    zmq_close (frontend);
    zmq_close (backend);
    zmq_ctx_destroy (context);
    return 0;
}
```
> [옮긴이] 빌드 및 테스트

~~~ {.bash}
PS D:\git_store\zguide-kr\examples\C> cl -EHsc wuserver.c libzmq.lib
PS D:\git_store\zguide-kr\examples\C> cl -EHsc wuclient.c libzmq.lib
PS D:\git_store\zguide-kr\examples\C> cl -EHsc wuproxy.c libzmq.lib

PS D:\git_store\zguide-kr\examples\C> ./wuserver

PS D:\git_store\zguide-kr\examples\C> ./wuproxy

PS D:\git_store\zguide-kr\examples\C> ./wuclient
Collecting updates from weather server...
Average temperature for zipcode '10000' was 18F
PS D:\git_store\zguide-kr\examples\C> ./wuclient
Collecting updates from weather server...
Average temperature for zipcode '10000' was 36F
~~~

### 공유 대기열(DEALER와 ROUTER 소켓)
;In the Hello World client/server application, we have one client that talks to one service. However, in real cases we usually need to allow multiple services as well as multiple clients. This lets us scale up the power of the service (many threads or processes or nodes rather than just one). The only constraint is that services must be stateless, all state being in the request or in some shared storage such as a database.

Hello World 클라이언트/서버 응용프로그램에서, 하나의 클라이언트는 하나의 서비스와 통신할 수 있었습니다. 그러나 실제상황에서는 다수의 서비스와 다수의 클라이언트에 대하여 사용 가능해야 하며, 이를 통해 서비스의 성능을 향상할 수 있습니다(단 하나의 스레드가 아닌 다수의 스레드들, 프로세스들 및 노드들). 유일한 제약은 서비스는 상태가 지정되지 않고, 요청 중인 모든 상태는 데이터베이스와 같은 일부 공유 스토리지에 저장되어야 합니다.

그림 15 - 요청 분배

![Request Distribution](images/fig15.svg)

;There are two ways to connect multiple clients to multiple servers. The brute force way is to connect each client socket to multiple service endpoints. One client socket can connect to multiple service sockets, and the REQ socket will then distribute requests among these services. Let's say you connect a client socket to three service endpoints; A, B, and C. The client makes requests R1, R2, R3, R4. R1 and R4 go to service A, R2 goes to B, and R3 goes to service C.

다수의 클라이언트들을 다수의 서버에 연결하는 방법에는 두 가지가 있습니다. 가장 강력한 방법은 각 클라이언트 소켓을 여러 서비스 단말에 연결하는 것입니다. 하나의 클라이언트 소켓은 여러 서비스 소켓에 연결할 수 있으며, REQ 소켓은 이러한 서비스 간에 요청을 분배합니다. 다시 말해 하나의 클라이언트 소켓을 세 개의 서비스 단말에 연결한다고 하면 : 서비스 단말들(A, B 및 C)에 대하여 클라이언트는 R1, R2, R3, R4를 요청합니다. R1 및 R4는 서비스 A로 이동하고 R2는 B로 이동하고 R3은 서비스 C로 이동합니다.

> [옮긴이] 서버에서 서비스를 제공하기 때문에, 서버란 용어 대신 서비스로 대체하여 사용하기도 합니다.

;This design lets you add more clients cheaply. You can also add more services. Each client will distribute its requests to the services. But each client has to know the service topology. If you have 100 clients and then you decide to add three more services, you need to reconfigure and restart 100 clients in order for the clients to know about the three new services.

이 디자인을 사용하면 더 많은 클라이언트를 쉽게 추가할 수 있습니다. 더 많은 서비스를 추가할 수도 있습니다. 각 클라이언트는 서비스에 요청들을 분배합니다. 그러나 각 클라이언트는 서비스 토폴로지(어디서 무슨 서비스를 제공하는지)를 알아야 합니다. 100개의 클라이언트들이 있고 3개의 서비스들을 추가하기로 결정한 경우 클라이언트들이 3개의 새로운 서비스에 대해 알 수 있도록 100 개의 클라이언트를 서비스들의 구성 정보를 변경하고 다시 시작해야 합니다.

;That's clearly not the kind of thing we want to be doing at 3 a.m. when our supercomputing cluster has run out of resources and we desperately need to add a couple of hundred of new service nodes. Too many static pieces are like liquid concrete: knowledge is distributed and the more static pieces you have, the more effort it is to change the topology. What we want is something sitting in between clients and services that centralizes all knowledge of the topology. Ideally, we should be able to add and remove services or clients at any time without touching any other part of the topology.

슈퍼 컴퓨팅 클러스터에 자원이 부족하여 수백 개의 새로운 서비스 노드들을 추가해야 할 경우, 새벽 3시에 일어나 하고 싶지는 않으실 겁니다. 너무 많은 정적 조각들은 액체 콘크리트와 같습니다 : 지식이 분산되고 정적 조각이 많을수록 네트워크 토폴로지를 변경하는 것에는 많은 노력이 필요합니다. 우리가 원하는 것은 클라이언트들과 서비스들 사이에 네트워크 토폴로지에 대한 모든 지식을 중앙화하는 것입니다. 이상적으로는 토폴로지의 다른 부분을 건드리지 않고 원격 서비스 또는 클라이언트를 추가 및 삭제할 수 있어야 합니다.

;So we'll write a little message queuing broker that gives us this flexibility. The broker binds to two endpoints, a frontend for clients and a backend for services. It then uses zmq_poll() to monitor these two sockets for activity and when it has some, it shuttles messages between its two sockets. It doesn't actually manage any queues explicitly—ØMQ does that automatically on each socket.

따라서 우리는 이러한 유연성을 제공하는 작은 메시지 대기열 브로커를 작성하겠습니다. 브로커는 클라이언트의 프론트엔드와 서비스의 백엔드의 두 단말을 바인딩합니다. 그런 다음 `zmq_poll()`을 사용하여이 두 소켓의 활동을 모니터링하고 소켓에 메세지가 전달 되었을 때, 두 소켓 사이에서 메시지를 오가도록 합니다. 브로커는 명시적으로 어떤 대기열도 관리하지 않습니다 - ØMQ에서 각 소켓에서 자동으로 처리하게 합니다.

;When you use REQ to talk to REP, you get a strictly synchronous request-reply dialog. The client sends a request. The service reads the request and sends a reply. The client then reads the reply. If either the client or the service try to do anything else (e.g., sending two requests in a row without waiting for a response), they will get an error.

REQ를 사용하여 REP와 통신할 때 엄격한 동기식 요청-응답가 필요하였습니다. 클라이언트가 요청을 보내면 서비스는 요청을 받아 응답을 보냅니다. 그런 다음 클라이언트는 응답을 받습니다. 만약 클라이언트나 서비스가 다른 작업을 시도하면(예 : 응답을 기다리지 않고 두 개의 요청을 연속으로 전송) 오류가 발생합니다.

;But our broker has to be nonblocking. Obviously, we can use zmq_poll() to wait for activity on either socket, but we can't use REP and REQ.

그러나 우리의 브로커는 비차단적으로 동작하며. 명확하게 `zmq_poll()`을 이용하여 어떤 소켓에 활동을 대기할 수 있지만, REP와 REQ 소켓은 사용할 수 없습니다.

그림 16 - 확장된 요청-응답

![Extended Request-Reply](images/fig16.svg)

;Luckily, there are two sockets called DEALER and ROUTER that let you do nonblocking request-response. You'll see in Chapter 3 - Advanced Request-Reply Patterns how DEALER and ROUTER sockets let you build all kinds of asynchronous request-reply flows. For now, we're just going to see how DEALER and ROUTER let us extend REQ-REP across an intermediary, that is, our little broker.

다행히도 비차단 요청-응답을 수행할 수 있는 DEALER와 ROUTER라는 2개의 소켓이 있습니다. "3장-고급 요청-응답 패턴"에서 DEALER 및 ROUTER 소켓을 사용하여 모든 종류의 비동기 요청-응답 흐름을 구축하는 방법을 보도록 하겠습니다. 지금은 DEALER와 ROUTER가 중개자로 요청-응답을 확장하는 방법을 보도록 하겠습니다.

;In this simple extended request-reply pattern, REQ talks to ROUTER and DEALER talks to REP. In between the DEALER and ROUTER, we have to have code (like our broker) that pulls messages off the one socket and shoves them onto the other.

이 간단한 확장된 요청-응답 패턴으로, REQ 소켓은 ROUTER 소켓과 통신하고 DEALER 소켓은 REP 소켓과 통신합니다. DEALER와 ROUTER 사이에는 하나의 소켓에서 메시지를 가져와서 다른 소켓으로 보내는 코드(`zmq_proxy()`)가 존재해야 합니다.

;The request-reply broker binds to two endpoints, one for clients to connect to (the frontend socket) and one for workers to connect to (the backend). To test this broker, you will want to change your workers so they connect to the backend socket. Here is a client that shows what I mean:

요청-응답 브로커는 2개의 단말에 바인딩합니다. 하나는 클라이언트들을 연결(프론트엔드 소켓)하고 다른 하나는 작업자들을 연결(백엔드 소켓)하는 것입니다. 이 브로커를 테스트하기 위하여 백엔드 소켓에 연결하는 작업자들의 개수를 변경할 수도 있습니다. 다음은 요청하는 클라이언트 소스 코드입니다.

rrclient.c: 요청-응답 클라이언트

```cpp
//  Hello World client
//  Connects REQ socket to tcp://localhost:5559
//  Sends "Hello" to server, expects "World" back

#include "zhelpers.h"

int main (void) 
{
    void *context = zmq_ctx_new ();

    //  Socket to talk to server
    void *requester = zmq_socket (context, ZMQ_REQ);
    zmq_connect (requester, "tcp://localhost:5559");

    int request_nbr;
    for (request_nbr = 0; request_nbr != 10; request_nbr++) {
        s_send (requester, "Hello");
        char *string = s_recv (requester);
        printf ("Received reply %d [%s]\n", request_nbr, string);
        free (string);
    }
    zmq_close (requester);
    zmq_ctx_destroy (context);
    return 0;
}
```

다음은 작업자(worker)의 코드입니다.

rrworker.c: 요청-응답 작업자

```cpp
//  Hello World worker
//  Connects REP socket to tcp://localhost:5560
//  Expects "Hello" from client, replies with "World"

#include "zhelpers.h"
#ifndef _WIN32
#include <unistd.h>
#endif

int main (void) 
{
    void *context = zmq_ctx_new ();

    //  Socket to talk to clients
    void *responder = zmq_socket (context, ZMQ_REP);
    zmq_connect (responder, "tcp://localhost:5560");

    while (1) {
        //  Wait for next request from client
        char *string = s_recv (responder);
        printf ("Received request: [%s]\n", string);
        free (string);

        //  Do some 'work'
        s_sleep (1000);

        //  Send reply back to client
        s_send (responder, "World");
    }
    //  We never get here, but clean up anyhow
    zmq_close (responder);
    zmq_ctx_destroy (context);
    return 0;
}
```

다음이 브로커 코드입니다. 멀티파트 메시지를 제대로 처리 할 수 있습니다.

rrbroker.c: 요청-응답 브로커

```cpp
//  Simple request-reply broker

#include "zhelpers.h"

int main (void) 
{
    //  Prepare our context and sockets
    void *context = zmq_ctx_new ();
    void *frontend = zmq_socket (context, ZMQ_ROUTER);
    void *backend  = zmq_socket (context, ZMQ_DEALER);
    zmq_bind (frontend, "tcp://*:5559");
    zmq_bind (backend,  "tcp://*:5560");

    //  Initialize poll set
    zmq_pollitem_t items [] = {
        { frontend, 0, ZMQ_POLLIN, 0 },
        { backend,  0, ZMQ_POLLIN, 0 }
    };
    //  Switch messages between sockets
    while (1) {
        zmq_msg_t message;
        zmq_poll (items, 2, -1);
        if (items [0].revents & ZMQ_POLLIN) {
            while (1) {
                //  Process all parts of the message
                zmq_msg_init (&message);
                zmq_msg_recv (&message, frontend, 0);
                int more = zmq_msg_more (&message);
                zmq_msg_send (&message, backend, more? ZMQ_SNDMORE: 0);
                zmq_msg_close (&message);
                if (!more)
                    break;      //  Last message part
            }
        }
        if (items [1].revents & ZMQ_POLLIN) {
            while (1) {
                //  Process all parts of the message
                zmq_msg_init (&message);
                zmq_msg_recv (&message, backend, 0);
                int more = zmq_msg_more (&message);
                zmq_msg_send (&message, frontend, more? ZMQ_SNDMORE: 0);
                zmq_msg_close (&message);
                if (!more)
                    break;      //  Last message part
            }
        }
    }
    //  We never get here, but clean up anyhow
    zmq_close (frontend);
    zmq_close (backend);
    zmq_ctx_destroy (context);
    return 0;
}

```
그림 17 - 요청-응답 브로커

![Request-Reply Broker](images/fig17.svg)

;Using a request-reply broker makes your client/server architectures easier to scale because clients don't see workers, and workers don't see clients. The only static node is the broker in the middle.

요청-응답 브로커를 사용하면 클라이언트는 작업자를 보지 못하고 작업자는 클라이언트를 보지 않기 때문에 클라이언트/서버 아키텍처를 쉽게 확장할 수 있습니다. 유일한 정적 노드는 중간에 있는 브로커입니다.

> [옮긴이] 빌드 및 테스트

~~~ {.bash}
PS D:\git_store\zguide-kr\examples\C> cl -EHsc rrclient.c libzmq.lib
PS D:\git_store\zguide-kr\examples\C> cl -EHsc rrworker.c libzmq.lib
PS D:\git_store\zguide-kr\examples\C> cl -EHsc rrproxy.c libzmq.lib

PS D:\git_store\zguide-kr\examples\C> ./rrbroker

PS D:\git_store\zguide-kr\examples\C> ./rrworker
Received request: [Hello]
Received request: [Hello]
Received request: [Hello]
Received request: [Hello]
Received request: [Hello]
...

PS D:\git_store\zguide-kr\examples\C> ./rrclient
Received reply 0 [World]
Received reply 1 [World]
Received reply 2 [World]
Received reply 3 [World]
Received reply 4 [World]
...
~~~

> [옮긴이] 이런 구조에서는 작업자에서 제공하는 서비스를 구분해야 할 경우(설비 연계 서비스, 사용자 연계 서비스, 트랜젝션 처리 서비스 등) 개별 서비스에 대하여 요청/응답 브로커에서 분배합니다. 클라이언트에서 각 서비스를 제공하는 작업자들의 정보를 등록하여 찾아가게 하거나, 브로커에 서비스 추가/삭제하여 클라이언트가 브로커에 서비스를 요청하고 제공받게 할 수 있습니다.
작업자의 서비스들을 클라이언트에 정적으로 보관하거나, 브로커에서 서비스 관리하지 않기 위해서는 클라이언트에서 필요한 서비스를 찾을 수 있도록 서비스를 식별하기 위한 중앙의 저장소를 두어 운영할 수도 있습니다.

### ØMQ의 내장된 프록시 함수
;It turns out that the core loop in the previous section's rrbroker is very useful, and reusable. It lets us build pub-sub forwarders and shared queues and other little intermediaries with very little effort. ØMQ wraps this up in a single method, zmq_proxy():

이전 섹션의 rrbroker의 코어 루프는 매우 유용하고 재사용 가능합니다. 적은 노력으로 PUB-SUB 포워더와 공유 대기열과 같은 작은 중개자를 구축할 수 있었습니다. ØMQ는 이 같은 기능을 감싼 하나의 함수 `zmq_proxy()`를 준비하고 있습니다.

```cpp
zmq_proxy (frontend, backend, capture);
```

> [옮긴이] 아래의 코드를 zmq_proxy() 함수에서 대응할 수 있습니다.

```cpp
    //  Initialize poll set
    zmq_pollitem_t items [] = {
        { frontend, 0, ZMQ_POLLIN, 0 },
        { backend,  0, ZMQ_POLLIN, 0 }
    };
    //  Switch messages between sockets
    while (1) {
        zmq_msg_t message;
        zmq_poll (items, 2, -1);
        if (items [0].revents & ZMQ_POLLIN) {
            while (1) {
                //  Process all parts of the message
                zmq_msg_init (&message);
                zmq_msg_recv (&message, frontend, 0);
                int more = zmq_msg_more (&message);
                zmq_msg_send (&message, backend, more? ZMQ_SNDMORE: 0);
                zmq_msg_close (&message);
                if (!more)
                    break;      //  Last message part
            }
        }
        if (items [1].revents & ZMQ_POLLIN) {
            while (1) {
                //  Process all parts of the message
                zmq_msg_init (&message);
                zmq_msg_recv (&message, backend, 0);
                int more = zmq_msg_more (&message);
                zmq_msg_send (&message, frontend, more? ZMQ_SNDMORE: 0);
                zmq_msg_close (&message);
                if (!more)
                    break;      //  Last message part
            }
        }
    }
```

> [옮긴이] 빌드 및 테스트

~~~ {.bash}
PS D:\git_store\zguide-kr\examples\C> cl -EHsc msgbroker.c libzmq.lib

PS D:\git_store\zguide-kr\examples\C> ./msgbroker

PS D:\git_store\zguide-kr\examples\C> ./rrworker
Received request: [Hello]
Received request: [Hello]
Received request: [Hello]
Received request: [Hello]
Received request: [Hello]
...

PS D:\git_store\zguide-kr\examples\C> ./rrclient
Received reply 0 [World]
Received reply 1 [World]
Received reply 2 [World]
Received reply 3 [World]
Received reply 4 [World]
...
~~~
;The two (or three sockets, if we want to capture data) must be properly connected, bound, and configured. When we call the zmq_proxy method, it's exactly like starting the main loop of rrbroker. Let's rewrite the request-reply broker to call zmq_proxy, and re-badge this as an expensive-sounding "message queue" (people have charged houses for code that did less):

2개의 소켓(또는 데이터를 캡처하려는 경우 3개 소켓)이 연결, 바인딩 및 구성되어야 합니다. `zmq_proxy()` 함수를 호출하면 rrbroker의 메인 루프를 시작하는 것과 같습니다. `zmq_proxy()`를 호출하도록 요청-응답 브로커를 다시 작성하겠습니다.

msgqueue.c: message queue broker

```cpp
//  Simple message queuing broker
//  Same as request-reply broker but using shared queue proxy

#include "zhelpers.h"

int main (void) 
{
    void *context = zmq_ctx_new ();

    //  Socket facing clients
    void *frontend = zmq_socket (context, ZMQ_ROUTER);
    int rc = zmq_bind (frontend, "tcp://*:5559");
    assert (rc == 0);

    //  Socket facing services
    void *backend = zmq_socket (context, ZMQ_DEALER);
    rc = zmq_bind (backend, "tcp://*:5560");
    assert (rc == 0);

    //  Start the proxy
    zmq_proxy (frontend, backend, NULL);

    //  We never get here...
    zmq_close (frontend);
    zmq_close (backend);
    zmq_ctx_destroy (context);
    return 0;
}
```

;If you're like most ØMQ users, at this stage your mind is starting to think, "What kind of evil stuff can I do if I plug random socket types into the proxy?" The short answer is: try it and work out what is happening. In practice, you would usually stick to ROUTER/DEALER, XSUB/XPUB, or PULL/PUSH.

대부분의 ØMQ 사용자의 경우, 이 단계에서 "임의의 소켓 유형을 프록시에 넣으면 어떤 일이 일어날까?" 생각합니다. 짧게 대답하면 : 그것을 시도하고 무슨 일이 일어나는지 보아야 합니다. 실제로는 보통 ROUTER/DEALER, XSUB/XPUB 또는 PULL/PUSH을 사용합니다.

### 전송방식 간의 연결(Transport Bridging)
;A frequent request from ØMQ users is, "How do I connect my ØMQ network with technology X?" where X is some other networking or messaging technology.

ØMQ 사용자의 계속되는 요청은 "기술 X와 ØMQ 네트워크를 어떻게 연결합니까?"입니다. 여기서 X는 다른 네트워킹 또는 메시징 기술입니다.

그림 18 - Pub-Sub 포워드 프록시(Forwarder Proxy)

![Pub-Sub Forwarder Proxy](images/fig18.svg)

;The simple answer is to build a bridge. A bridge is a small application that speaks one protocol at one socket, and converts to/from a second protocol at another socket. A protocol interpreter, if you like. A common bridging problem in ØMQ is to bridge two transports or networks.

간단한 대답은 브리지(Bridge)를 만드는 것입니다. 브리지는 한 소켓에서 하나의 통신규약을 말하고, 다른 소켓에서 두 번째 통신규약으로 변환하는 작은 응용프로그램입니다. 통신규약 번역기로 할 수 있으며, ØMQ는 2개의 서로 다른 전송방식과 네트워크를 연결할 수 있습니다.

;As an example, we're going to write a little proxy that sits in between a publisher and a set of subscribers, bridging two networks. The frontend socket (SUB) faces the internal network where the weather server is sitting, and the backend (PUB) faces subscribers on the external network. It subscribes to the weather service on the frontend socket, and republishes its data on the backend socket.

예제로, 발행자와 일련의 구독자들 간에 2개의 네트워크를 연결하는 작은 프록시를 작성하도록 하겠습니다. 프론트엔드 소켓은 날씨 서버가 있는 내부 네트워크를 향하며, 백엔드 소켓은 외부 네트워크의 일련의 구독자들을 향하게 합니다. 프록시의 프론트엔드 소켓에서 날씨 서비스에 구독하고 백엔드 소켓으로 데이터를 재배포하게 합니다.

wuproxy.c: 기상정보 변경 프록시

```cpp
//  Weather proxy device

#include "zhelpers.h"

int main (void)
{
    void *context = zmq_ctx_new ();

    //  This is where the weather server sits
    void *frontend = zmq_socket (context, ZMQ_XSUB);
    zmq_bind (frontend, "tcp://*:5556");

    //  This is our public endpoint for subscribers
    void *backend = zmq_socket (context, ZMQ_XPUB);
    zmq_bind (backend, "tcp://*:5557");

    //  Run the proxy until the user interrupts us
    zmq_proxy (frontend, backend, NULL);
    
    zmq_close (frontend);
    zmq_close (backend);
    zmq_ctx_destroy (context);
    return 0;
}
```
> [옮긴이] wuserver와 wuclient간에 wuproxy를 넣어서 테스트 가능합니다.

;It looks very similar to the earlier proxy example, but the key part is that the frontend and backend sockets are on two different networks. We can use this model for example to connect a multicast network (pgm transport) to a tcp publisher.

이전 프록시 예제와 매우 유사하지만, 중요한 부분은 프론트엔드와 백엔드 소켓이 서로 다른 2개의 네트워크에 있다는 것입니다. 예를 들어 이 모델을 사용하여 멀티캐스트 네트워크(pgm 전송방식)를 TCP 발행자에 연결할 수도 있습니다.

## 오류 처리와 ETERM
;ØMQ's error handling philosophy is a mix of fail-fast and resilience. Processes, we believe, should be as vulnerable as possible to internal errors, and as robust as possible against external attacks and errors. To give an analogy, a living cell will self-destruct if it detects a single internal error, yet it will resist attack from the outside by all means possible.

ØMQ의 오류 처리 철학은 빠른 실패와 회복의 조합입니다. 프로세스는 내부 오류에 대해 가능한 취약하고 외부 공격 및 오류에 대해 가능한 강력해야 대응해야 합니다. 유사한 예로 살아있는 세포는 내부 오류가 하나만 감지되면 스스로 죽지만, 외부로부터의 공격에는 모든 가능한 수단을 통하여 저항합니다.

;Assertions, which pepper the ØMQ code, are absolutely vital to robust code; they just have to be on the right side of the cellular wall. And there should be such a wall. If it is unclear whether a fault is internal or external, that is a design flaw to be fixed. In C/C++, assertions stop the application immediately with an error. In other languages, you may get exceptions or halts.

어설션(assertion)은 ØMQ의 안정적인 코드에 필수적인 조미료입니다. 그들은 세포막에 있어야 하며 그런 막을 통하여 결함이 내부 또는 외부인지 구분하고, 불확실한 경우 설계 결함으로 수정해야 합니다.. C/C++에서 어설션은 오류 발생 시 응용프로그램을 즉시 중지합니다. 다른 언어에서는 예외 혹은 중지가 발생할 수 있습니다.

> [옮긴이] 어설션(assertion) 사용 사례(msgqueue.c에서 발취), assert()는 조건식이 거짓(false) 일 때 프로그램을 중단하며 참(true) 일 때는 프로그램이 계속 실행합니다.

```cpp
    ...
    //  Socket facing clients
    void *frontend = zmq_socket (context, ZMQ_ROUTER);
    int rc = zmq_bind (frontend, "tcp://*:5559");
    assert (rc == 0);

    //  Socket facing services
    void *backend = zmq_socket (context, ZMQ_DEALER);
    rc = zmq_bind (backend, "tcp://*:5560");
    assert (rc == 0);
    ...
```

;When ØMQ detects an external fault it returns an error to the calling code. In some rare cases, it drops messages silently if there is no obvious strategy for recovering from the error.

ØMQ가 외부 오류를 감지하면 호출 코드에 오류를 반환합니다. 드문 경우지만 오류를 복구하기 위한 명확한 전략이 없으면 메시지를 자동으로 삭제합니다.

;In most of the C examples we've seen so far there's been no error handling. Real code should do error handling on every single ØMQ call. If you're using a language binding other than C, the binding may handle errors for you. In C, you do need to do this yourself. There are some simple rules, starting with POSIX conventions:

지금까지 살펴본 대부분의 C 예제에서는 오류 처리가 없었습니다. 실제 코드는 모든  ØMQ 함수 호출에서 오류 처리를 수행해야 하며, C 이외의 다른 개발 언어 바인딩(예 : java)을 사용하는 경우 바인딩이 오류를 처리할 수 있습니다. C 개발 언어에서는 오류 처리 작업을 직접 해야 합니다. POSIX 관례로 시작하는 몇 가지 간단한 규칙이 있습니다.

;* Methods that create objects return NULL if they fail.
;* Methods that process data may return the number of bytes processed, or -1 on an error or failure.
;* Other methods return 0 on success and -1 on an error or failure.
;* The error code is provided in errno or zmq_errno().
;* A descriptive error text for logging is provided by zmq_strerror().

* 객체 생성 함수의 경우 실패하면 NULL 값을 반환합니다.
* 데이터 처리하는 함수의 경우 처리되는 데이터의 바이트 크기를 반환하지만, 오류나 실패 시 -1을 반환합니다.
* 기타 처리 함수들의 경우 반환값이 0이면 성공, -1이면 실패입니다.
* 오류 코드는 `errno` 혹은 `zmq_errno(void)`에서 제공합니다.
* 로그를 위한 오류 메시지는 `zmq_strerror(int errnum)`를 통해 제공됩니다.

사용 예는 다음과 같습니다.

```cpp
void *context = zmq_ctx_new ();
assert (context);
void *socket = zmq_socket (context, ZMQ_REP);
assert (socket);
int rc = zmq_bind (socket, "tcp://*:5555");
if (rc == -1) {
    printf ("E: bind failed: %s\n", strerror (errno));
    return -1;
}
```
> [옮긴이] hwserver.c를 수정(hwserver1.c)하여 "strerror(errno)" 확인하겠습니다.

hwserver1.c : Hellow World 서버 수정본
```cpp
//  Hello World server

#include <zmq.h>
#include <stdio.h>
#ifndef _WIN32
#include <unistd.h>
#else
#include <windows.h>
#define sleep(n)    Sleep(n*1000)
#endif
#include <string.h>
#include <assert.h>

int main (void)
{
    //  Socket to talk to clients
    void *context = zmq_ctx_new ();
    void *responder = zmq_socket (context, ZMQ_REP);
    int rc = zmq_bind (responder, "tcp://*:5555");
    if (rc == -1){
        printf("E: bind failed: %s\n", zmq_strerror(zmq_errno()));
        return -1;
    }

    while (1) {
        char buffer [10];
        zmq_recv (responder, buffer, 10, 0);
        printf ("Received Hello\n");
        sleep (1);          //  Do some 'work'
        zmq_send (responder, "World", 5, 0);
    }
    return 0;
}
```
> [옮긴이] 빌드 및 테스트

~~~{.bash}
PS D:\git_store\zguide-kr\examples\C> cl -EHsc hwserver1.c libzmq.lib
Microsoft (R) C/C++ 최적화 컴파일러 버전 19.16.27035(x64)
Copyright (c) Microsoft Corporation. All rights reserved.
hwserver1.c
Microsoft (R) Incremental Linker Version 14.16.27035.0
Copyright (C) Microsoft Corporation.  All rights reserved.
/out:hwserver1.exe
hwserver1.obj
libzmq.lib

PS D:\git_store\zguide-kr\examples\C> ./hwserver1

PS D:\git_store\zguide-kr\examples\C> ./hwserver1
E: bind failed: Address in use
~~~

;There are two main exceptional conditions that you should handle as nonfatal:

치명적이지 않은 것으로 취급해야 하는 두 가지 주요 예외 조건이 있습니다 : 

;* When your code receives a message with the ZMQ_DONTWAIT option and there is no waiting data, ØMQ will return -1 and set errno to EAGAIN.
;* When one thread calls zmq_ctx_destroy(), and other threads are still doing blocking work, the zmq_ctx_destroy() call closes the context and all blocking calls exit with -1, and errno set to ETERM.

* 소스 코드 내에서 `ZMQ_DONTWAIT` 옵션이 포함된 메시지가 수신되고 대기 데이터가 없는 경우 ØMQ는 -1을 반환하고 errno를 EAGAIN으로 설정합니다.
* 하나의 스레드가 `zmq_ctx_destroy()`를 호출하고 다른 스레드가 여전히 차단 작업을 수행하는 경우 `zmq_ctx_destroy()` 호출은 컨텍스트를 닫고 모든 차단 호출은 -1로 종료되고 errno는 ETERM으로 설정됩니다.

;In C/C++, asserts can be removed entirely in optimized code, so don't make the mistake of wrapping the whole ØMQ call in an assert(). It looks neat; then the optimizer removes all the asserts and the calls you want to make, and your application breaks in impressive ways.

C/C ++의 `assert()`는 최적화에 의해 완전히 제거되므로 `assert()`내에서 ØMQ API를 호출해서는 안됩니다.
최적화를 통해 모든`assert()`는 제거된 응용프로그램에서 개발자의 의도에 따라 ØMQ API 호출을 작성하여 오류가 발생할 경우 중단하게 해야 합니다.

> [옮긴이] assert() 매크로는 assert.h 헤더 파일에 정의되어 있으며, 원도우 VC 2017에서 디버그 모드에서만 동작하며 최적화된 릴리즈 모드에서는 제외됩니다.

그림 19 - 강제 종료 신호를 가진 병렬 파이프라인(Pipeline)

![Parallel Pipeline with Kill Signaling](images/fig19.svg)

;Let's see how to shut down a process cleanly. We'll take the parallel pipeline example from the previous section. If we've started a whole lot of workers in the background, we now want to kill them when the batch is finished. Let's do this by sending a kill message to the workers. The best place to do this is the sink because it really knows when the batch is done.

프로세스를 깨끗하게 종료하는 방법을 알아보겠습니다. 이전 섹션의 병렬 파이프라인 예제를 보면, 백그라운드에서 많은 수의 작업자를 시작한 후 작업이 완료되면 수동으로 해당 작업자를 종료해야 했습니다. 이것을 작업이 완료되면 작업자에서 종료 신호를 보내 자동으로 종료하도록 하겠습니다. 이 작업을 수행하기 가장 좋은 대상은 작업이 완료된 시점을 알고 있는 수집기입니다.

;How do we connect the sink to the workers? The PUSH/PULL sockets are one-way only. We could switch to another socket type, or we could mix multiple socket flows. Let's try the latter: using a pub-sub model to send kill messages to the workers:

수집기와 작업자를 연결하기 위한 방법은 단방향 PUSH/PULL 소켓입니다. 다른 소켓 유형으로 전환하거나 다중 소켓을 혼합할 수 있습니다. 후자를 선택하여 : 발행-구독 모델로 수집기에서 종료 메시지를 작업자에게 보냅니다.

;* The sink creates a PUB socket on a new endpoint.
;* Workers bind their input socket to this endpoint.
;* When the sink detects the end of the batch, it sends a kill to its PUB socket.
;* When a worker detects this kill message, it exits.

* 수집기는 새로운 단말에 PUB 소켓을 생성합니다.
* 작업자들은 SUB 소켓을 수집기 단말에 연결합니다.
* 수집기가 일괄 작업의 종료 감지하면 PUB 소켓에 종료 신호를 보냅니다.
* 작업자들이 종료 메시지(kill message)를 감지하면 종료됩니다.

; It doesn't take much new code in the sink:

이러한 작업은 수집기에서 새로운 코드 추가가 필요합니다.

```cpp
void *controller = zmq_socket (context, ZMQ_PUB);
zmq_bind (controller, "tcp://*:5559");
…
// 작업자(worker)를 종료시키는 신호를 전송
s_send (controller, "KILL");
```

;Here is the worker process, which manages two sockets (a PULL socket getting tasks, and a SUB socket getting control commands), using the zmq_poll() technique we saw earlier:

작업자 프로세스에서는 2개의 소켓(호흡기에서 작업을 받아 오는 PULL 소켓과 제어 명령을 가져오는 SUB 소켓)을 이전에 보았던 `zmq_poll()` 기술을 사용해서 관리합니다.

taskwork2.c: 종료 신호를 처리하는 작업자

```cpp
//  Task worker - design 2
//  Adds pub-sub flow to receive and respond to kill signal

#include "zhelpers.h"

int main (void) 
{
    //  Socket to receive messages on
    void *context = zmq_ctx_new ();
    void *receiver = zmq_socket (context, ZMQ_PULL);
    zmq_connect (receiver, "tcp://localhost:5557");

    //  Socket to send messages to
    void *sender = zmq_socket (context, ZMQ_PUSH);
    zmq_connect (sender, "tcp://localhost:5558");

    //  Socket for control input
    void *controller = zmq_socket (context, ZMQ_SUB);
    zmq_connect (controller, "tcp://localhost:5559");
    zmq_setsockopt (controller, ZMQ_SUBSCRIBE, "", 0);

    //  Process messages from either socket
    while (1) {
        zmq_pollitem_t items [] = {
            { receiver, 0, ZMQ_POLLIN, 0 },
            { controller, 0, ZMQ_POLLIN, 0 }
        };
        zmq_poll (items, 2, -1);
        if (items [0].revents & ZMQ_POLLIN) {
            char *string = s_recv (receiver);
            printf ("%s.", string);     //  Show progress
            fflush (stdout);
            s_sleep (atoi (string));    //  Do the work
            free (string);
            s_send (sender, "");        //  Send results to sink
        }
        //  Any waiting controller command acts as 'KILL'
        if (items [1].revents & ZMQ_POLLIN)
            break;                      //  Exit loop
    }
    zmq_close (receiver);
    zmq_close (sender);
    zmq_close (controller);
    zmq_ctx_destroy (context);
    return 0;
}
```

;Here is the modified sink application. When it's finished collecting results, it broadcasts a kill message to all workers:

수정된 수집기 응용프로그램에서는 결과 수집이 완료되면 모든 작업자들에게 종료 메시지를 전송합니다.

tasksink2.c: 강제 종료(kill) 신호를 가진 병렬 작업 수집기(sink)
```cpp
//  Task sink - design 2
//  Adds pub-sub flow to send kill signal to workers

#include "zhelpers.h"

int main (void) 
{
    //  Socket to receive messages on
    void *context = zmq_ctx_new ();
    void *receiver = zmq_socket (context, ZMQ_PULL);
    zmq_bind (receiver, "tcp://*:5558");

    //  Socket for worker control
    void *controller = zmq_socket (context, ZMQ_PUB);
    zmq_bind (controller, "tcp://*:5559");

    //  Wait for start of batch
    char *string = s_recv (receiver);
    free (string);
    
    //  Start our clock now
    int64_t start_time = s_clock ();

    //  Process 100 confirmations
    int task_nbr;
    for (task_nbr = 0; task_nbr < 100; task_nbr++) {
        char *string = s_recv (receiver);
        free (string);
        if (task_nbr % 10 == 0)
            printf (":");
        else
            printf (".");
        fflush (stdout);
    }
    printf ("Total elapsed time: %d msec\n", 
        (int) (s_clock () - start_time));

    //  Send kill signal to workers
    s_send (controller, "KILL");

    zmq_close (receiver);
    zmq_close (controller);
    zmq_ctx_destroy (context);
    return 0;
}
```

> [옮긴이] 빌드 및 테스트

~~~{.bash}
PS D:\git_store\zguide-kr\examples\C> cl -EHsc taskwork2.c libzmq.lib
PS D:\git_store\zguide-kr\examples\C> cl -EHsc tasksink2.c libzmq.lib

PS D:\git_store\zguide-kr\examples\C> ./taskvent
Press Enter when the workers are ready:
Sending tasks to workers...
Total expected cost: 5086 msec
PS D:\git_store\zguide-kr\examples\C>

PS D:\git_store\zguide-kr\examples\C> ./taskwork2
13.22.47.94.99.65.5.88.38.22.36.27.3.47.98.36.66.43.82.93.5.91.41.87.43.7.43.41.94.48.43.43.74.66.53.7.46.16.21.94.59.10.88.12.59.40.27.43.26.44.51.3.4.59.85.81.96.16.72.71.86.26.46.37.45.20.8.27.15.83.51.80.86.51.71.36.96.4.8.93.58.32.56.22.27.100.81.98.38.96.91.27.9.82.94.57.52.33.92.39.
PS D:\git_store\zguide-kr\examples\C>

PS D:\git_store\zguide-kr\examples\C> ./tasksink2
:.........:.........:.........:.........:.........:.........:.........:.........:.........:.........Total elapsed time: 5184 msec
PS D:\git_store\zguide-kr\examples\C>
~~~

## 인터럽트 신호 처리
;Realistic applications need to shut down cleanly when interrupted with Ctrl-C or another signal such as SIGTERM. By default, these simply kill the process, meaning messages won't be flushed, files won't be closed cleanly, and so on.

실제 응용프로그램에서는 Ctrl-C나 SIGTERM과 같은 신호로 인한 인터럽트가 왔을 때 작업 종료 처리가 필요하다. 기본적으로 이러한 인터럽트는 단순히 프로세스만 종료하지 메시지 대기열을 정리하지 않고, 파일을 깨끗하게 닫지 않습니다.

;Here is how we handle a signal in various languages:

다양한 개발 언어에서 신호를 처리하는 방법을 보겠습니다.

> [옮긴이] 신호 처리를 위한 예제는 리눅스/유닉스에서 실행 필요합니다.

interrupt.c: Ctrl-C를 제대로 처리하는 방법
```cpp
//  Shows how to handle Ctrl-C

#include <stdlib.h>
#include <stdio.h>
#include <signal.h>
#include <unistd.h>
#include <fcntl.h>

#include <zmq.h>

//  Signal handling
//
//  Create a self-pipe and call s_catch_signals(pipe's writefd) in your application
//  at startup, and then exit your main loop if your pipe contains any data.
//  Works especially well with zmq_poll.

#define S_NOTIFY_MSG " "
#define S_ERROR_MSG "Error while writing to self-pipe.\n"
static int s_fd;
static void s_signal_handler (int signal_value)
{
    int rc = write (s_fd, S_NOTIFY_MSG, sizeof(S_NOTIFY_MSG));
    if (rc != sizeof(S_NOTIFY_MSG)) {
        write (STDOUT_FILENO, S_ERROR_MSG, sizeof(S_ERROR_MSG)-1);
        exit(1);
    }
}

static void s_catch_signals (int fd)
{
    s_fd = fd;

    struct sigaction action;
    action.sa_handler = s_signal_handler;
    //  Doesn't matter if SA_RESTART set because self-pipe will wake up zmq_poll
    //  But setting to 0 will allow zmq_read to be interrupted.
    action.sa_flags = 0;
    sigemptyset (&action.sa_mask);
    sigaction (SIGINT, &action, NULL);
    sigaction (SIGTERM, &action, NULL);
}

int main (void)
{
    int rc;

    void *context = zmq_ctx_new ();
    void *socket = zmq_socket (context, ZMQ_REP);
    zmq_bind (socket, "tcp://*:5555");

    int pipefds[2];
    rc = pipe(pipefds);
    if (rc != 0) {
        perror("Creating self-pipe");
        exit(1);
    }
    int flags = fcntl(pipefds[0], F_GETFL, 0);
    if (flags < 0) {
        perror ("fcntl(F_GETFL)");
        exit(1);
    }
    rc = fcntl (pipefds[0], F_SETFL, flags | O_NONBLOCK);
    if (rc != 0) {
        perror ("fcntl(F_SETFL)");
        exit(1);
    }

    s_catch_signals (pipefds[1]);

    zmq_pollitem_t items [] = {
        { 0, pipefds[0], ZMQ_POLLIN, 0 },
        { socket, 0, ZMQ_POLLIN, 0 }
    };

    while (1) {
        rc = zmq_poll (items, 2, -1);
        if (rc == 0) {
            continue;
        } else if (rc < 0) {
            if (errno == EINTR) { continue; }
            perror("zmq_poll");
            exit(1);
        }

        // Signal pipe FD
        if (items [0].revents & ZMQ_POLLIN) {
            char buffer [1];
            read (pipefds[0], buffer, 1);  // clear notifying byte
            printf ("W: interrupt received, killing server...\n");
            break;
        }

        // Read socket
        if (items [1].revents & ZMQ_POLLIN) {
            char buffer [255];
            // Use non-blocking so we can continue to check self-pipe via zmq_poll
            rc = zmq_recv (socket, buffer, 255, ZMQ_DONTWAIT);
            if (rc < 0) {
                if (errno == EAGAIN) { continue; }
                if (errno == EINTR) { continue; }
                perror("recv");
                exit(1);
            }
            printf ("W: recv\n");

            // Now send message back.
            // ...
        }
    }

    printf ("W: cleaning up\n");
    zmq_close (socket);
    zmq_ctx_destroy (context);
    return 0;
}
```
> 
> [옮긴이] 빌드 및 테스트(CentOS 7에서 수행함)

~~~{.bash}
[gmsec@felix C]$ gcc -o interrupt interrupt.c -lzmq
[gmsec@felix C]$ gcc -o hwclient hwclient.c -lzmq
[gmsec@felix C]$ ./hwclient
Connecting to hello world server...
Sending Hello 0...
Received World 0
Sending Hello 1...
Received World 1
Sending Hello 2...

[gmsec@felix C]$ ./interrupt
W: recv
W: recv
^C
W: interrupt received, killing server...
W: cleaning up
~~~

;The program provides s_catch_signals(), which traps Ctrl-C (SIGINT) and SIGTERM. When either of these signals arrive, the s_catch_signals() handler sets the global variable s_interrupted. Thanks to your signal handler, your application will not die automatically. Instead, you have a chance to clean up and exit gracefully. You have to now explicitly check for an interrupt and handle it properly. Do this by calling s_catch_signals() (copy this from interrupt.c) at the start of your main code. This sets up the signal handling. The interrupt will affect ØMQ calls as follows:

이 프로그램은 `s_catch_signals()`함수가 Ctrl-C(SIGINT) 및 SIGTERM 신호를 잡을 수 있게 하며 `s_catch_signals()` 핸들러는 전역 변수 `s_interrupted`를 설정합니다. 신호 처리기 덕분에 응용프로그램이 자동으로 종료되지 않게 하여 의도에 따라 자원을 정리하고 종료할 수 있습니다. 이제 명시적으로 인터럽트 발생을 확인하고 올바르게 처리해야 합니다. 메인 코드 시작 시 `s_catch_signals()`(interrupt.c에서 복사)를 호출하여 신호 처리를 설정할 수 있습니다. 인터럽트는 다음과 같이 ØMQ API 호출에 영향을 미칩니다.

;* If your code is blocking in a blocking call (sending a message, receiving a message, or polling), then when a signal arrives, the call will return with EINTR.
;* Wrappers like s_recv() return NULL if they are interrupted.

* 코드가 차단 호출(메시지 전송, 메시지 수신 또는 폴링)에서 차단된 경우, 신호가 도착하면 호출은 `EINTR` 반환합니다.
* `s_recv()`와 같은 래퍼 함수는 인터럽트 발생 시 `NULL`을 반환합니다.

;So check for an EINTR return code, a NULL return, and/or s_interrupted.

따라서 `EINTR ` 반환 코드, NULL 반환 시 `s_interrupted` 전역 변수를 점검하십시오.

;Here is a typical code fragment:

전형적인 사용 예시입니다.

```cpp
s_catch_signals ();
client = zmq_socket (...);
while (!s_interrupted) {
    char *message = s_recv (client);
    if (!message)
        break;          //  Ctrl-C 사용될 경우
}
zmq_close (client);
```

;If you call s_catch_signals() and don't test for interrupts, then your application will become immune to Ctrl-C and SIGTERM, which may be useful, but is usually not.

`s_catch_signals()`를 호출하고 인터럽트를 테스트하지 않으면 응용프로그램이 Ctrl-C 및 SIGTERM에 무시하게 되어, 유용할지는 몰라도 보통은 그렇지 않습니다.

## 메모리 누수 탐지
;Any long-running application has to manage memory correctly, or eventually it'll use up all available memory and crash. If you use a language that handles this automatically for you, congratulations. If you program in C or C++ or any other language where you're responsible for memory management, here's a short tutorial on using valgrind, which among other things will report on any leaks your programs have.

오랫동안 구동되는 응용프로그램의 경우 올바르게 메모리 관리를 수행하지 않으면 사용 가능한 메모리 공간을 차지하여 충돌이 발생합니다. 메모리 관리를 자동으로 수행하는 개발 언어를 사용한다면 축하할 일이지만 C 나 C++와 같이 메모리 관리가 필요한 개발 언어로 프로그래밍하는 경우 `valgrind` 사용을 통하여 프로그램에서 메모리 누수(leak)를 탐지할 수 있습니다.

;* To install valgrind, e.g., on Ubuntu or Debian, issue this command:

유분투 혹은 데비안 운영체제에서 valgrind를 설치하려면 다음 명령을 실행하십시오.

~~~ {.bash}
sudo apt-get install valgrind
~~~

> [옮긴이] CentOS의 경우 아래와 같이 설치를 진행합니다.

~~~ {.bash}
[root@felix C]# yum install valgrind
Last metadata expiration check: 0:48:43 ago on Fri 17 Jan 2020 09:34:23 AM KST.
Package valgrind-1:3.14.0-10.el8.x86_64 is already installed.
Dependencies resolved.
====================================================================
 Package    Arch   Version      Repository     Size
====================================================================
Upgrading:
 valgrind   x86_64  1:3.15.0-9.el8   AppStream  12 M
 valgrind-devel x86_64 1:3.15.0-9.el8   AppStream   90 k
Transaction Summary
============================================================================
Upgrade  2 Packages

Total download size: 12 M
Is this ok [y/N]: y
Downloading Packages:
(1/2): valgrind-devel-3.15.0-9.el8.x86_64.rpm     992 kB/s |  90 kB     00:00    
(2/2): valgrind-3.15.0-9.el8.x86_64.rpm      3.6 MB/s |  12 MB     00:03    
----------------------------------------------------------------------------
... 
~~~

;* By default, ØMQ will cause valgrind to complain a lot. To remove these warnings, create a file called vg.supp that contains this:

기본적으로 ØMQ는 valgrind가 많은 로그를 발생시키며, 이러한 경고를 제거하려면 다음 내용을 포함하는 `vg.supp`라는 파일을 작성하십시오.

~~~
{
   <socketcall_sendto>
   Memcheck:Param
   socketcall.sendto(msg)
   fun:send
   ...
}
{
    <socketcall_sendto>
    Memcheck:Param
    socketcall.send(msg)
    fun:send
    ...
}
~~~

;* Fix your applications to exit cleanly after Ctrl-C. For any application that exits by itself, that's not needed, but for long-running applications, this is essential, otherwise valgrind will complain about all currently allocated memory.

Ctrl-C 수행 후에 응용프로그램이 완전히 종료되도록 수정하십시오. 자체 종료되는 응용프로그램에서는 필요하지 없겠지만, 오랫동안 구동되는 응용프로그램의 경우 필수이며, 그렇지 않을 경우 `valgrind`가 현재 할당된 모든 메모리에 대하여 경고를 표시합니다.

;* Build your application with -DDEBUG if it's not your default setting. That ensures valgrind can tell you exactly where memory is being leaked.

응용프로그램을 빌드 수행 시 -DDEBUG 옵션을 사용하면 `valgrind`가 메모리 누수 위치를 정확하게 알려 줄 수 있습니다.

;* Finally, run valgrind thus:

* 마지막으로 'vagrind'을 아래와 같이 실행하십시오.

~~~ {.bash}
valgrind --tool=memcheck --leak-check=full --suppressions=vg.supp someprog
~~~

;And after fixing any errors it reported, you should get the pleasant message:

그리고 보고된 오류를 수정하면 아래와 같이 행복한 메시지가 출력됩니다.

~~~
==30536== ERROR SUMMARY: 0 errors from 0 contexts...
~~~
> [옮긴이] CentOS의 경우 아래와 테스트를 수행합니다.

~~~ {.bash}
[gmsec@felix C]$ ./interrupt
^CW: interrupt received, killing server...
W: cleaning up

[gmsec@felix C]$ valgrind --tool=memcheck --leak-check=full ./interrupt
==20245== Memcheck, a memory error detector
==20245== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==20245== Using Valgrind-3.15.0 and LibVEX; rerun with -h for copyright info
==20245== Command: ./interrupt
==20245== 
^CW: interrupt received, killing server...
W: cleaning up
==20245== 
==20245== HEAP SUMMARY:
==20245==     in use at exit: 0 bytes in 0 blocks
==20245==   total heap usage: 37 allocs, 37 frees, 97,084 bytes allocated
==20245== 
==20245== All heap blocks were freed -- no leaks are possible
==20245== 
==20245== For lists of detected and suppressed errors, rerun with: -s
==20245== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
~~~ 

## ØMQ에서 멀티스레드(Multithread)
;ØMQ is perhaps the nicest way ever to write multithreaded (MT) applications. Whereas ØMQ sockets require some readjustment if you are used to traditional sockets, ØMQ multithreading will take everything you know about writing MT applications, throw it into a heap in the garden, pour gasoline over it, and set it alight. It's a rare book that deserves burning, but most books on concurrent programming do.

ØMQ는 멀티스레드 응용프로그램의 작성에 최적화되어 있습니다. 전통적인 소켓을 사용하셨다면, ØMQ 소켓은 사용 시 약간의 재조정이 필요하지만, ØMQ 멀티스레딩은 기존에 알고 계셨던 멀티스레드 응용프로그램 작성 경험이 거의 필요하지 않습니다. 기존의 지식을 정원에 던져버리고 기름을 부어 태워 버리십시오. 책(지식)을 불태우는 경우는 드물지만, 동시성 프로그래밍의 경우 필요합니다.

;To make utterly perfect MT programs (and I mean that literally), we don't need mutexes, locks, or any other form of inter-thread communication except messages sent across ØMQ sockets.

ØMQ에서는 완벽한 멀티스레드 프로그램을 만들기 위해(그리고 문자 그대로), 뮤텍스, 잠금이 불필요하며 ØMQ 소켓을 통해 전송되는 메시지를 제외하고는 어떤 다른 형태의 스레드 간 통신이 필요하지 않습니다.

> [옮긴이] 전통적인 멀티스레드 프로그램에서는 스레드 간의 동기화를 위해 뮤텍스, 잠금, 세마포어를 사용하여 교착을 회피합니다.

;By "perfect MT programs", I mean code that's easy to write and understand, that works with the same design approach in any programming language, and on any operating system, and that scales across any number of CPUs with zero wait states and no point of diminishing returns.

"완벽한 멀티스레드 프로그램"은 작성하기 쉽고 이해하기 쉬운 코드로 어떤 프로그래밍 언어와 운영체제에서도 동일한 설계 방식으로 동작하며, CPU 수에 비례하여 자원의 낭비 없이 성능이 확장되어야 합니다.

;If you've spent years learning tricks to make your MT code work at all, let alone rapidly, with locks and semaphores and critical sections, you will be disgusted when you realize it was all for nothing. If there's one lesson we've learned from 30+ years of concurrent programming, it is: just don't share state. It's like two drunkards trying to share a beer. It doesn't matter if they're good buddies. Sooner or later, they're going to get into a fight. And the more drunkards you add to the table, the more they fight each other over the beer. The tragic majority of MT applications look like drunken bar fights.

멀티스레드 코드 작성하기 위한 기술(잠금 및 세마포어, 크리티컬 섹션 등)을 배우기 위해 수년의 시간을 보냈다면, 그것이 아무것도 아니라는 것을 깨닫는 순간 어처구니가 없을 것입니다. 30년 이상의 동시성 프로그래밍에서 배운 교훈이 있다면, 그것은 단지 "상태(& 자원)를 공유하지 마십시오"입니다. 이것은 한잔의 맥주를 같이 마시려고 하는 두 명의 술꾼들과 같습니다. 그들이 좋은 친구인지는 중요하지 않습니다. 조만간 그들은 싸우게 될 것입니다. 그리고 술꾼들과 테이블이 많을수록 맥주를 놓고 서로 더 많이 싸우게 됩니다. 멀티스레드 응용프로그램의 비극의 대다수는 술집에서 술꾼들의 싸움처럼 보입니다.

;The list of weird problems that you need to fight as you write classic shared-state MT code would be hilarious if it didn't translate directly into stress and risk, as code that seems to work suddenly fails under pressure. A large firm with world-beating experience in buggy code released its list of "11 Likely Problems In Your Multithreaded Code", which covers forgotten synchronization, incorrect granularity, read and write tearing, lock-free reordering, lock convoys, two-step dance, and priority inversion.

고전적인 공유 상태 멀티스레드 코드를 작성할 때 싸워야 할 일련의 이상한 문제들은 스트레스와 위험으로 직결되지 않는다면 재미있을 수도 있지만, 이러한 코드는 정상적으로 동작하는 것처럼 보여도 갑자기 중단될 수 있습니다. 세계 최고의 경험을 가진 대기업(Microsoft)에서 "멀티스레드 코드에서 발생 가능한 11 개의 문제" 목록을 발표하였으며,  잊힌 동기화(forgotten synchronization), 부적절한 세분화(incorrect granularity), 읽기 및 쓰기 분열(read and write tearing), 잠금 없는 순서 변경(lock-free reordering), 잠금 호송(lock convoys), 2단계 댄스(two-step dance) 및 우선순위 반전(priority inversion)을 다루고 있습니다.

> [옮긴이] 잠금 호송(lock convoys)은 잠금(Lock)을 소유한 스레드가 스케줄링에서 제외되어 지연됨으로써 그 잠금(Lock) 해제를 기다리는 다른 스레드(Thread)들도 함께 지연되는 현상입니다.

;Yeah, we counted seven problems, not eleven. That's not the point though. The point is, do you really want that code running the power grid or stock market to start getting two-step lock convoys at 3 p.m. on a busy Thursday? Who cares what the terms actually mean? This is not what turned us on to programming, fighting ever more complex side effects with ever more complex hacks.

몰론 위에서는 11개가 아닌 7개의 문제만 나열하였습니다. 하지만 요지는 발전소 혹은 주식 시장과 같이 기간산업에 적용된 코드에서, 바쁜 목요일 오후 3시에 2 단계 잠금 호송으로 인하여 처리가 지연되기를 원하지는 않습니다. 용어가 실제로 의미하는 것에는 누구도 신경 쓰지 않으며 이것으로 프로그래밍이 되지도 않으며 더 복잡한 해결 방안으로 인한 더 복잡한 부작용과 싸우게 됩니다.

;Some widely used models, despite being the basis for entire industries, are fundamentally broken, and shared state concurrency is one of them. Code that wants to scale without limit does it like the Internet does, by sending messages and sharing nothing except a common contempt for broken programming models.

이와 같이 널리 사용되는 모델들은 전체 산업의 기반임에도 불구하고 근본적으로 잘못되었으며, 공유 상태 동시성(shared state concurrency)은 그중 하나입니다. 인터넷이 그러하듯 제한 없이 확장하고자 하는 코드는 메시지를 보내고 어떤 것도 공유하지 않는 것입니다(결함 있는(broken) 프로그래밍 모델에 대한 일반적인 오류는 제외).

;You should follow some rules to write happy multithreaded code with ØMQ:

ØMQ로 즐거운 멀티스레드 코드를 작성하려면 몇 가지 규칙을 따라야 합니다.

;* Isolate data privately within its thread and never share data in multiple threads. The only exception to this are ØMQ contexts, which are threadsafe.
;* Stay away from the classic concurrency mechanisms like as mutexes, critical sections, semaphores, etc. These are an anti-pattern in ØMQ applications.
;* Create one ØMQ context at the start of your process, and pass that to all threads that you want to connect via inproc sockets.
;* Use attached threads to create structure within your application, and connect these to their parent threads using PAIR sockets over inproc. The pattern is: bind parent socket, then create child thread which connects its socket.
;* Use detached threads to simulate independent tasks, with their own contexts. Connect these over tcp. Later you can move these to stand-alone processes without changing the code significantly.
;* All interaction between threads happens as ØMQ messages, which you can define more or less formally.
;* Don't share ØMQ sockets between threads. ØMQ sockets are not threadsafe. Technically it's possible to migrate a socket from one thread to another but it demands skill. The only place where it's remotely sane to share sockets between threads are in language bindings that need to do magic like garbage collection on sockets.

* 스레드 내에 데이터를 개별적으로 고립시키고 스레드들 간에 공유하지 않습니다. 예외는 ØMQ 컨텍스트로 스레드 안전합니다.
* 전통적인 병렬 처리인 뮤텍스, 크리티컬 섹션, 세마포어 등을 멀리 하며, 이것은 ØMQ 응용프로그램에 부적절합니다.
* 프로세스 시작 시 단 하나의 ØMQ 컨텍스트를 생성하여 inproc(스레드 간 통신) 소켓을 통해 연결하려는 모든 스레드들에 전달합니다.
* 응용프로그램에서 구조체를 생성하기 위하여 할당된 스레드들(attached threads)을 사용하며, inproc상에서 페어 소켓을 사용하여 부모 스레드에 연결합니다. 이러한 패턴은 부모 소켓을 바인딩 한 다음에 부모 소켓을 연결하는 자식 스레드를 만듭니다.
* 독립적인 작업을 위하여 자체 컨텍스트를 가진 해제된 스레드(detached threads) 사용합니다. TCP상에서 연결하며 나중에 소스 코드의 큰 변경 없이 독립적인 프로세스들로 전환 가능합니다.
* 모든 스레드들 간의 정보 교환은 ØMQ 메시지로 이루어지며, 어느 정도 공식적으로 정의할 수 있습니다.
* 스레드들 간에 ØMQ 소켓을 공유하지 말아야 하며, ØMQ 소켓은 스레드 안전하지 않습니다. 소켓을 다른 스레드로 전환하는 것은 기술적으로는 가능하지만 숙련된 기술이 필요합니다. 여러 스레드에서 하나의 소켓을 취급하는 유일한 방법은 마법 같은 가비지 수집을 가진 언어 바인딩 정도입니다.

> [옮긴이] 스레드 안전은 스레드를 통한 동시성 프로그래밍 수행 시, 교착, 경쟁조건을 감지하여 회피할 수 있게 합니다.

;If you need to start more than one proxy in an application, for example, you will want to run each in their own thread. It is easy to make the error of creating the proxy frontend and backend sockets in one thread, and then passing the sockets to the proxy in another thread. This may appear to work at first but will fail randomly in real use. Remember: Do not use or close sockets except in the thread that created them.

응용프로그램에서 2개 이상의 프록시를 시작해야 하는 경우, 예를 들어 각 스레드에서 각각의 프록시를 실행하려고 하면, 한 스레드에서 프록시 프론트엔드 및 백엔드 소켓을 작성한 후 다른 스레드의 프록시로 소켓을 전달하게 되면 오류가 발생하기 쉽습니다. 이것은 처음에는 작동하는 것처럼 보이지만 실제 사용에서는 무작위로 실패합니다. 알아두기 : 소켓을 만든 스레드를 제외하고 소켓을 사용하거나 닫지 마십시오.

;If you follow these rules, you can quite easily build elegant multithreaded applications, and later split off threads into separate processes as you need to. Application logic can sit in threads, processes, or nodes: whatever your scale needs.

이러한 규칙을 따르면 우아한 멀티스레드 응용프로그램을 쉽게 구축할 수 있으며 나중에 필요에 따라 스레드를 별도의 프로세스로 분리할 수 있습니다. 응용프로그램 로직은 요구되는 규모에 따라, 스레드들, 프로세스들 혹은 노드들로 확장할 수 있습니다.

;ØMQ uses native OS threads rather than virtual "green" threads. The advantage is that you don't need to learn any new threading API, and that ØMQ threads map cleanly to your operating system. You can use standard tools like Intel's ThreadChecker to see what your application is doing. The disadvantages are that native threading APIs are not always portable, and that if you have a huge number of threads (in the thousands), some operating systems will get stressed.

ØMQ는 가상의 "그린"스레드 대신 기본 운영체제 스레드를 사용합니다. 장점은 새로운 스레드 API를 배울 필요가 없고 ØMQ 스레드가 운영체제에 매핑되어 사용 가능합니다. 인텔의 스레드 검사기와 같은 표준 도구를 사용하여 응용프로그램이 수행 중인 작업을 확인할 수 있습니다. 단점은 운영체제 기반 스레드 API가 항상 이식 가능한 것은 아니며 거대한 수의 스레드가 필요한 경우 일부 운영체제에서는 성능에 영향을 주게 됩니다.

;Let's see how this works in practice. We'll turn our old Hello World server into something more capable. The original server ran in a single thread. If the work per request is low, that's fine: one ØMQ thread can run at full speed on a CPU core, with no waits, doing an awful lot of work. But realistic servers have to do nontrivial work per request. A single core may not be enough when 10,000 clients hit the server all at once. So a realistic server will start multiple worker threads. It then accepts requests as fast as it can and distributes these to its worker threads. The worker threads grind through the work and eventually send their replies back.

실제 동작 방법으로 이전 Hello World 서버(hwserver)를 보다 유능한 것으로 변환하겠습니다. 원래 서버는 단일 스레드로 구동되었습니다. 요청당 작업이 적을 경우 문제 되지 않지만 : 하나의 ØMQ 스레드가 CPU 코어에서 최고 속도로 실행되며 대기 없이 많은 작업을 수행할 수 있습니다. 그러나 현실적인 서버는 요청마다 중요한 작업을 수행해야 합니다. 단일 CPU 코어로는 10,000개의 클라이언트의 요청을 하나의 스레드인 서버에서 처리하기에는 충분하지 않습니다. 그래서 현실적인 서버는 여러 작업자 스레드들을 구동시켜 10,000개의 클라이언트의 요청을 최대한 빨리 받아 작업자 스레드에 배포합니다. 작업자 스레드들은 작업을 분산 처리하여 결국 응답을 보냅니다.

;You can, of course, do all this using a proxy broker and external worker processes, but often it's easier to start one process that gobbles up sixteen cores than sixteen processes, each gobbling up one core. Further, running workers as threads will cut out a network hop, latency, and network traffic.

물론 브로커와 외부 작업자 프로세스를 사용하여 모든 작업을 수행할 수 있지만, 16개 프로세스로 16 코어 CPU를 소모하기보다는 하나의 프로세스에서 멀티스레드를 구동하는 것이 더욱 쉽습니다.
추가로 하나의 프로세스에서 작업자 스레드들을 실행하면 네트워크 홉, 지연시간, 트래픽을 줄일 수 있습니다.

;The MT version of the Hello World service basically collapses the broker and workers into a single process:

멀티스레드 버전의 Hello World 서비스는 기본적으로 하나의 프로세스 내에서 브로커와 작업자로 구성하게 합니다.

mtserver.c: 멀티스레드 서비스

```cpp
//  Multithreaded Hello World server

#include "zhelpers.h"
#include <pthread.h>
#ifndef _WIN32
#include <unistd.h>
#endif

static void *
worker_routine (void *context) {
    //  Socket to talk to dispatcher
    void *receiver = zmq_socket (context, ZMQ_REP);
    zmq_connect (receiver, "inproc://workers");

    while (1) {
        char *string = s_recv (receiver);
        printf ("Received request: [%s]\n", string);
        free (string);
        //  Do some 'work'
        s_sleep (1000);
        //  Send reply back to client
        s_send (receiver, "World");
    }
    zmq_close (receiver);
    return NULL;
}

int main (void)
{
    void *context = zmq_ctx_new ();

    //  Socket to talk to clients
    void *clients = zmq_socket (context, ZMQ_ROUTER);
    zmq_bind (clients, "tcp://*:5555");

    //  Socket to talk to workers
    void *workers = zmq_socket (context, ZMQ_DEALER);
    zmq_bind (workers, "inproc://workers");

    //  Launch pool of worker threads
    int thread_nbr;
    for (thread_nbr = 0; thread_nbr < 5; thread_nbr++) {
        pthread_t worker;
        pthread_create (&worker, NULL, worker_routine, context);
    }
    //  Connect work threads to client threads via a queue proxy
    zmq_proxy (clients, workers, NULL);

    //  We never get here, but clean up anyhow
    zmq_close (clients);
    zmq_close (workers);
    zmq_ctx_destroy (context);
    return 0;
}
```

그림 20 - 멀티스레드 서비스

![멀티스레드 서비스](images/fig20.svg)

> [옮긴이] 원도우 운영체제에서 POSIX 스레드(pthread) 사용하기 위해서는 [pthreads Win32](http://www.sourceware.org/pthreads-win32/)를 참조합니다.
응용프로그램 빌드 시 timespec 관련 오류(Error C2011 'timespec': 'struct' type redefinition) 발생할 경우 `pthread.h" 파일에 
`#define HAVE_STRUCT_TIMESPEC` 추가합니다.
사이트(ftp://sourceware.org/pub/pthreads-win32/dll-latest)에서 최신 파일(pthreadVC2.dll, pthreadVC2.lib, pthread.h, sched.h, semaphore.h)을 받아 설치합니다.

;All the code should be recognizable to you by now. How it works:

이제 코드를 보면 무엇을 하는지 아실 수 있으실 겁니다. 작동 방법 :

;* The server starts a set of worker threads. Each worker thread creates a REP socket and then processes requests on this socket. Worker threads are just like single-threaded servers. The only differences are the transport (inproc instead of tcp), and the bind-connect direction.
;* The server creates a ROUTER socket to talk to clients and binds this to its external interface (over tcp).
;* The server creates a DEALER socket to talk to the workers and binds this to its internal interface (over inproc).
;* The server starts a proxy that connects the two sockets. The proxy pulls incoming requests fairly from all clients, and distributes those out to workers. It also routes replies back to their origin.

* 서버(mtserver)는 일련의 작업자 스레드들(worker threads)를 시작합니다. 각 작업자 스레드는 REP 소켓을 만들어 연결하여 소켓의 요청을 처리합니다. 작업자 스레드는 단일 스레드 서버와 같습니다. 유일한 차이점은 전송방식(tcp 대신 inproc)과 바인드-연결(bind-connect) 방향입니다.
* 서버는 클라이언트(hwclient)와 통신하기 위해 ROUTER 소켓을 생성하고 외부 인터페이스(tcp를 통해)에 바인딩(bind)합니다.
* 서버는 작업자들과 통신하기 위해 DEALER 소켓을 생성하고 내부 인터페이스 (inproc 통해)에 바인딩(bind)합니다.
* 서버는 두 소켓(ROUTER-DEALER)을 연결하는 프록시(`zmq_proxy()`)를 시작합니다. 프록시는 모든 클라이언트에서 들어오는 요청을 작업자들에게 배포하며 작업자들의 응답을 클라이언트에 전달합니다.

> [옮긴이] 빌드 및 테스트

~~~{.bash}
PS D:\git_store\zguide-kr\examples\C> cl -EHsc mtserver.c libzmq.lib pthreadVC2.lib
Microsoft (R) C/C++ 최적화 컴파일러 버전 19.16.27035(x64)
Copyright (c) Microsoft Corporation. All rights reserved.
mtserver.c
Microsoft (R) Incremental Linker Version 14.16.27035.0
Copyright (C) Microsoft Corporation.  All rights reserved.
/out:mtserver.exe
mtserver.obj
libzmq.lib
pthreadVC2.lib

PS D:\git_store\zguide-kr\examples\C> ./mtserver
Received request: [Hello]
Received request: [Hello]
Received request: [Hello]
Received request: [Hello]

PS D:\git_store\zguide-kr\examples\C> ./hwclient
Connecting to hello world server...
Sending Hello 0...
Received World 0
Sending Hello 1...
Received World 1

PS D:\git_store\zguide-kr\examples\C> ./hwclient
Connecting to hello world server...
Sending Hello 0...
Received World 0
Sending Hello 1...
Received World 1
~~~

;Note that creating threads is not portable in most programming languages. The POSIX library is pthreads, but on Windows you have to use a different API. In our example, the pthread_create call starts up a new thread running the worker_routine function we defined. We'll see in Chapter 3 - Advanced Request-Reply Patterns how to wrap this in a portable API.

스레드들의 생성은 대부분 프로그래밍 언어에서 이식성이 없다는 점에 유의하십시오.
POSIX 라이브러리 pthreads이지만 원도우에서는 다른 API를 사용해야 됩니다.
예제에서는 `pthread_create()`를 호출하여 정의된 작업자의 기능을 수행하고 있습니다.
'3장-고급 요청-응답 패턴'에서 이식 가능한 고급 API(CZMQ)로 변환 방법을 살펴보겠습니다.

;Here the "work" is just a one-second pause. We could do anything in the workers, including talking to other nodes. This is what the MT server looks like in terms of ØMQ sockets and nodes. Note how the request-reply chain is REQ-ROUTER-queue-DEALER-REP.

여기서 "작업(작업자 스레드가 요청을 받아 응답)"은 1초 동안 대기(`s_sleep(1000)`)합니다. 우리는 다른 노드와 통신하는 것을 포함하여 작업자들에 대하여 무엇이든 할 수 있습니다. 이것은 멀티스레드 서버를 ØMQ 소켓과 노드 같이 보이게 하였습니다. 그리고 REQ-REP 통로는 REQ-ROUTER-queue-DEALER-REP로 사용되는 것을 확인하였습니다.

## 스레드들간의 신호(PAIR 소캣)
;When you start making multithreaded applications with ØMQ, you'll encounter the question of how to coordinate your threads. Though you might be tempted to insert "sleep" statements, or use multithreading techniques such as semaphores or mutexes, the only mechanism that you should use are ØMQ messages. Remember the story of The Drunkards and The Beer Bottle.

ØMQ로 멀티스레드 응용프로그램을 만들 때 스레드들 간 협업하는 방법에 대하여 궁금하실 겁니다. `sleep()` 함수를 삽입하거나 멀티스레딩 기술인 세마포어 혹은 뮤텍스와 같이 사용하고 싶더라도 ØMQ 메시지만 사용해야 합니다. 술꾼과 맥주병 이야기에서 언급한 것처럼 스레드들 간에 메시지 전달만 수행하고 공유하는 상태(& 자원)가 없어야 합니다.

;Let's make three threads that signal each other when they are ready. In this example, we use PAIR sockets over the inproc transport:

준비되셨다면 서로 신호를 보내는 3개의 스레드를 만들어 봅시다. 예제에서 inproc 전송방식로 PAIR 소켓을 사용합니다.

mtrelay.c: 멀티 스레드(MT) relay

```cpp
//  Multithreaded relay

#include "zhelpers.h"
#include <pthread.h>

static void *
step1 (void *context) {
    //  Connect to step2 and tell it we're ready
    void *xmitter = zmq_socket (context, ZMQ_PAIR);
    zmq_connect (xmitter, "inproc://step2");
    printf ("Step 1 ready, signaling step 2\n");
    s_send (xmitter, "READY");
    zmq_close (xmitter);

    return NULL;
}

static void *
step2 (void *context) {
    //  Bind inproc socket before starting step1
    void *receiver = zmq_socket (context, ZMQ_PAIR);
    zmq_bind (receiver, "inproc://step2");
    pthread_t thread;
    pthread_create (&thread, NULL, step1, context);

    //  Wait for signal and pass it on
    char *string = s_recv (receiver);
    printf ("Step 2 received sting from step 1 : %s\n", string);
    free (string);
    zmq_close (receiver);

    //  Connect to step3 and tell it we're ready
    void *xmitter = zmq_socket (context, ZMQ_PAIR);
    zmq_connect (xmitter, "inproc://step3");
    printf ("Step 2 ready, signaling step 3\n");
    s_send (xmitter, "READY");
    zmq_close (xmitter);

    return NULL;
}

int main (void)
{
    void *context = zmq_ctx_new ();

    //  Bind inproc socket before starting step2
    void *receiver = zmq_socket (context, ZMQ_PAIR);
    zmq_bind (receiver, "inproc://step3");
    pthread_t thread;
    pthread_create (&thread, NULL, step2, context);

    //  Wait for signal
    char *string = s_recv (receiver);
    printf ("Step 3 received sting from step 2 : %s\n", string);
    free (string);
    zmq_close (receiver);

    printf ("Test successful!\n");
    zmq_ctx_destroy (context);
    return 0;
}
```
그림 21 - 릴레이(Relay)

![Relay](images/fig21.svg)

;This is a classic pattern for multithreading with ØMQ:

이것은 ØMQ 멀티스레드을 위한 고전적인 패턴입니다.

;* Two threads communicate over inproc, using a shared context.
;* The parent thread creates one socket, binds it to an inproc: endpoint, and //then starts the child thread, passing the context to it.
;* The child thread creates the second socket, connects it to that inproc: endpoint, and //then signals to the parent thread that it's ready.

* 2개의 스레드는 공유 컨텍스트를 통하여 inproc로 통신합니다.
* 부모 스레드는 하나의 소켓을 만들어 `inproc://step3(혹은 step2)`에 바인딩한 다음, 자식 스레드를 시작하여 컨텍스트를 전달합니다.
* 자식 스레드는 두 번째 소켓을 만들어서 `inproc://step3(혹은 step2)`에 연결한 다음, 준비된 부모 스레드에 신호를 보냅니다.

;Note that multithreading code using this pattern is not scalable out to processes. If you use inproc and socket pairs, you are building a tightly-bound application, i.e., one where your threads are structurally interdependent. Do this when low latency is really vital. The other design pattern is a loosely bound application, where threads have their own context and communicate over ipc or tcp. You can easily break loosely bound threads into separate processes.

이 패턴(ZMQ_PAIR)을 사용하는 멀티스레드 코드는 프로세스에서는 사용할 수 없습니다. inproc 및 소켓 쌍(PAIR)을 사용하여 강하게 연결된 응용프로그램(스레드가 구조적으로 상호 의존적)을 작성할 수 있습니다. 낮은 지연 시간이 핵심적일 경우 사용하시기 바랍니다. 다른 디자인 패턴으로 느슨하게 연결된 응용프로그램의 경우, 스레드들이 자체 컨텍스트가 있고 ipc 또는 tcp를 통해 통신합니다. 느슨하게 연결된 스레드들은 독립적인 프로세스로 쉽게 분리 할 수 있습니다.

;This is the first time we've shown an example using PAIR sockets. Why use PAIR? Other socket combinations might seem to work, but they all have side effects that could interfere with signaling:

PAIR 소켓을 사용한 예제를 처음으로 보여 주었습니다. PAIR를 사용하는 이유는 다른 소켓 조합으로도 동작할 것 같지만 이러한 신호 인터페이스로 사용하면 부작용이 있습니다.

;* You can use PUSH for the sender and PULL for the receiver. This looks simple and will work, but remember that PUSH will distribute messages to all available receivers. If you by accident start two receivers (e.g., you already have one running and you start a second), you'll "lose" half of your signals. PAIR has the advantage of refusing more than one connection; the pair is exclusive.
;* You can use DEALER for the sender and ROUTER for the receiver. ROUTER, however, wraps your message in an "envelope", meaning your zero-size signal turns into a multipart message. If you don't care about the data and treat anything as a valid signal, and if you don't read more than once from the socket, that won't matter. If, however, you decide to send real data, you will suddenly find ROUTER providing you with "wrong" messages. DEALER also distributes outgoing messages, giving the same risk as PUSH.
;* You can use PUB for the sender and SUB for the receiver. This will correctly deliver your messages exactly as you sent them and PUB does not distribute as PUSH or DEALER do. However, you need to configure the subscriber with an empty subscription, which is annoying.

* 송신자는 PUSH를 사용하고 수신자 PULL을 사용할 수 있습니다. 이것은 단순하고 동작하지만 PUSH는 모든 사용 가능한 수신자에게 메시지를 분배합니다. 실수로 2개의 수신자를 시작한 경우(예 : 이미 첫 번째 실행 중이고 잠시 후(1초) 두 번째는 시작한 경우) 신호의 절반을 "손실"됩니다. PAIR는 하나 이상의 연결을 거부할 수 있는 장점이 있습니다. PAIR 독점적(exclusive)합니다.
* 송신자는 DEALER를, 수신자는 ROUTER를 사용할 수 있습니다. 그러나 ROUTER는 메시지를 "봉투"속에 넣으며, 0 크기 신호가 멀티파트 메시지(공백 구분자 + 데이터) 변환됩니다. 데이터에 신경 쓰지 않고 유효한 신호로 어떤 것도 처리하지 않고 소켓에서 한번 이상 읽지 않는다면 문제가 되지는 않습니다. 
그러나 실제 데이터를 전송하기로 결정하면 ROUTER가 갑자기 잘못된 메시지를 제공하는 것을 알게 됩니다. 또한 DEALER는 ROUTER에서 전달된 메시지를 배포하여 PUSH와 동일한 위험(실수로 2개의 수신자를 시작)이 있습니다.
* 송신자에 PUB를 사용하고 수신자에 SUB를 사용할 수 있습니다. 그러면 메시지를 보낸 그대로 정확하게 전달할 수 있으며 PUB는 PUSH 또는 DEALER처럼 배포하지 않습니다. 그러나 빈 구독으로 구독자로 설정해야 하나 귀찮은 일입니다.

> [옮긴이] 빈 구독(empty subscrition)은 구독자가 발행자가 전송하는 모든 메시지를 수신 가능하도록 필터를 두지 않도록 설정합니다.
- 설정 예시 : `zmq_setsockopt (subscriber, ZMQ_SUBSCRIBE, "", 0);`

;For these reasons, PAIR makes the best choice for coordination between pairs of threads.

이러한 이유로 PAIR 패턴은 스레드들 간의 협업을 위한 최상의 선택입니다.

## 노드 간의 협업(Node Coordination)
;When you want to coordinate a set of nodes on a network, PAIR sockets won't work well any more. This is one of the few areas where the strategies for threads and nodes are different. Principally, nodes come and go whereas threads are usually static. PAIR sockets do not automatically reconnect if the remote node goes away and comes back.

네트워크상에서 노드들 간의 협업하려 하면, PAIR 소켓을 적용할 수 없습니다.  이것은 스레드들과 노드들 간에 전략이 다른 몇 가지 영역 중 하나입니다. 주로 노드들은 왔다 갔다 하고 스레드들은 보통 정적입니다. PAIR 소켓은 원격 노드가 가고 다시 돌아오면 자동으로 재연결하지 않습니다.

그림 22 - 발행-구독 동기화

![Pub-Sub Synchronization](images/fig22.svg)

;The second significant difference between threads and nodes is that you typically have a fixed number of threads but a more variable number of nodes. Let's take one of our earlier scenarios (the weather server and clients) and use node coordination to ensure that subscribers don't lose data when starting up.

스레드들과 노드들 간의 두 번째 중요한 차이점은 일반적으로 스레드들의 수는 고정되어 있지만 노드들의 수는 가변적입니다. 이전 시나리오(날씨 서버 및 클라이언트)를 가져와서 노드 협업을 하여 구독자 시작 시 연결에 소요되는 시간으로 인하여 데이터가 유실하지 않도록 하겠습니다.

;This is how the application will work:

다음은 응용프로그램 동작 방식입니다.

;* The publisher knows in advance how many subscribers it expects. This is just a magic number it gets from somewhere.
;* The publisher starts up and waits for all subscribers to connect. This is the node coordination part. Each subscriber subscribes and then tells the publisher it's ready via another socket.
;* When the publisher has all subscribers connected, it starts to publish data.

* 발행자는 미리 예상되는 구독자들의 숫자를 알고 있으며 이것은 어딘가에서 얻을 수 있는 마법 숫자입니다.
* 발행자가 시작하면서 모든 구독자들이 연결될 때까지 기다립니다. 이것이 노드 협업 부분입니다. 각 구독자들이 구독하기 시작하면 다른 소켓으로 발행자에게 준비가 되었음을 알립니다.
* 발행자는 모든 구독자들이 연결되면 데이터 전송을 시작됩니다.

> [옮긴이] 마법 숫자(magic number)는 이름 없는 숫자 상수(unnamed numerical constants)로 상황에 따라 상수값이 변경(예 : 하드웨어 아키텍처에 따란 INT는 16 비트 혹은 32 비트로 표현)될 수 있을 때 지정합니다. 현실에서는 구독자들의 숫자가 실제 얼마인지 정확히 알기 어렵기 때문에 예제에서는 정해진 구독자의 숫자를 마법 숫자(magic number)로 지칭합니다.


;In this case, we'll use a REQ-REP socket flow to synchronize subscribers and publisher. Here is the publisher:

이 경우에 REP-REQ 소켓을 통하여 구독자들과 발행자 간에 동기화하도록 하겠습니다. 아래는 발행자의 소스 코드입니다.

syncpub.c: 동기화된 발행자

> [옮긴이] 원도우 환경에서 테스트 위해 `SUBSCRIBERS_EXPECTED`을 10에서 2로 변경

```cpp
//  Synchronized publisher

#include "zhelpers.h"
#define SUBSCRIBERS_EXPECTED  2  //  We wait for 2 subscribers 

int main (void)
{
    void *context = zmq_ctx_new ();

    //  Socket to talk to clients
    void *publisher = zmq_socket (context, ZMQ_PUB);

    int sndhwm = 1100000;
    zmq_setsockopt (publisher, ZMQ_SNDHWM, &sndhwm, sizeof (int));

    zmq_bind (publisher, "tcp://*:5561");

    //  Socket to receive signals
    void *syncservice = zmq_socket (context, ZMQ_REP);
    zmq_bind (syncservice, "tcp://*:5562");

    //  Get synchronization from subscribers
    printf ("Waiting for subscribers\n");
    int subscribers = 0;
    while (subscribers < SUBSCRIBERS_EXPECTED) {
        //  - wait for synchronization request
        char *string = s_recv (syncservice);
        free (string);
        //  - send synchronization reply
        s_send (syncservice, "");
        subscribers++;
    }
    //  Now broadcast exactly 1M updates followed by END
    printf ("Broadcasting messages\n");
    int update_nbr;
    for (update_nbr = 0; update_nbr < 1000000; update_nbr++)
        s_send (publisher, "Rhubarb");

    s_send (publisher, "END");

    zmq_close (publisher);
    zmq_close (syncservice);
    zmq_ctx_destroy (context);
    return 0;
}
```

;And here is the subscriber:

아래는 구독자 소스 코드입니다.

syncsub.c: 동기화된 구독자

```cpp
//  Synchronized subscriber

#include "zhelpers.h"
#ifndef _WIN32
#include <unistd.h>
#endif

int main (void)
{
    void *context = zmq_ctx_new ();

    //  First, connect our subscriber socket
    void *subscriber = zmq_socket (context, ZMQ_SUB);
    zmq_connect (subscriber, "tcp://localhost:5561");
    zmq_setsockopt (subscriber, ZMQ_SUBSCRIBE, "", 0);

    //  ØMQ is so fast, we need to wait a while...
    s_sleep (1000);

    //  Second, synchronize with publisher
    void *syncclient = zmq_socket (context, ZMQ_REQ);
    zmq_connect (syncclient, "tcp://localhost:5562");

    //  - send a synchronization request
    s_send (syncclient, "");

    //  - wait for synchronization reply
    char *string = s_recv (syncclient);
    free (string);

    //  Third, get our updates and report how many we got
    int update_nbr = 0;
    while (1) {
        char *string = s_recv (subscriber);
        if (strcmp (string, "END") == 0) {
            free (string);
            break;
        }
        free (string);
        update_nbr++;
    }
    printf ("Received %d updates\n", update_nbr);

    zmq_close (subscriber);
    zmq_close (syncclient);
    zmq_ctx_destroy (context);
    return 0;
}
```

;This Bash shell script will start ten subscribers and then the publisher:

아래 배시 쉴 스크립터는 10개의 먼저 구독자들을 구동하고 발행자를 구동합니다. 

~~~ {.bash}
echo "Starting subscribers..."
for ((a=0; a<10; a++)); do
    syncsub &
done
echo "Starting publisher..."
syncpub
~~~

;Which gives us this satisfying output:

만족스러운 결과를 보여 줍니다.

~~~
Starting subscribers...
Starting publisher...
Received 1000000 updates
Received 1000000 updates
...
Received 1000000 updates
Received 1000000 updates
~~~

> [옮긴이] 빌드 및 테스트

~~~{.bash}
PS D:\git_store\zguide-kr\examples\C> cl -EHsc syncpub.c libzmq.lib
PS D:\git_store\zguide-kr\examples\C> cl -EHsc syncsub.c libzmq.lib

PS D:\git_store\zguide-kr\examples\C> ./syncpub
Waiting for subscribers
Broadcasting messages

PS D:\git_store\zguide-kr\examples\C> ./syncsub
Received 1000000 updates

PS D:\git_store\zguide-kr\examples\C> ./syncsub
Received 1000000 updates
~~~
;We can't assume that the SUB connect will be finished by the time the REQ/REP dialog is complete. There are no guarantees that outbound connects will finish in any order whatsoever, if you're using any transport except inproc. So, the example does a brute force sleep of one second between subscribing, and sending the REQ/REP synchronization.

'syncsub'에서 REQ/REP 통신가 완료될 때까지 SUB 연결이 완료되지 않는다고 하면 inproc 이외의 전송을 사용하는 경우 아웃바운드 연결이 어떤 순서로든 완료된다는 보장은 없습니다. 따라서 예제에서는 `syncsub'에서 SUB 소켓 연결하고 REQ/REP 동기화 전송 사이에 1초의 대기를 수행하였습니다.

;A more robust model could be:

보다 강력한 모델은 다음과 같습니다.

;* Publisher opens PUB socket and starts sending "Hello" messages (not data).
;* Subscribers connect SUB socket and when they receive a Hello message they tell the publisher via a REQ/REP socket pair.
;* When the publisher has had all the necessary confirmations, it starts to send real data.

* 발행자가 PUB 소켓을 열고 `Hello`메시지(데이터 아님)를 보내기 시작합니다.
* 구독자는 SUB 소켓을 연결하고 `Hello` 메시지를 받으면 REQ/REP 소켓 쌍을 통해 발행자에게 알려줍니다.
* 발행자가 모든 필요한 확인 완료하며, 실제 데이터를 보내기 시작합니다.

## 제로 복사(Zero-Copy)
;ØMQ's message API lets you can send and receive messages directly from and to application buffers without copying data. We call this zero-copy, and it can improve performance in some applications.

ØMQ 메시지 API는 응용프로그램 버퍼의 데이터를 복사하지 않고 직접 메시지를 송/수신하게 합니다. 우리는 이것을 제로 복사라고 부르며 일부 응용프로그램의 성능을 향상할 수 있습니다.

;You should think about using zero-copy in the specific case where you are sending large blocks of memory (thousands of bytes), at a high frequency. For short messages, or for lower message rates, using zero-copy will make your code messier and more complex with no measurable benefit. Like all optimizations, use this when you know it helps, and measure before and after.

제로 복사는 높은 빈도로 큰 메모리 공간(수천 바이트) 전송하는 특정한 경우 사용할 수 있습니다. 짧은 메시지 또는 낮은 빈도에서 제로 복사를 사용하면 별다른 효과 없이 코드를 더 복잡하고 엉망으로 만들 수 있습니다. 모든 최적화처럼 도움이 될 거라면 사용하고 전/후의 성능을 측정하십시오.

;To do zero-copy, you use zmq_msg_init_data() to create a message that refers to a block of data already allocated with malloc() or some other allocator, and then you pass that to zmq_msg_send(). When you create the message, you also pass a function that ØMQ will call to free the block of data, when it has finished sending the message. This is the simplest example, assuming buffer is a block of 1,000 bytes allocated on the heap:

제로 복사(zero-copy)를 수행하려면 `zmq_msg_init_data()`를 사용하여 `malloc()` 또는 다른 할당자에 이미 할당된 데이터 블록을 참조하는 메시지를 생성하여 `zmq_msg_send()`을 통해 전달합니다. 메시지를 생성할 때, ØMQ는 메시지 전송이 완료되면 데이터 블록을 해제하기 위해 호출하는 기능도 전달합니다. 단순한 예로, 힙에 할당된 버퍼가 1,000 바이트로 가정합니다.
```cpp
void my_free (void *data, void *hint) {
    free (data);
}
// Send message from buffer, which we allocate and ØMQ will free for us
zmq_msg_t message;
zmq_msg_init_data (&message, buffer, 1000, my_free, NULL);
zmq_msg_send (&message, socket, 0);
```

;Note that you don't call zmq_msg_close() after sending a message—libzmq will do this automatically when it's actually done sending the message.

`zmq_msg_send()`로 메시지를 송신하고 `zmq_msg_close()`를 호출하지 않는 점에 주의하십시오.
`libzmq` 라이브러리는 실제 메시지를 송신하고 자동으로 작업(메모리 해제)을 수행합니다.

;There is no way to do zero-copy on receive: ØMQ delivers you a buffer that you can store as long as you wish, but it will not write data directly into application buffers.

수신에는 제로 복사를 수행할 방법은 없습니다 : ØMQ는 원하는 기간 동안 저장할 수 있는 버퍼를 제공하지만 데이터를 응용프로그램 버퍼에 직접 쓰지는 않습니다.

;On writing, ØMQ's multipart messages work nicely together with zero-copy. In traditional messaging, you need to marshal different buffers together into one buffer that you can send. That means copying data. With ØMQ, you can send multiple buffers coming from different sources as individual message frames. Send each field as a length-delimited frame. To the application, it looks like a series of send and receive calls. But internally, the multiple parts get written to the network and read back with single system calls, so it's very efficient.

데이터를 쓸 때, ØMQ의 멀티파트 메시지는 제로 복사와 함께 잘 작동합니다. 전통적인 메시징에서는 전송할 하나의 버퍼와 함께 다른 버퍼 마살링해야 합니다. 이것은 데이터 복사를 의미합니다. ØMQ에서는 서로 다른 소스에서 여러 개의 버퍼를 개별 메시지 프레임으로 보낼 수 있습니다. 각 필드를 길이가 구분된 프레임으로 보냅니다. 응용프로그램에서는 일련의 송/수신 호출처럼 보입니다. 그러나 내부적으로 다중 부분이 네트워크에 쓰이고 단일 시스템 호출로 다시 읽히므로 매우 효율적입니다.

> [옮긴이] 마살링(marshalling or marshaling)은 객체의 메모리상 표현을 다른 데이터 형태로 변경하는 절차입니다. 컴퓨터상에서 응용프로그램들 간의 데이터 통신 수행시 서로 다른 포맷에 대하여 전환하는 작업이 필요하며, 직렬화(Serialization)과 유사합니다.

## 발행-구독 메시지 봉투(Message Envelope)
;In the pub-sub pattern, we can split the key into a separate message frame that we call an envelope. If you want to use pub-sub envelopes, make them yourself. It's optional, and in previous pub-sub examples we didn't do this. Using a pub-sub envelope is a little more work for simple cases, but it's cleaner especially for real cases, where the key and the data are naturally separate things.

발행-구독 패턴에서, 키를 개별 메시지 프레임으로 분할하고 봉투라고 했습니다. 발행-구독 봉투를 사용하기 위해서는 직접 만들어야 합니다. 선택 사항으로 이전 발행-구독 예제에서는 이런 작업을 수행하지 않았습니다. 
발행-구독 봉투를 이용하려면 약간의 코드를 추가해야 합니다.
키와 데이터를 별도의 메시지 프레임에 나누어서 있어 실제 코드를 보면 바로 이해할 수 있습니다. 

그림 23 - 개별 키를 가진 발행-구독 봉투 

![Pub-Sub Envelope with Separate Key](images/fig23.svg)

;Recall that subscriptions do a prefix match. That is, they look for "all messages starting with XYZ". The obvious question is: how to delimit keys from data so that the prefix match doesn't accidentally match data. The best answer is to use an envelope because the match won't cross a frame boundary. Here is a minimalist example of how pub-sub envelopes look in code. This publisher sends messages of two types, A and B.

구독자에서 구독시 필터를 사용하여 접두사 일치 수행하는 것을 기억하십시오. 즉, "XYZ로 시작하는 모든 메시지"를 찾아야 합니다. 분명한 질문으로 실수로 접두사 일치가 데이터와 일치하지 않도록 데이터에서 키를 구분하는 방법입니다. 가장 좋은 방법은 봉투를 사용하여 프레임 경계를 넘지 않으므로 일치 여부 확인하는 것입니다. 최소 예제로 발행-구독 봉투가 코드로 구현하였으며 발행자는 A와 B의 두 가지 유형의 메시지를 보냅니다.


;The envelope holds the message type:

봉투에는 메시지 유형이 있습니다.

psenvpub.c: Pub-Sub 봉투 발행자(enveloper publisher)

```cpp
//  Pubsub envelope publisher
//  Note that the zhelpers.h file also provides s_sendmore

#include "zhelpers.h"
#ifndef _WIN32
#include <unistd.h>
#endif

int main (void)
{
    //  Prepare our context and publisher
    void *context = zmq_ctx_new ();
    void *publisher = zmq_socket (context, ZMQ_PUB);
    zmq_bind (publisher, "tcp://*:5563");

    while (1) {
        //  Write two messages, each with an envelope and content
        s_sendmore (publisher, "A");
        s_send (publisher, "We don't want to see this");
        s_sendmore (publisher, "B");
        s_send (publisher, "We would like to see this");
        s_sleep (1000);
    }
    //  We never get here, but clean up anyhow
    zmq_close (publisher);
    zmq_ctx_destroy (context);
    return 0;
}
```

;The subscriber wants only messages of type B:

구독자(subscriber)는 B 유형의 메시지만 원합니다.

psenvsub.c: Pub-Sub 봉투 구독자(enveloper subscriber)

```cpp
//  Pubsub envelope subscriber

#include "zhelpers.h"

int main (void)
{
    //  Prepare our context and subscriber
    void *context = zmq_ctx_new ();
    void *subscriber = zmq_socket (context, ZMQ_SUB);
    zmq_connect (subscriber, "tcp://localhost:5563");
    zmq_setsockopt (subscriber, ZMQ_SUBSCRIBE, "B", 1);

    while (1) {
        //  Read envelope with address
        char *address = s_recv (subscriber);
        //  Read message contents
        char *contents = s_recv (subscriber);
        printf ("[%s] %s\n", address, contents);
        free (address);
        free (contents);
    }
    //  We never get here, but clean up anyhow
    zmq_close (subscriber);
    zmq_ctx_destroy (context);
    return 0;
}
```

;When you run the two programs, the subscriber should show you this:

2개의 프로그램을 실행하면 구독자(subscriber)는 아래와 같이 보입니다.
~~~
[B] We would like to see this
[B] We would like to see this
[B] We would like to see this
...
~~~

> [옮긴이] 빌드 및 테스트

~~~{.bash}
PS D:\git_store\zguide-kr\examples\C> cl -EHsc psenvpub.c libzmq.lib
PS D:\git_store\zguide-kr\examples\C> cl -EHsc psenvsub.c libzmq.lib

PS D:\git_store\zguide-kr\examples\C> ./psenvpub

PS D:\git_store\zguide-kr\examples\C> ./psenvsub
[B] We would like to see this
[B] We would like to see this
[B] We would like to see this
...
~~~

> [옮긴이] 다중 필터(multiple filters)를 아래와 같이 수행하면 "A', "B" 2개 중에 한 개만 있어도 수신 가능함.

```cpp
    zmq_setsockopt (subscriber, ZMQ_SUBSCRIBE, "B", 1);
    zmq_setsockopt (subscriber, ZMQ_SUBSCRIBE, "A", 1);
```

;This example shows that the subscription filter rejects or accepts the entire multipart message (key plus data). You won't get part of a multipart message, ever. If you subscribe to multiple publishers and you want to know their address so that you can send them data via another socket (and this is a typical use case), create a three-part message.

예제에서 구독 필터가 전체 멀티 파트 메시지(키와 데이터)를 거부하거나 수락함을 보여줍니다. 멀티파트 메시지의 일부만 받지 않습니다. 여러 발행자들을 구독하고 다른 소켓을 통해 데이터를 보낼 수 있도록 주소를 알고 싶다면 (이것은 일반적인 사용 사례임) ㅅ3 부분으로 된 메시지(key-address-data)를 만드십시오.

그림 24 -  서버 주소(Address) 가진 발행-구독 봉투

![Pub-Sub Envelope with Sender Address](images/fig24.svg)

## 최고 수위선(High-Water Marks)
;When you can send messages rapidly from process to process, you soon discover that memory is a precious resource, and one that can be trivially filled up. A few seconds of delay somewhere in a process can turn into a backlog that blows up a server unless you understand the problem and take precautions.

프로세스에서 프로세스로 메시지를 빠르게 보낼 때에 곧 메모리가 쉽게 채워질 수 있다는 것을 알게 됩니다. 프로그램 상의 문제에 대하여 이해하고 예방 조치를 하지 않아 메세지 수신 프로세스의 어딘가에서 몇 초의 지연이 발생하면 메시지 데이터는 메모리 상의 백로그 전환되어 서버의 자원을 고갈시킬 수 있습니다.

;The problem is this: imagine you have process A sending messages at high frequency to process B, which is processing them. Suddenly B gets very busy (garbage collection, CPU overload, whatever), and can't process the messages for a short period. It could be a few seconds for some heavy garbage collection, or it could be much longer, if there's a more serious problem. What happens to the messages that process A is still trying to send frantically? Some will sit in B's network buffers. Some will sit on the Ethernet wire itself. Some will sit in A's network buffers. And the rest will accumulate in A's memory, as rapidly as the application behind A sends them. If you don't take some precaution, A can easily run out of memory and crash.

문제는 다음과 같습니다 : 프로세스 A가 높은 빈도로 프로세스 B로 메시지를 전송하고 프로세스 B에서 메시지를 처리하고 있다고 가정하면, 갑자기 B 프로세스가 바쁜 상태(가비지 수집, CPU 과부하 등)가 되어 잠시 메시지를 처리할 수 없습니다. 많은 가비지 수집(역주 : java 가상 머신과 같이)의 경우 몇 초가 소요되거나 더 심각한 문제가 있는 경우 처리 시간이 지연될 수 있습니다. 
프로세스 A가 여전히 미친 듯이 메시지를 보내면 메시지는 어떻게 될까요? 
일부는 B의 네트워크 버퍼(buffer)에 있습니다. 
일부는 이더넷 네트워크 자체에 있으며 
일부는 프로세스 A의 네트워크 버퍼에 있습니다. 
나머지는 나중에 신속하게 재전송할 수 있도록 프로세스 A의 메모리에 남아 있습니다.
사전 예방을 하지 않을 경우 프로세스 A는 메모리가 고갈되어 오류가 발생할 수 있습니다.

;It is a consistent, classic problem with message brokers. What makes it hurt more is that it's B's fault, superficially, and B is typically a user-written application which A has no control over.

이것은 메시지 브로커의 일관되고 고전적인 문제입니다. 더 아프게 만드는 것은 프로세스 B 오류이며 피상적이며 B는 일반적으로 A가 제어할 수 없는 사용자 작성 응용프로그램입니다.

;What are the answers? One is to pass the problem upstream. A is getting the messages from somewhere else. So tell that process, "Stop!" And so on. This is called flow control. It sounds plausible, but what if you're sending out a Twitter feed? Do you tell the whole world to stop tweeting while B gets its act together?

해결 방법의 하나는 문제를 메시지를 계속 전달하는 상류에 알리는 것입니다. 프로세스 A는 다른 곳에서 메시지를 받고 있으며 "Stop!"이라고 전달됩니다. 이것을 흐름 제어라고 합니다. 타당해 보이지만 트위터 피드에 "Stop!"을 보내면 어떨까요? B 정상화될 때까지 전 세계에서 트윗을 중단하라고 지시합니까?

;Flow control works in some cases, but not in others. The transport layer can't tell the application layer to "stop" any more than a subway system can tell a large business, "please keep your staff at work for another half an hour. I'm too busy". The answer for messaging is to set limits on the size of buffers, and then when we reach those limits, to take some sensible action. In some cases (not for a subway system, though), the answer is to throw away messages. In others, the best strategy is to wait.

흐름 제어는 경우에 따라 작동하거나 하지 않을 수 있습니다. 전송계층은 응용프로그램 계층에 "stop"하도록 지시할 수 없습니다. 마치 지하철 시스템이 연계된 많은 계열사에게 "너무 바쁘니 직원을 30분 더 근무하도록 하십시오"하는 것처럼 할 수는 없습니다.
메시징의 해결책은 버퍼 크기에 제한을 두어 전송되는 메시지가 버퍼 제한에 도달하면 적절한 조치를 취하는 것입니다. 경우에 따라 메시지를 버리는 것이 해결책이 될 수도 있으며 다른 경우는 대기하는 것이 최선의 전략입니다.

;ØMQ uses the concept of HWM (high-water mark) to define the capacity of its internal pipes. Each connection out of a socket or into a socket has its own pipe, and HWM for sending, and/or receiving, depending on the socket type. Some sockets (PUB, PUSH) only have send buffers. Some (SUB, PULL, REQ, REP) only have receive buffers. Some (DEALER, ROUTER, PAIR) have both send and receive buffers.

ØMQ는 HWM(최고수위 표시) 개념을 사용하여 내부 파이프의 용량을 정의합니다. 내/외부 소켓에 대한 각 연결에서 소켓 유형에 따라 자체 파이프 및 송/수신을 위한 HWM이 있습니다. 일부 소켓(PUB, PUSH)은 송신 버퍼만 있으며 일부 소켓(SUB, PULL, REQ, REP)은 수신 버퍼만 있습니다. 일부 소켓(DEALER, ROUTER, PAIR)은 송신 및 수신 버퍼가 모두 있습니다.

;In ØMQ v2.x, the HWM was infinite by default. This was easy but also typically fatal for high-volume publishers. In ØMQ v3.x, it's set to 1,000 by default, which is more sensible. If you're still using ØMQ v2.x, you should always set a HWM on your sockets, be it 1,000 to match ØMQ v3.x or another figure that takes into account your message sizes and expected subscriber performance.

ØMQ v2.x에서는 HWM은 기본적으로 제한이 없어 대량 데이터를 발행하는 발행자에는 치명적인 문제였습니다(옮긴이 : PUB 소켓 사용으로 메모리 버퍼 초과). ØMQ v3.x부터는 기본적으로 HWM을 1,000으로 설정되어 있습니다. 여전히 ØMQ v2.x를 사용하는 경우 메시지 크기 및 예상 구독자 성능을 고려하여 ØMQ v3.x과 일치하도록 HWM을 1,000으로 설정하거나 다른 수치로 조정해야 합니다. 

;When your socket reaches its HWM, it will either block or drop data depending on the socket type. PUB and ROUTER sockets will drop data if they reach their HWM, while other socket types will block. Over the inproc transport, the sender and receiver share the same buffers, so the real HWM is the sum of the HWM set by both sides.

소켓이 HWM에 도달하면 소켓 유형에 따라 데이터를 차단하거나 삭제합니다. PUB 및 ROUTER 소켓은 HWM에 도달하면 데이터를 삭제하지만 다른 소켓 유형은 차단합니다. inproc 전송방식에서 송신자와 수신자는 동일한 버퍼를 공유하므로 실제 HWM은 양쪽에서 설정한 HWM의 합입니다.

;Lastly, the HWMs are not exact; while you may get up to 1,000 messages by default, the real buffer size may be much lower (as little as half), due to the way libzmq implements its queues.

마지막으로, HWM은 정확하지 않습니다. 기본적으로 최대 1,000개의 메시지를 받을 수 있지만 libzmq가 대기열을 구현하는 방식으로 인해 실제 버퍼 크기가 훨씬 작을(절반까지) 수 있습니다.

## 메시지 분실 문제 해결 방법
;As you build applications with ØMQ, you will come across this problem more than once: losing messages that you expect to receive. We have put together a diagram that walks through the most common causes for this.

ØMQ를 사용하여 응용프로그램을 구축하면 "수신할 메시지가 손실되었습니다"와 같은 문제가 한번 이상 발생합니다 :  가장 일반적인 원인을 설명하는 다이어그램을 작성했습니다.

그림 25 - 메시지 분실 문제 해결 방법

![Missing Message Problem Solver](images/fig25.svg)

;Here's a summary of what the graphic says:

도표가 이야기하는 바는 다음과 같습니다.

;* On SUB sockets, set a subscription using zmq_setsockopt() with ZMQ_SUBSCRIBE, or you won't get messages. Because you subscribe to messages by prefix, if you subscribe to "" (an empty subscription), you will get everything.
;* If you start the SUB socket (i.e., establish a connection to a PUB socket) after the PUB socket has started sending out data, you will lose whatever it published before the connection was made. If this is a problem, set up your architecture so the SUB socket starts first, then the PUB socket starts publishing.
;* Even if you synchronize a SUB and PUB socket, you may still lose messages. It's due to the fact that internal queues aren't created until a connection is actually created. If you can switch the bind/connect direction so the SUB socket binds, and the PUB socket connects, you may find it works more as you'd expect.
;* If you're using REP and REQ sockets, and you're not sticking to the synchronous send/recv/send/recv order, ØMQ will report errors, which you might ignore. Then, it would look like you're losing messages. If you use REQ or REP, stick to the send/recv order, and always, in real code, check for errors on ØMQ calls.
;* If you're using PUSH sockets, you'll find that the first PULL socket to connect will grab an unfair share of messages. The accurate rotation of messages only happens when all PULL sockets are successfully connected, which can take some milliseconds. As an alternative to PUSH/PULL, for lower data rates, consider using ROUTER/DEALER and the load balancing pattern.
;* If you're sharing sockets across threads, don't. It will lead to random weirdness, and crashes.
;* If you're using inproc, make sure both sockets are in the same context. Otherwise the connecting side will in fact fail. Also, bind first, then connect. inproc is not a disconnected transport like tcp.
;* If you're using ROUTER sockets, it's remarkably easy to lose messages by accident, by sending malformed identity frames (or forgetting to send an identity frame). In general setting the ZMQ_ROUTER_MANDATORY option on ROUTER sockets is a good idea, but do also check the return code on every send call.

* SUB 소켓에서 `ZMQ_SUBSCRIBE`로 `zmq_setsockopt()`를 사용하여 구독을 설정하십시오. 그렇지 않으면 메시지를 받을 수 없습니다. 접두사로 메시지를 구독하기 때문에 필터에 ""(빈 구독)를 구독하면 모든 메시지를 받을 수 있습니다.
* PUB 소켓이 데이터 전송을 시작한 후 SUB 소켓을 시작하면(즉, PUB 소켓에 연결 설정) SUB 소켓 연결 전의 발행된 데이터를 잃게 됩니다. 이것이 문제이면 아키텍처를 수정하여 SUB 소켓이 먼저 시작한 후에 PUB 소켓이 발행하도록 하십시오
* SUB 및 PUB 소켓을 동기화하더라도 메시지가 손실될 수 있습니다. 이것은 실제로 연결이 생성될 때까지 내부 대기열가 생성되지 않기 때문입니다. 바인드/연결 방향을 바꾸어, SUB 소켓이 바인딩하고 PUB 소켓이 연결되면 기대한 것처럼 동작할 수 있습니다.
* REP 및 REQ 소켓을 사용하고 동기화된 송신/수신/송신/수신 순서를 지키지 않으면, ØMQ가 오류를 보고하지만 무시할 경우가 있습니다. 그러면 메시지를 잃어버린 것과 같습니다. REQ 또는 REP를 사용하는 경우 송신/수신 순서를 지켜야 하며 항상 실제 코드에서는 ØMQ 호출에서 오류가 있는지 확인하십시오.
* PUSH 소켓을 사용한다면, 연결할 첫 번째 PULL 소켓은 불공정하게 분배된 메시지들을 가지게 됩니다. 메시지의 정확한 순환은 모든 PULL 소켓이 성공적으로 연결된 경우에 발생하며 다소(몇 밀리초) 시간이 걸립니다. PUSH/PULL 소켓의 대안으로 낮은 데이터 빈도를 위해서는 ROUTER/DEALER 및 부하 분산 패턴을 고려하시기 바랍니다.
* 스레드 간에 소켓을 공유하지 마십시오. 무작위적인 이상 현상을 초래하고 충돌합니다.
* inproc을 사용하는 경우 두 소켓이 동일한 컨텍스트에 있는지 확인하십시오. 그렇지 않으면 연결 측이 실제로 실패합니다. 또한 먼저 바인딩한 다음 연결하십시오. inproc은 tcp와 같은 연결이 끊긴 전송방식이 아닙니다.
* ROUTER 소켓을 사용하는 경우, 우연히 잘못된 인식 프레임을 보내어(또는 인식 프레임을 보내지 않음) 메시지를 잃어버리기가 쉽습니다. 일반적으로 ROUTER 소켓에서 `ZMQ_ROUTER_MANDATORY` 옵션을 설정하는 것이 좋지만 모든 송신 호출에서 반환값을 확인해야 합니다.
* 마지막으로, 무엇이 잘못되었는지 알 수 없다면 문제를 재현하는 최소한의 테스트 사례를 만들어 문제가 발생하는지 테스트 수행하시기 바라며 ØMQ 커뮤니티에 도움을 요청하십시오.

> [옮긴이] `zmq_setsockopt (subscriber, ZMQ_SUBSCRIBE, "", 0)` 설정 시 모든 메시지 구독합니다.