;# Chapter 5 - Advanced Pub-Sub Patterns

# 5장 - 고급 발행-구독 패턴(Advanced Pub Sub Patterns)

;In Chapter 3 - Advanced Request-Reply Patterns and Chapter 4 - Reliable Request-Reply Patterns we looked at advanced use of ØMQ's request-reply pattern. If you managed to digest all that, congratulations. In this chapter we'll focus on publish-subscribe and extend ØMQ's core pub-sub pattern with higher-level patterns for performance, reliability, state distribution, and monitoring.

"3장 - 고급 요청-응답 패턴" 및 "4장 - 신뢰할 수 있는 요청-응답 패턴"에서 ØMQ의 요청-응답 패턴의 고급 사용을 보았습니다. 
그 모든 것을 이해하셨다면 축하드리며, 
이 장에서는 발행-구독(publish-subscribe)에 중점을 두고, ØMQ의 핵심인 발행-구독 패턴을 성능, 안정성, 변경정보 배포 및 모니터링을 위한 상위 수준 패턴으로 확장합니다.

;We'll cover:

다루는 내용은 다음과 같습니다.

;* When to use publish-subscribe
;* How to handle too-slow subscribers (the Suicidal Snail pattern)
;* How to design high-speed subscribers (the Black Box pattern)
;* How to monitor a pub-sub network (the Espresso pattern)
;* How to build a shared key-value store (the Clone pattern)
;* How to use reactors to simplify complex servers
;* How to use the Binary Star pattern to add failover to a server

* 발행-구독을 사용해야 할 때
* 너무 느린 구독자를 처리하는 방법(자살하는 달팽이 패턴)
* 고속 구독자 설계 방법(블랙박스 패턴)
* 발행-구독 네트워크를 모니터링하는 방법(에스프레소(Espresso) 패턴)
* 공유 카-값 저장소를 만드는 방법(복제 패턴)
* 리엑터를 사용하여 복잡한 서버를 단순화하는 방법
* 바이너리 스타 패턴을 사용하여 서버에 장애조치를 추가하는 방법

;## Pros and Cons of Pub-Sub

## 발행-구독의 장점과 단점

;ØMQ's low-level patterns have their different characters. Pub-sub addresses an old messaging problem, which is multicast or group messaging. It has that unique mix of meticulous simplicity and brutal indifference that characterizes ØMQ. It's worth understanding the trade-offs that pub-sub makes, how these benefit us, and how we can work around them if needed.

ØMQ의 저수준 패턴은 서로 다른 특성을 가지고 있습니다. 발행-구독은 멀티캐스트 혹은 그룹 메시징과 같은 오래된 메시징 문제를 해결합니다. 그것은 ØMQ를 특징짓는 꼼꼼한 단순성과 잔인한 무관심의 고유한 혼합을 가지고 있습니다. 발행-구독이 만드는 장단점이 어떻게 도움이 되는지, 필요한 경우 어떻게 사용할지 이해할 가치가 있습니다.

;First, PUB sends each message to "all of many", whereas PUSH and DEALER rotate messages to "one of many". You cannot simply replace PUSH with PUB or vice versa and hope that things will work. This bears repeating because people seem to quite often suggest doing this.

첫째, PUB 소켓은 각 메시지를 "all of many"로 보내는 반면 PUSH 및 DEALER 소켓은 메시지를 "one of many"로 수신자들에게 순차적으로 전달합니다. 단순히 PUSH를 PUB로 바꾸거나 역으로 하더라도 모든 것이 동작하기를 바랄 수는 없습니다. 사람들이 바꾸어 사용(PUSH, PUB)하는 것을 자주 제안하기 때문에 문제는 반복됩니다.

;More profoundly, pub-sub is aimed at scalability. This means large volumes of data, sent rapidly to many recipients. If you need millions of messages per second sent to thousands of points, you'll appreciate pub-sub a lot more than if you need a few messages a second sent to a handful of recipients.

좀 더 깊게 생각하면 발행-구독은 확장성을 목표로 합니다. 
이는 많은 수신자들에게 빠르게 대량의 데이터를 송신하는 것입니다. 초당 수백만 개의 메시지를 수천 개의 단말로 송신해야 하는 경우, 소수의 수신자들에게 초당 몇 개의 메시지를 보내는 경우보다 발행-구독의 기능에 고마워할 것입니다.

;To get scalability, pub-sub uses the same trick as push-pull, which is to get rid of back-chatter. This means that recipients don't talk back to senders. There are some exceptions, e.g., SUB sockets will send subscriptions to PUB sockets, but it's anonymous and infrequent.

확장성을 얻기 위해 발행-구독(pub-sub)은 푸시-풀(push-pull)과 동일하게 백-채터(back-chatter)를 제거합니다. 백-채터는 수신자가 발신자에게 응답하지 않음을 의미합니다. 일부 예외적인 경우로, SUB 소켓은 PUB 소켓에 구독을 보내지만 익명이며 드물게 일어납니다.

;Killing back-chatter is essential to real scalability. With pub-sub, it's how the pattern can map cleanly to the PGM multicast protocol, which is handled by the network switch. In other words, subscribers don't connect to the publisher at all, they connect to a multicast group on the switch, to which the publisher sends its messages.

백-채터를 제거하는 것은 실제 확장성에 필수적입니다. 발행-구독 패턴은  네트워크 스위치에 의해 처리되는 PGM 멀티캐스트 통신규약에 말끔하게 매핑될 수 있습니다. 
즉, 구독자는 발행자에 연결하지 않고, 발행자가 메시지를 보내는 네트워크 스위치 상의 멀티캐스트 그룹에 연결합니다.

;When we remove back-chatter, our overall message flow becomes much simpler, which lets us make simpler APIs, simpler protocols, and in general reach many more people. But we also remove any possibility to coordinate senders and receivers. What this means is:

백-채터를 제거하면, 전체 메시지 흐름은 훨씬 단순해져, 많은 사람들이 단순한 API와 단순한 통신규약을 사용할 수 있습니다.
그러나 백-채터의 제거는 송신자들과 수신자들를 조정할 수 없게 되며 의미하는 바는 다음과 같습니다.

;* Publishers can't tell when subscribers are successfully connected, both on initial connections, and on reconnections after network failures.
;* Subscribers can't tell publishers anything that would allow publishers to control the rate of messages they send. Publishers only have one setting, which is full-speed, and subscribers must either keep up or lose messages.
;* Publishers can't tell when subscribers have disappeared due to processes crashing, networks breaking, and so on.

* 발행자들은 구독자들의 연결 성공 시점이 초기 연결 혹은 네트워크 장애 후에 재연결인지 모릅니다.
* 구독자들은 발행자들에게 "전송 메시지 속도를 조정해 주세요"와 같은 어떤 메시지도 송신할 수 없습니다. 발행자들은 최대 속도로 메시지를 전송하면, 구독자의 성능의 따라 메시지 계속 받거나 유실하게 됩니다.
* 발행자들은 구독자들이 프로세스 충돌, 네트워크 중단 등으로 사라진 시점을 모릅니다.

;The downside is that we actually need all of these if we want to do reliable multicast. The ØMQ pub-sub pattern will lose messages arbitrarily when a subscriber is connecting, when a network failure occurs, or just if the subscriber or network can't keep up with the publisher.

위의 단점은 안정적인 멀티캐스트를 수행하기 위해 해결이 필요합니다. 
ØMQ 발행-구독 패턴은 구독자가 연결 중이거나 네트워크 장애가 발생하거나 구독자나 네트워크가 발행자의 송신하는 많은 메시지를 처리할 수 없는 경우 메시지가 유실됩니다.

;The upside is that there are many use cases where almost reliable multicast is just fine. When we need this back-chatter, we can either switch to using ROUTER-DEALER (which I tend to do for most normal volume cases), or we can add a separate channel for synchronization (we'll see an example of this later in this chapter).

하지만 긍정적인 면에서 안정적인 멀티캐스팅의 괜찮은 사용 사례가 많다는 것입니다. 
백-채터가 필요할 때, ROUTER-DEALER(보통 수준의 메시지 처리량일 경우 주로 사용)를 사용하도록 전환하거나 동기화를 위한 별도의 채널을 추가할 수 있습니다(이에 대한 예는 나중에 살펴보겠습니다).

;Pub-sub is like a radio broadcast; you miss everything before you join, and then how much information you get depends on the quality of your reception. Surprisingly, this model is useful and widespread because it maps perfectly to real world distribution of information. Think of Facebook and Twitter, the BBC World Service, and the sports results.

발행-구독은 라디오 방송과 같이 특정 채널에 가입하기 전에는 모든 것을 놓치며, 수신 품질에 따라 정보의 양이 달라집니다. 놀랍게도 이 모델은 실제 정보 배포에 완벽하게 매핑되어 유용하게 널리 사용되고 있습니다. 페이스북(Facebook)과 트위터(Twitter), BBC 세계 서비스, 스포츠 방송을 생각해 보십시오.

;As we did for request-reply, let's define reliability in terms of what can go wrong. Here are the classic failure cases for pub-sub:

요청-응답과 같이 무엇이 잘못될 수 있는지의 관점에서 신뢰성을 정의하겠습니다. 
다음은 발행-구독의 전형적인 장애 사례입니다.

;* Subscribers join late, so they miss messages the server already sent.
;* Subscribers can fetch messages too slowly, so queues build up and then overflow.
;* Subscribers can drop off and lose messages while they are away.
;* Subscribers can crash and restart, and lose whatever data they already received.
;* Networks can become overloaded and drop data (specifically, for PGM).
;* Networks can become too slow, so publisher-side queues overflow and publishers crash.

* 구독자가 늦게 가입하여 발행자가 이미 보낸 메시지들을 유실합니다.
* 구독자는 메시지들를 너무 느리게 처리하여, 대기열이 가득 차면 유실됩니다.
* 구독자는 자리를 비운 동안 메시지들를 유실할 수 있습니다.
* 구독자가 장애 후 재시작되면 이미 받은 데이터들을 유실할 수 있습니다.
* 네트워크는 과부하가 되어 데이터들을 유실할 수 있습니다(특히 PGM 통신의 경우).
* 네트워크의 처리 속도가 너무 느려 발행자의 대기열이 가득 차고 발행자에게 장애가 발생할 수 있습니다(ØMQ v3.2 이전).

;A lot more can go wrong but these are the typical failures we see in a realistic system. Since v3.x, ØMQ forces default limits on its internal buffers (the so-called high-water mark or HWM), so publisher crashes are rarer unless you deliberately set the HWM to infinite.

더 많은 것이 잘못될 수 있지만 이것들은 우리가 현실적인 시스템에서 보는 전형적인 장애입니다. 
ØMQ v3.x부터 ØMQ 대기열의 내부 버퍼(소위 HWM(high-water mark))에 대한 기본적인 제한을 적용하므로 의도적으로 HWM을 무한으로 설정하지 않으면 발행자의 대기열이 가득 차는 장애는 발생하지 않습니다.
> [옮긴이] HWM은 `zmq_setsockopt()`에서 송신의 경우 `ZMQ_SNDHWM`, 수신의 경우 `ZMQ_RCVHWM`으로 설정합니다.
- PUB, PUSH : 송신 버퍼 존재
- SUB, PULL, REQ, REP : 수신 버퍼 존재
- DEALER/ROUTER/PAIR : 송신 및 수신 버퍼 존재

;All of these failure cases have answers, though not always simple ones. Reliability requires complexity that most of us don't need, most of the time, which is why ØMQ doesn't attempt to provide it out of the box (even if there was one global design for reliability, which there isn't).

항상 단순한 것은 아니지만 장애 사례들에는 해결책이 있습니다. 
신뢰성은 대부분의 경우 필요하지 않는 복잡한 기능을 요구하지만, ØMQ는 기본적으로 제품 형태로 제공하려고 시도하진 않습니다(신뢰성을 적용하기 위한 범용적인 설계 방안은 존재하지 않습니다.).

;## Pub-Sub Tracing (Espresso Pattern)

## 발행-구독 추적 (에스프레소 패턴)

;Let's start this chapter by looking at a way to trace pub-sub networks. In Chapter 2 - Sockets and Patterns we saw a simple proxy that used these to do transport bridging. The zmq_proxy() method has three arguments: a frontend and backend socket that it bridges together, and a capture socket to which it will send all messages.

발행-구독 네트워크를 추적하는 방법을 보면서 이장을 시작하겠습니다. "2장 - 소켓 및 패턴"에서 전송 계층 간의 다리 역할을 수행하는 간단한 프록시를 보았습니다.
`zmq_proxy()` 메서드에는 3개 인수는 소켓들로 전송 계층 간 연결할 프론트앤드 소켓과 백엔드 소켓, 모든 메시지를 송신할 캡처 소켓입니다.
> [옮긴이] int zmq_proxy (void *frontend_, void *backend_, void *capture_)

;The code is deceptively simple:

다음 코드는 매우 단순합니다.

espresso.c: Espresso Pattern in C
```cpp
//  Espresso Pattern
//  This shows how to capture data using a pub-sub proxy

#include "czmq.h"

//  The subscriber thread requests messages starting with
//  A and B, then reads and counts incoming messages.

static void
subscriber_thread (void *args, zctx_t *ctx, void *pipe)
{
    //  Subscribe to "A" and "B"
    void *subscriber = zsocket_new (ctx, ZMQ_SUB);
    zsocket_connect (subscriber, "tcp://localhost:6001");
    zsocket_set_subscribe (subscriber, "A");
    zsocket_set_subscribe (subscriber, "B");

    int count = 0;
    while (count < 5) {
        char *string = zstr_recv (subscriber);
        if (!string)
            break;              //  Interrupted
        free (string);
        count++;
    }
    zsocket_destroy (ctx, subscriber);
}

//  .split publisher thread
//  The publisher sends random messages starting with A-J:

static void
publisher_thread (void *args, zctx_t *ctx, void *pipe)
{
    void *publisher = zsocket_new (ctx, ZMQ_PUB);
    zsocket_bind (publisher, "tcp://*:6000");

    while (!zctx_interrupted) {
        char string [10];
        sprintf (string, "%c-%05d", randof (10) + 'A', randof (100000));
        if (zstr_send (publisher, string) == -1)
            break;              //  Interrupted
        zclock_sleep (100);     //  Wait for 1/10th second
    }
}

//  .split listener thread
//  The listener receives all messages flowing through the proxy, on its
//  pipe. In CZMQ, the pipe is a pair of ZMQ_PAIR sockets that connect
//  attached child threads. In other languages your mileage may vary:

static void
listener_thread (void *args, zctx_t *ctx, void *pipe)
{
    //  Print everything that arrives on pipe
    while (true) {
        zframe_t *frame = zframe_recv (pipe);
        if (!frame)
            break;              //  Interrupted
        zframe_print (frame, NULL);
        zframe_destroy (&frame);
    }
}

//  .split main thread
//  The main task starts the subscriber and publisher, and then sets
//  itself up as a listening proxy. The listener runs as a child thread:

int main (void)
{
    //  Start child threads
    zctx_t *ctx = zctx_new ();
    zthread_fork (ctx, publisher_thread, NULL);
    zthread_fork (ctx, subscriber_thread, NULL);

    void *subscriber = zsocket_new (ctx, ZMQ_XSUB);
    zsocket_connect (subscriber, "tcp://localhost:6000");
    void *publisher = zsocket_new (ctx, ZMQ_XPUB);
    zsocket_bind (publisher, "tcp://*:6001");
    void *listener = zthread_fork (ctx, listener_thread, NULL);
    zmq_proxy (subscriber, publisher, listener);

    puts (" interrupted");
    //  Tell attached threads to exit
    zctx_destroy (&ctx);
    return 0;
}
```
> [옮긴이] 빌드 및 테스트

~~~{.bash}
PS D:\git_store\zguide-kr\examples\C> ./espresso
D: 20-08-26 13:43:30 [002] 0141
D: 20-08-26 13:43:31 [002] 0142
D: 20-08-26 13:43:31 [007] A-30398
D: 20-08-26 13:43:31 [007] B-14730
D: 20-08-26 13:43:32 [007] A-11907
D: 20-08-26 13:43:33 [007] B-53933
D: 20-08-26 13:43:33 [007] A-84011
D: 20-08-26 13:43:33 [002] 0041
D: 20-08-26 13:43:33 [002] 0042
~~~

;Espresso works by creating a listener thread that reads a PAIR socket and prints anything it gets. That PAIR socket is one end of a pipe; the other end (another PAIR) is the socket we pass to zmq_proxy(). In practice, you'd filter interesting messages to get the essence of what you want to track (hence the name of the pattern).

에스프레소는 생성한 리스너 스레드는 PAIR 소켓을 읽고 출력을 합니다. PAIR 소켓은 파이프(pipe)의 한쪽 끝이며 다른 쪽 끝(추가 PAIR)은 `zmq_proxy()`에 전달하는 소켓입니다. 
실제로 관심 있는 메시지를 필터링하여 추적하려는 대상(에스프레소 패턴의 이름처럼)의 본질을 얻습니다.
> [옮긴이] 에스프레소(Espresso)는 곱게 갈아 압축한 원두가루에 뜨거운 물을 고압으로 통과시켜 뽑아낸 이탈리안 정통 커피로 아주 진한 향과 맛이 특징이다.

;The subscriber thread subscribes to "A" and "B", receives five messages, and then destroys its socket. When you run the example, the listener prints two subscription messages, five data messages, two unsubscribe messages, and then silence:

구독자 스레드는 "A"및 "B"를 포함한 메시지 구독하며, 수신된 메시지가 5 개째 소켓을 제거하고 종료합니다. 예제를 실행하면 리스너는 2개의 구독 메시지(/01)들, 5개의 데이터 메시지들("A", "B" 포함)과 2개의 구독 취소 메시지(/00)를 출력하고 조용해집니다.

~~~{.bash}
[002] 0141
[002] 0142
[007] B-91164
[007] B-12979
[007] A-52599
[007] A-06417
[007] A-45770
[002] 0041
[002] 0042
~~~

;This shows neatly how the publisher socket stops sending data when there are no subscribers for it. The publisher thread is still sending messages. The socket just drops them silently.

이것은 발행자 소켓이 구독자가 없을 때 데이터 전송을 중지하는 방법을 깔끔하게 보여줍니다. 
발행자 스레드가 여전히 메시지를 보내고 있지만 소켓은 조용히 그들을 버립니다.

> [옮긴이] 구독자는 발행자에게 구독 시와 구독 취소 시 이벤트(event)를 보내며 보내는 메시지는 바이트(byte) 형태로 첫 번째 바이트(HEX 코드)는 "00"은 구독, "01"은 구독 취소이며 나머지 바이트들은 토픽(sizeof(event)-1)으로 구성됩니다.

~~~{.bash}
[002] 0141       --> "01" 구독, 토픽 : "41" A
[002] 0142       --> "01" 구독, 토픽 : "41" B
[007] B-91164
[007] B-12979
[007] A-52599
[007] A-06417
[007] A-45770
[002] 0041       --> "00" 구독 취소, 토픽 : "41" A
[002] 0042       --> "00" 구독 취소, 토픽 : "41" B
~~~

;## Last Value Caching

## 마지막 값 캐싱

;If you've used commercial pub-sub systems, you may be used to some features that are missing in the fast and cheerful ØMQ pub-sub model. One of these is last value caching (LVC). This solves the problem of how a new subscriber catches up when it joins the network. The theory is that publishers get notified when a new subscriber joins and subscribes to some specific topics. The publisher can then rebroadcast the last message for those topics.

상용 발행-구독 시스템을 사용했다면 빠르고 즐거운 ØMQ 발행-구독 모델에서 누락된 일부 익숙한 기능들을 알 수 있습니다. 이 중 하나는 마지막 값 캐싱(LVC : Last Value Caching)입니다. 이것은 새로운 구독자가 네트워크에 참여할 때 누락한 메시지 받을 수 있는 방식으로 문제를 해결합니다. 이론은 발행자들에게 새로운 구독자가 참여했고 특정 토픽에 구독하려는 시점을 알려주면, 발행자는 해당 토픽에 대한 마지막 메시지를 재송신할 수 있습니다.
> [옮긴이] TIB/RV(TIBCO Rendezvous)는 TIBCO사에서 개발한 상용 메시지 소프트웨어로 요청/응답(Request/reply), 브로드케스팅(Broadcasting), 신뢰성 메시징(reliable messaging), 메시지 전달 보장(Certified Messaging), 분산 메시지(Distributed Message), 원격 통신(Remote communication)등의 기능을 제공합니다.

;I've already explained why publishers don't get notified when there are new subscribers, because in large pub-sub systems, the volumes of data make it pretty much impossible. To make really large-scale pub-sub networks, you need a protocol like PGM that exploits an upscale Ethernet switch's ability to multicast data to thousands of subscribers. Trying to do a TCP unicast from the publisher to each of thousands of subscribers just doesn't scale. You get weird spikes, unfair distribution (some subscribers getting the message before others), network congestion, and general unhappiness.

발행자들이 새로운 구독자들이 참여하는 것을 알지 못하는 것은 대규모 발행-구독 시스템에서는 데이터의 양으로 인해 거의 불가능하기 때문입니다.
정말 대규모 발행-구독 네트워크를 만들려면 PGM과 같은 통신규약을 통해 이더넷 스위치의 기능을 활용하여 수천 명의 구독자에게 데이터를 멀티캐스트 해야 합니다. 
발행자가 각각 수천 명의 구독자들에게 TCP 유니캐스트를 시도하는 것으로 확장할 수 없으며, 이상한 불협화음, 불공정한 배포(일부 구독자가 다른 구독자보다 먼저 메시지를 받음), 네트워크 정체 등으로 보통 좋지 않은 결과를 겪습니다.

;PGM is a one-way protocol: the publisher sends a message to a multicast address at the switch, which then rebroadcasts that to all interested subscribers. The publisher never sees when subscribers join or leave: this all happens in the switch, which we don't really want to start reprogramming.

PGM은 단방향 통신규약입니다. 발행자는 네트워크 스위치의 멀티캐스트 주소로 메시지를 보내면 모든 관심 있는 구독자들에게 다시 브로드캐스트 합니다. 발행자는 구독자가 가입하거나 탈퇴하는 시점을  결코 알 수 없습니다. 이 모든 작업은 스위치에서 발생하며, 네트워크 스위치의 프로그래밍을 다시 작성하고 싶지는 않습니다.

;However, in a lower-volume network with a few dozen subscribers and a limited number of topics, we can use TCP and then the XSUB and XPUB sockets do talk to each other as we just saw in the Espresso pattern.

그러나 낮은 대역폭을 가지는 저용량 네트워크에 수십 명의 구독자의 제한된 토픽가 있으면 TCP를 사용할 수 있으며, XSUB 및 XPUB 소켓은 에스프레소 패턴에서 본 것처럼 상호 통신합니다.

;Can we make an LVC(last value cache) using ØMQ? The answer is yes, if we make a proxy that sits between the publisher and subscribers; an analog for the PGM switch, but one we can program ourselves.

"ØMQ를 사용하여 LVC(last value cache)를 만들 수 있을까요?"에 대한 대답은 "예"이며 발행자와 구독자들 사이에 프록시를 만들면 됩니다. PGM 네트워크 스위치와 유사하지만 ØMQ는 우리가 직접 프로그래밍할 수 있습니다.

;I'll start by making a publisher and subscriber that highlight the worst case scenario. This publisher is pathological. It starts by immediately sending messages to each of a thousand topics, and then it sends one update a second to a random topic. A subscriber connects, and subscribes to a topic. Without LVC, a subscriber would have to wait an average of 500 seconds to get any data. To add some drama, let's pretend there's an escaped convict called Gregor threatening to rip the head off Roger the toy bunny if we can't fix that 8.3 minutes' delay.

최악의 시나리오를 가정한 발행자와 구독자를 만드는 것부터 시작하겠습니다. 이 발행자는 병리학적입니다. 발행자는 각각 1000개의 토픽들에 즉시 메시지를 보내고 1초마다 임의의 토픽에 하나의 변경정보를 전송합니다. 구독자는 연결하고 토픽을 구독합니다. 마지막 값 캐핑(LVC)이 없으면 구독자는 데이터를 얻기 위해 평균 500초를 기다려야 합니다. 
TV 드라마에서 그레고르라는 탈옥한 죄수가 8.3분 내에 구독자의 데이터를 얻게 하지 못한다면 장난감 토끼 로저에게서 머리를 날려버리겠다고 위협하는 것처럼 말입니다.

;Here's the publisher code. Note that it has the command line option to connect to some address, but otherwise binds to an endpoint. We'll use this later to connect to our last value cache:

다음은 발행자 코드입니다. 일부 주소에 연결하는 명령 줄 옵션이 없으면 단말에 바인딩되며, 나중에 마지막 값 캐시(LVC)에 연결에 사용합니다.

pathopub.c : Pathologic Publisher in C
```cpp
//  Pathological publisher
//  Sends out 1,000 topics and then one random update per second

#include "czmq.h"
#include "zhelpers.h"

int main (int argc, char *argv [])
{
    zctx_t *context = zctx_new ();
    void *publisher = zsocket_new (context, ZMQ_PUB);
    if (argc == 2)
        zsocket_bind (publisher, argv [1]);
    else
        zsocket_bind (publisher, "tcp://*:5556");

    //  Ensure subscriber connection has time to complete
    s_sleep (1000);

    //  Send out all 1,000 topic messages
    int topic_nbr;
    for (topic_nbr = 0; topic_nbr < 1000; topic_nbr++) {
        zstr_sendfm (publisher, "%03d", topic_nbr);
        zstr_send (publisher, "Save Roger");
    }
    //  Send one random update per second
    srandom ((unsigned) time (NULL));
    while (!zctx_interrupted) {
        s_sleep (1000);
        zstr_sendfm (publisher, "%03d", randof (1000));
        zstr_send (publisher, "Off with his head!");
    }
    zctx_destroy (&context);
    return 0;
}
```

;And here's the subscriber:

구독자 코드는 다음과 같습니다.

pathosub.c : Pathologic Subscriber in C
```cpp
//  Pathological subscriber
//  Subscribes to one random topic and prints received messages

#include "czmq.h"

int main (int argc, char *argv [])
{
    zctx_t *context = zctx_new ();
    void *subscriber = zsocket_new (context, ZMQ_SUB);
    if (argc == 2)
        zsocket_connect (subscriber, argv [1]);
    else
        zsocket_connect (subscriber, "tcp://localhost:5556");

    srandom ((unsigned) time (NULL));
    char subscription [5];
    sprintf (subscription, "%03d", randof (1000));
    zsocket_set_subscribe (subscriber, subscription);
    
    while (true) {
        char *topic = zstr_recv (subscriber);
        if (!topic)
            break;
        char *data = zstr_recv (subscriber);
        assert (streq (topic, subscription));
        puts (data);
        free (topic);
        free (data);
    }
    zctx_destroy (&context);
    return 0;
}
```

;Try building and running these: first the subscriber, then the publisher. You'll see the subscriber reports getting "Save Roger" as you'd expect:

먼저 구독자를 실행하고 다음 발행자를 실행하면 구독자가 기대한 대로 "Save Roger"를 출력합니다.
~~~{.bash}
./pathosub &
./pathopub
~~~

> [옮긴이] 빌드 및 테스트
 - "pathsub" 실행하면 생성된 토픽에 해당되는 "Sava Roger"을 수신하고 임의의 긴 시간(1초~16.7분) 후에 "Off with his head!"을 수신하게 됩니다.

~~~{.bash}
PS D:\git_store\zguide-kr\examples\C> cl -EHsc pathopub.c libzmq.lib czmq.lib
PS D:\git_store\zguide-kr\examples\C> cl -EHsc pathosub.c libzmq.lib czmq.lib

PS D:\git_store\zguide-kr\examples\C> ./pathopub

PS D:\git_store\zguide-kr\examples\C> ./pathosub
Save Roger
Off with his head!
~~~

;It's when you run a second subscriber that you understand Roger's predicament. You have to leave it an awful long time before it reports getting any data. So, here's our last value cache. As I promised, it's a proxy that binds to two sockets and then handles messages on both:

두 번째 구독자를 실행하면 Roger의 곤경("Save Roger"을 수신하지 못하고 오랜 시간 후에 "Off with his head!" 수신)을 알게 되며 데이터 수신하기까지 오랜 시간을 기다려야 합니다. 여기에 마지막 값 캐시(LVC)가 있습니다. 프록시가 2개의 소켓에 바인딩하고 양쪽의 메시지를 처리합니다.

lvcach.c : 마지막 값 저장 프록시
```cpp
//  Last value cache
//  Uses XPUB subscription messages to re-send data

#include "czmq.h"

int main (void)
{
    zctx_t *context = zctx_new ();
    void *frontend = zsocket_new (context, ZMQ_SUB);
    zsocket_connect (frontend, "tcp://*:5557");
    void *backend = zsocket_new (context, ZMQ_XPUB);
    zsocket_bind (backend, "tcp://*:5558");

    //  Subscribe to every single topic from publisher
    zsocket_set_subscribe (frontend, "");

    //  Store last instance of each topic in a cache
    zhash_t *cache = zhash_new ();

    //  .split main poll loop
    //  We route topic updates from frontend to backend, and
    //  we handle subscriptions by sending whatever we cached,
    //  if anything:
    while (true) {
        zmq_pollitem_t items [] = {
            { frontend, 0, ZMQ_POLLIN, 0 },
            { backend,  0, ZMQ_POLLIN, 0 }
        };
        if (zmq_poll (items, 2, 1000 * ZMQ_POLL_MSEC) == -1)
            break;              //  Interrupted

        //  Any new topic data we cache and then forward
        if (items [0].revents & ZMQ_POLLIN) {
            char *topic = zstr_recv (frontend);
            char *current = zstr_recv (frontend);
            if (!topic)
                break;
            char *previous = zhash_lookup (cache, topic);
            if (previous) {
                zhash_delete (cache, topic);
                free (previous);
            }
            zhash_insert (cache, topic, current);
            zstr_sendm (backend, topic);
            zstr_send (backend, current);
            free (topic);
        }
        //  .split handle subscriptions
        //  When we get a new subscription, we pull data from the cache:
        if (items [1].revents & ZMQ_POLLIN) {
            zframe_t *frame = zframe_recv (backend);
            if (!frame)
                break;
            //  Event is one byte 0=unsub or 1=sub, followed by topic
            byte *event = zframe_data (frame);
            if (event [0] == 1) {
                char *topic = zmalloc (zframe_size (frame));
                memcpy (topic, event + 1, zframe_size (frame) - 1);
                printf ("Sending cached topic %s\n", topic);
                char *previous = zhash_lookup (cache, topic);
                if (previous) {
                    zstr_sendm (backend, topic);
                    zstr_send (backend, previous);
                }
                free (topic);
            }
            zframe_destroy (&frame);
        }
    }
    zctx_destroy (&context);
    zhash_destroy (&cache);
    return 0;
}
```

;Now, run the proxy, and then the publisher:

프록시와 발행자를 순서대로 실행합니다.

~~~{.bash}
./lvcache &
./pathopub tcp://localhost:5557
~~~

;And now run as many instances of the subscriber as you want to try, each time connecting to the proxy on port 5558:

그리고 원하는 수의 구독자들을 실행 시 프록시의 5558 포트에 연결합니다.

~~~{.bash}
./pathosub tcp://localhost:5558
~~~
>[옮긴이] 빌드 및 테스트
 - "lvcahe.c" 소스코드의 이상과 pathopub 인수가 잘못되어 정상 동작하지 않습니다.

~~~{.bash}
PS D:\git_store\zguide-kr\examples\C> ./lvcache
Sending cached topic 000

PS D:\git_store\zguide-kr\examples\C> ./pathopub tcp://localhost:5557

PS D:\git_store\zguide-kr\examples\C> ./pathosub tcp://localhost:5558
~~~
"lvcahe.c"에서 프론트엔드 소켓에 접속하기 위한 부분이 잘못되었기 때문입니다.
```cpp
// 수정전
    zsocket_connect (frontend, "tcp://*:5557");
// 수정후
    zsocket_connect (frontend, "tcp://localhost:5557");
```
pathopub 수행 시 인수도 수정 필요합니다.
~~~{.bash}
// 수정전
PS D:\git_store\zguide-kr\examples\C> ./pathopub tcp://localhost:5557
// 수정후
PS D:\git_store\zguide-kr\examples\C> ./pathopub tcp://*:5557
~~~

> [옮긴이] 수정된 lvcache.c 

```cpp
//  Last value cache
//  Uses XPUB subscription messages to re-send data

#include "czmq.h"

int main (void)
{
    zctx_t *context = zctx_new ();
    void *frontend = zsocket_new (context, ZMQ_SUB);
    zsocket_connect (frontend, "tcp://localhost:5557");
    void *backend = zsocket_new (context, ZMQ_XPUB);
    zsocket_bind (backend, "tcp://*:5558");

    //  Subscribe to every single topic from publisher
    zsocket_set_subscribe (frontend, "");

    //  Store last instance of each topic in a cache
    zhash_t *cache = zhash_new ();

    //  .split main poll loop
    //  We route topic updates from frontend to backend, and
    //  we handle subscriptions by sending whatever we cached,
    //  if anything:
    while (true) {
        zmq_pollitem_t items [] = {
            { frontend, 0, ZMQ_POLLIN, 0 },
            { backend,  0, ZMQ_POLLIN, 0 }
        };
        if (zmq_poll (items, 2, 1000 * ZMQ_POLL_MSEC) == -1)
            break;              //  Interrupted

        //  Any new topic data we cache and then forward
        if (items [0].revents & ZMQ_POLLIN) {
            char *topic = zstr_recv (frontend);
            char *current = zstr_recv (frontend);
            if (!topic)
                break;
            char *previous = zhash_lookup (cache, topic);
            if (previous) {
                zhash_delete (cache, topic);
                free (previous);
            }
            zhash_insert (cache, topic, current);
            zstr_sendm (backend, topic);
            zstr_send (backend, current);
            free (topic);
        }
        //  .split handle subscriptions
        //  When we get a new subscription, we pull data from the cache:
        if (items [1].revents & ZMQ_POLLIN) {
            zframe_t *frame = zframe_recv (backend);
            if (!frame)
                break;
            //  Event is one byte 0=unsub or 1=sub, followed by topic
            byte *event = zframe_data (frame);
            if (event [0] == 1) {
                char *topic = zmalloc (zframe_size (frame));
                memcpy (topic, event + 1, zframe_size (frame) - 1);
                printf ("Sending cached topic %s\n", topic);
                char *previous = zhash_lookup (cache, topic);
                if (previous) {
                    zstr_sendm (backend, topic);
                    zstr_send (backend, previous);
                }
                free (topic);
            }
            zframe_destroy (&frame);
        }
    }
    zctx_destroy (&context);
    zhash_destroy (&cache);
    return 0;
}
```
> [옮긴이] pathopub 인수 변경 후 테스트

~~~{.bash} 
PS D:\git_store\zguide-kr\examples\C> ./lvcache
Sending cached topic 045

PS D:\git_store\zguide-kr\examples\C> ./pathopub tcp://*:5557

PS D:\git_store\zguide-kr\examples\C> ./pathosub tcp://localhost:5558
Save Roger

PS D:\git_store\zguide-kr\examples\C> ./pathosub tcp://localhost:5558
Off with his head!
~~~

;Each subscriber happily reports "Save Roger", and Gregor the Escaped Convict slinks back to his seat for dinner and a nice cup of hot milk, which is all he really wanted in the first place.

각 구독자는 "Save Roger"를 보고 받으며, 탈옥한 죄수 그레고르는 다시 감옥으로 돌아가 따뜻한 우유 한잔과 저녁 식사를 하게 되었습니다. 그는 진정 범죄를 저지르기보다는 관심받기를 원한 것이었습니다.

;One note: by default, the XPUB socket does not report duplicate subscriptions, which is what you want when you're naively connecting an XPUB to an XSUB. Our example sneakily gets around this by using random topics so the chance of it not working is one in a million. In a real LVC proxy, you'll want to use the ZMQ_XPUB_VERBOSE option that we implement in Chapter 6 - The ØMQ Community as an exercise.

참고 : 기본적으로 XPUB 소켓은 중복 구독들을 보고하지 않습니다. 이는 XPUB를 XSUB에 그냥 연결할 때 원하는 것입니다. 우리의 예제는 임의의 토픽들을 사용하여 몰래 처리하므로 작동하지 않을 가능성은 백만분의 1입니다. 실제 LVC 프록시에서는 `ZMQ_XPUB_VERBOSE` 옵션을 사용하며 "6장 - ØMQ 커뮤니티"에서 연습문제로 구현하겠습니다.

;## Slow Subscriber Detection (Suicidal Snail Pattern)

## 느린 구독자 감지(자살하는 달팽이 패턴)

;A common problem you will hit when using the pub-sub pattern in real life is the slow subscriber. In an ideal world, we stream data at full speed from publishers to subscribers. In reality, subscriber applications are often written in interpreted languages, or just do a lot of work, or are just badly written, to the extent that they can't keep up with publishers.

실제 생활에서 발행-구독 패턴을 사용할 때 발생하는 일반적인 문제는 느린 구독자입니다. 이상적인 세상에서 우리는 발행자에서 구독자로 데이터를 전속력으로 전송합니다. 실제로 구독자 응용프로그램은 종종 인터프리터 개발 언어로 작성되었거나 혹은 많은 작업을 처리하거나 잘못된 코드로 인한 오류로 발행자가 전송하는 데이터를 처리하지 못하는 상황이 발생할 수 있습니다.
> [옮긴이] 인터프리터(interpreter) 개발 언어는 단말기를 통하여 컴퓨터와 대화하면서 작성할 수 있는 프로그래밍 언어로 PYTHON, BASIC, Prolog. LISP, LOGO, R 등이 있습니다.

;How do we handle a slow subscriber? The ideal fix is to make the subscriber faster, but that might take work and time. Some of the classic strategies for handling a slow subscriber are:

느린 구독자를 처리하는 방법으로 이상적인 해결책은 구독자를 더 빠르게 만드는 것이지만 응용프로그램을 최적화하는 작업에 많은 시간이 소요될 수 있습니다. 
느린 구독자를 개선하기 위한 몇 가지 전통적인 전략들은 다음과 같습니다.

;* Queue messages on the publisher. This is what Gmail does when I don't read my email for a couple of hours. But in high-volume messaging, pushing queues upstream has the thrilling but unprofitable result of making publishers run out of memory and crash—especially if there are lots of subscribers and it's not possible to flush to disk for performance reasons.
;* Queue messages on the subscriber. This is much better, and it's what ØMQ does by default if the network can keep up with things. If anyone's going to run out of memory and crash, it'll be the subscriber rather than the publisher, which is fair. This is perfect for "peaky" streams where a subscriber can't keep up for a while, but can catch up when the stream slows down. However, it's no answer to a subscriber that's simply too slow in general.
;* Stop queuing new messages after a while. This is what Gmail does when my mailbox overflows its precious gigabytes of space. New messages just get rejected or dropped. This is a great strategy from the perspective of the publisher, and it's what ØMQ does when the publisher sets a HWM. However, it still doesn't help us fix the slow subscriber. Now we just get gaps in our message stream.
;* Punish slow subscribers with disconnect. This is what Hotmail (remember that?) did when I didn't log in for two weeks, which is why I was on my fifteenth Hotmail account when it hit me that there was perhaps a better way. It's a nice brutal strategy that forces subscribers to sit up and pay attention and would be ideal, but ØMQ doesn't do this, and there's no way to layer it on top because subscribers are invisible to publisher applications.

* 발행자의 메시지 대기열에 보관 
몇 시간 동안 이메일을 읽지 않을 때 Gmail이 하는 일입니다. 그러나 대용량 메시징에서 구독자가 많고 성능상의 이유로 발행자 대기열을 디스크의 데이터를 저장할 수 없는 경우 발행자의 메모리 부족과 충돌이 발생할 수 있습니다. 
* 구독자의 메시지 대기열에 보관
이것은 훨씬 낫고, 네트워크의 대역폭이 가능하다면 ØMQ가 기본적으로 하는 일입니다. 누군가 메모리가 부족하고 충돌이 발생하면 발행자가 아닌 구독자가 될 것입니다. 
이것은 "피크(Peaky)" 스트림에 적합하며 메시징이 많아 구독자가 한동안 처리할 수 없지만 메시징이 적어지면 처리 가능합니다. 그러나 일반적으로 너무 느린 구독자에게는 해결책이 아닙니다.
* 잠시 동안 신규 메시지를 수신 중단. 
수신 편지함의 저장 공간을 초과하는 경우 Gmail에서 수행하는 작업입니다.  신규 메시지는 거부되거나 삭제됩니다. 이것은 발행자의 관점에서는 훌륭한 전략이며, ØMQ에서 발행자가 HWM(High Water Mark)을 설정 시 적용됩니다. 그러나 여전히 느린 구독자를 개선하지는 못합니다. 단지 메시지 전송을 하지 못한 간격(GAP)을 가지게 됩니다.
* 느린 구독자를 연결을 끊어 처벌하기. 
Hotmail에 2주 동안  로그인하지 않았을 때 수행하는 것이며, 내가 15번째 Hotmail 계정을 사용한 이유입니다.
이것은 잔인한 전략으로 구독자가 앉아서 주의를 기울이게 하며, 이상적이지만 ØMQ는 이를 수행하지 않는 이유는 발행자 응용프로그램에는 구독자가 보이지 않기 때문입니다.

;None of these classic strategies fit, so we need to get creative. Rather than disconnect the publisher, let's convince the subscriber to kill itself. This is the Suicidal Snail pattern. When a subscriber detects that it's running too slowly (where "too slowly" is presumably a configured option that really means "so slowly that if you ever get here, shout really loudly because I need to know, so I can fix this!"), it croaks and dies.

어떤 전통적인 전략들도 적합하지 않아, 창의력을 발휘해야 합니다. 
발행자와의 연결을 끊는 대신 구독자가 자살하도록 설득하겠습니다. 이것은 자살하는 달팽이 패턴입니다. 구독자가 너무 느리게 실행되고 있음을 감지할 때, 한번 울고 죽게 합니다. "너무 느리다"는 판단은 구성 옵션으로 의미하는 바는 "만약 너무 천천히 당신이 여기 온다면, 내가 알아야 하니까 크게 소리쳐. 내가 고칠 수 있어!"입니다.

;How can a subscriber detect this? One way would be to sequence messages (number them in order) and use a HWM at the publisher. Now, if the subscriber detects a gap (i.e., the numbering isn't consecutive), it knows something is wrong. We then tune the HWM to the "croak and die if you hit this" level.

구독자는 "너무 느리다"를 감지하는 방법으로 한가지는 메시지를 순서대로 나열하고(순서대로 번호 지정) 발행자에서 HWM을 사용하는 것입니다. 그리고 구독자가 간격(즉, 번호가 연속적이지 않음)을 감지하면 잘못됨을 알게 되며, HWM을 "이 수준에 도달하면 울고 죽기" 수준으로 조정합니다.

;There are two problems with this solution. One, if we have many publishers, how do we sequence messages? The solution is to give each publisher a unique ID and add that to the sequencing. Second, if subscribers use ZMQ_SUBSCRIBE filters, they will get gaps by definition. Our precious sequencing will be for nothing.

이 해결책에는 2가지 문제가 있습니다. 
첫째, 발행자들이 많은 경우, 메시지 순서를 지정하는 방법이며 해결책은 각 발행자는 고유한 식별자(ID)를 가지고 순서에 추가하는 것입니다.(PUB ID + SEQUENCE) 
둘째, 구독자들이 ZMQ_SUBSCRIBE 필터를 사용하면 정의에 따라 간격이 생기며, 우리의 귀중한 메시지 순서는 소용이 없습니다.

;Some use cases won't use filters, and sequencing will work for them. But a more general solution is that the publisher timestamps each message. When a subscriber gets a message, it checks the time, and if the difference is more than, say, one second, it does the "croak and die" thing, possibly firing off a squawk to some operator console first.

일부 사용 사례들은 필터를 미사용 해야지 메시지 순서가 작동합니다. 그러나 보다 일반적인 해결책은 발행자가 각 메시지에 타임스탬프를 찍는 것입니다. 구독자가 메시지를 받으면 시간을 확인하고 그 차이가 1초 이상이면 "울고 죽기" 작업을 수행하며, 먼저 일부 운영자 컴퓨터 화면에 신호를 보냅니다.

;The Suicide Snail pattern works especially when subscribers have their own clients and service-level agreements and need to guarantee certain maximum latencies. Aborting a subscriber may not seem like a constructive way to guarantee a maximum latency, but it's the assertion model. Abort today, and the problem will be fixed. Allow late data to flow downstream, and the problem may cause wider damage and take longer to appear on the radar.

자살하는 달팽이 패턴은 특히 구독자가 자신의 클라이언트들과 서비스 수준 계약(SLA, Service Level Agreements)을 맺고 있고 특정 최대 지연 시간을 보장해야 하는 경우에 효과적입니다. 최대 지연 시간을 보장하기 위해 구독자를 중단하는 것은 건설적인 방법은 아니지만, 가정 설정문(assertion) 모델입니다. 오늘 중단하면 문제는 해결되지만 지연되는 데이터가 하류(구독자들)로 흐르도록 허용하면 문제가 더 넓게 퍼져 많은 손상을 발생하지만 레이더(문제를 감지하는 수단)로 감지하는데 오래 걸릴 수 있습니다.
> [옮긴이] 가정 설정문(Assertions)은 프로그램상에서 실수는 일어난다고 가정하고 이에 대비하여 방지하기 위해 가정 설정문(Assertion)의 조건을 설정하고 참(TRUE)이면 아무것도 하지 않지만, 거짓(FALSE)이면 즉시 프로그램을 정지시키는 방어적 프로그래밍(defensive programming)의 수단입니다.

;Here is a minimal example of a Suicidal Snail:

자살하는 달팽이에 대한 작은 예제입니다.

suisnail.c: Suicidal Snail in C
```cpp
//  Suicidal Snail

#include "czmq.h"

//  This is our subscriber. It connects to the publisher and subscribes
//  to everything. It sleeps for a short time between messages to
//  simulate doing too much work. If a message is more than one second
//  late, it croaks.

#define MAX_ALLOWED_DELAY   1000    //  msecs

static void
subscriber (void *args, zctx_t *ctx, void *pipe)
{
    //  Subscribe to everything
    void *subscriber = zsocket_new (ctx, ZMQ_SUB);
    zsocket_set_subscribe (subscriber, "");
    zsocket_connect (subscriber, "tcp://localhost:5556");

    //  Get and process messages
    while (true) {
        char *string = zstr_recv (subscriber);
        printf("%s\n", string);
        int64_t clock;
        int terms = sscanf (string, "%" PRId64, &clock);
        assert (terms == 1);
        free (string);

        //  Suicide snail logic
        if (zclock_time () - clock > MAX_ALLOWED_DELAY) {
            fprintf (stderr, "E: subscriber cannot keep up, aborting\n");
            break;
        }
        //  Work for 1 msec plus some random additional time
        zclock_sleep (1 + randof (2));
    }
    zstr_send (pipe, "gone and died");
}

//  .split publisher task
//  This is our publisher task. It publishes a time-stamped message to its
//  PUB socket every millisecond:

static void
publisher (void *args, zctx_t *ctx, void *pipe)
{
    //  Prepare publisher
    void *publisher = zsocket_new (ctx, ZMQ_PUB);
    zsocket_bind (publisher, "tcp://*:5556");

    while (true) {
        //  Send current clock (msecs) to subscribers
        char string [20];
        sprintf (string, "%" PRId64, zclock_time ());
        zstr_send (publisher, string);
        char *signal = zstr_recv_nowait (pipe);
        if (signal) {
            free (signal);
            break;
        }
        zclock_sleep (1);            //  1msec wait
    }
}

//  .split main task
//  The main task simply starts a client and a server, and then
//  waits for the client to signal that it has died:

int main (void)
{
    zctx_t *ctx = zctx_new ();
    void *pubpipe = zthread_fork (ctx, publisher, NULL);
    void *subpipe = zthread_fork (ctx, subscriber, NULL);
    free (zstr_recv (subpipe));
    zstr_send (pubpipe, "break");
    zclock_sleep (100);
    zctx_destroy (&ctx);
    return 0;
}
```
> [옮긴이] 빌드 및 실행

~~~{.bash}
PS D:\git_store\zguide-kr\examples\C> cl -EHsc suisnail.c libzmq.lib czmq.lib

PS D:\git_store\zguide-kr\examples\C> ./suisnail
13242959186075
13242959186077
13242959186079
13242959186081
...
13242959203828
13242959203830
13242959203832
E: subscriber cannot keep up, aborting
~~~

;Here are some things to note about the Suicidal Snail example:

자살하는 달팽이 예제에서 몇 가지 주목할 사항은 다음과 같습니다.

;* The message here consists simply of the current system clock as a number of milliseconds. In a realistic application, you'd have at least a message header with the timestamp and a message body with data.
;* The example has subscriber and publisher in a single process as two threads. In reality, they would be separate processes. Using threads is just convenient for the demonstration.

* 여기에 있는 메시지는 밀리초(msec) 단위의 현재 시스템 시각으로만 구성됩니다. 현실적인 응용프로그램에서는 적어도 메시지는 "타임스탬프가 있는 메시지 헤더"와 "데이터가 있는 메시지 본문"으로 구성되어야 합니다.
* 예제에서는 단일 프로세스에서 2개의 스레드인 발행자와 구독자가 있습니다. 실제로는 별도의 프로세스들입니다. 스레드는 편의상 데모를 위하여 사용합니다.

;## High-Speed Subscribers (Black Box Pattern)

## 고속 구독자 (블랙 박스 패턴)

;Now lets look at one way to make our subscribers faster. A common use case for pub-sub is distributing large data streams like market data coming from stock exchanges. A typical setup would have a publisher connected to a stock exchange, taking price quotes, and sending them out to a number of subscribers. If there are a handful of subscribers, we could use TCP. If we have a larger number of subscribers, we'd probably use reliable multicast, i.e., PGM.

이제 구독자를 더 빠르게 만드는 한 가지 방법을 보겠습니다. 발행-구독의 일반적인 사용 사례는 증권거래소에서 발생하는 시장 데이터와 같은 대용량 데이터 스트림을 배포하는 것입니다. 
일반적인 설정에는 발행자가 증권거래소에 연결되어 주가를 여러 구독자들에게 보냅니다. 구독자가 적으면 TCP를 사용할 수 있습니다. 많은 구독자의 경우 안정적인 멀티캐스트, 즉 PGM을 사용합니다.

그림 56 - 단순 블랙박스 패턴

![The Simple Black Box Pattern](images/fig56.svg)

;Let's imagine our feed has an average of 100,000 100-byte messages a second. That's a typical rate, after filtering market data we don't need to send on to subscribers. Now we decide to record a day's data (maybe 250GB in 8 hours), and then replay it to a simulation network, i.e., a small group of subscribers. While 100K messages a second is easy for a ØMQ application, we want to replay it much faster.

전달 데이터(feed)가 초당 평균 10만 개의 100 바이트 메시지들이 있다고 가정합니다.
시장 데이터를 필터링한 후 구독자에게 전송할 필요가 없게 하면 일반적인 비율입니다.
이제 하루의 데이터(대략 8시간에 250GB)를 저장하고, 시뮬레이션 네트워크(소규모 구독자들의 그룹)에서 재현하기로 합니다. ØMQ 응용프로그램에서 초당 10만 개의 메시지들의 처리는 쉽지만, 더 빠르게 처리하고 싶습니다.

;So we set up our architecture with a bunch of boxes—one for the publisher and one for each subscriber. These are well-specified boxes—eight cores, twelve for the publisher.

그래서 아키텍처상에서 발행자와 구독자를 위한 각각에 하나의 몪음으로 박스를 설치하였습니다.
설치된 컴퓨터는 8개 코어를 사용하며 12개 발행자를 대응합니다.

;And as we pump data into our subscribers, we notice two things:

그리고 구독자에게 데이터를 전달하면서 다음 2가지에 대하여 주목하게 됩니다.

; 1. When we do even the slightest amount of work with a message, it slows down our subscriber to the point where it can't catch up with the publisher again.
; 2. We're hitting a ceiling, at both publisher and subscriber, to around 6M messages a second, even after careful optimization and TCP tuning.

1. 우리가 메시지에 대한 일정 작업을 수행할 경우, 구독자는 처리 속도가 느려지면서 발행자가 전달하는 메시지들을 처리할 수 없게 됩니다.
2. 신중하게 최적화하고 TCP를 조정한 후에, 발행자와 구독자의 메시지 처리 한계는 초당 6백만개 입니다.

;The first thing we have to do is break our subscriber into a multithreaded design so that we can do work with messages in one set of threads, while reading messages in another. Typically, we don't want to process every message the same way. Rather, the subscriber will filter some messages, perhaps by prefix key. When a message matches some criteria, the subscriber will call a worker to deal with it. In ØMQ terms, this means sending the message to a worker thread.

가장 먼저 할 일은 구독자를 멀티 스레드 설계로 분할하여 한 스레드 집합에서는 메시지들의 처리를 하며 다른 스레드는 집합에서는 메시지를 읽게 합니다.
일반적으로 우리는 동일한 방식으로 모든 메시지 처리하고 싶지 않기에 구독자가 메시지에 대한 필터링(토픽(TOPIC))을 수행합니다. 메시지가 어떤 기준과 일치하면 구독자는 작업자를 호출하여 작업을 처리하게 합니다.
ØMQ 용어로 작업자 스레드에 메시지를 보내는 것을 의미합니다.

;So the subscriber looks something like a queue device. We could use various sockets to connect the subscriber and workers. If we assume one-way traffic and workers that are all identical, we can use PUSH and PULL and delegate all the routing work to ØMQ. This is the simplest and fastest approach.

따라서 구독자는 대기열 장치처럼 보입니다. 다양한 소켓을 사용하여 구독자와 작업자들을 연결할 수 있습니다. 단방향 통신과 작업자들 모두 동일하다고 가정하면 PUSH 및 PULL 소켓을 사용하고 모든 라우팅(작업자들에게 분배) 작업을 ØMQ에 위임할 수 있습니다. 이것은 가장 간단하고 빠른 접근 방식입니다.

;The subscriber talks to the publisher over TCP or PGM. The subscriber talks to its workers, which are all in the same process, over inproc://.

구독자는 TCP 또는 PGM을 통해 발행자와 통신하며, 구독자는 `inproc://`를 통해 하나의 프로세스에 있는 모두 작업자들(스레드들)과 통신합니다.

그림 57 - 미친 블랙박스 패턴

![Mad Black Box Pattern](images/fig57.svg)

;Now to break that ceiling. The subscriber thread hits 100% of CPU and because it is one thread, it cannot use more than one core. A single thread will always hit a ceiling, be it at 2M, 6M, or more messages per second. We want to split the work across multiple threads that can run in parallel.

이제 한계를 돌파해 봅시다. 구독자 스레드는 CPU의 100%에 도달한 이유는 하나의 스레드로 구성하여 2 이상의 CPU 코어를 사용할 수 없습니다. 
단일 스레드는 초당 2백만, 6백만 혹은 이상의 메시지들에서 처리 한계에 도달합니다.
작업을 분할하여 멀티스레드를 통한 병렬로 실행하겠습니다.

;The approach used by many high-performance products, which works here, is sharding. Using sharding, we split the work into parallel and independent streams, such as half of the topic keys in one stream, and half in another. We could use many streams, but performance won't scale unless we have free cores. So let's see how to shard into two streams.

이러한 접근은 많은 고성능 메시지 처리 제품에서 사용되고 있으며 샤딩(sharding)이라 합니다. 샤딩을 사용하여 작업을 병렬 및 독립 스트림으로 분할합니다(한 스트림에서 토픽 키의 절반, 다른 스트림에서 토픽 키의 절반).
많은 스트림을 사용할 수 있지만 유휴 CPU 코어가 없으면 성능이 확장되지 않습니다. 
이제 하나의 메시지 스트림을 2개의 메시지 스트림으로 분할하는 방법을 보겠습니다.
> [옮긴이] 병렬 샤딩(Parallel Sharding)은  데이터가 서로 공유되지 않도록 완전히 분리하여 독립적으로 동작하는 샤딩(Sharding)을 여러 개로 묶어서 병렬 처리를 통해 성능 및 확장성을 확보하는 방식입니다.

;With two streams, working at full speed, we would configure ØMQ as follows:

2개의 메시지 스트림들에 대하여 최고 속도로 작업하기 위하여 ØMQ을 다음과 같이 구성합니다.

;* Two I/O threads, rather than one.
;* Two network interfaces (NIC), one per subscriber.
;* Each I/O thread bound to a specific NIC.
;* Two subscriber threads, bound to specific cores.
;* Two SUB sockets, one per subscriber thread.
;* The remaining cores assigned to worker threads.
;* Worker threads connected to both subscriber PUSH sockets.

* 하나가 아닌 2개의 I/O 스레드들.
* 2개의 네트워크 인터페이스(NIC), 구독자 당 하나씩.
* 개별 I/O 스레드는 지정된 네트워크 인터페이스(NIC)에 바인딩
* 2개의 구독자 스레드는 지정된 CPU 코어들에 바인딩.
* 2개의 SUB 소켓은 구독자 스레드당 하나씩 지정.
* 나머지 CPU 코어들은 작업자 스레드들에 할당.
* 작업자 스레드들은 가입자의 2개의 PUSH 소켓들에 연결

;Ideally, we want to match the number of fully-loaded threads in our architecture with the number of cores. When threads start to fight for cores and CPU cycles, the cost of adding more threads outweighs the benefits. There would be no benefit, for example, in creating more I/O threads.

이상적으로는 아키텍처에서 완전히 부하 처리가 가능한 스레드 수를 코어 수와 일치시키려고 했습니다. 스레드가 코어 및 CPU 주기를 놓고 싸우기(Time sharing) 시작하면 스레드를 추가하는 비용이 메시지 처리량보다 큽니다(더 많은 메시지 처리하기 위해 더 많은 CPU 코어 구매). 더 많은 I/O 스레드를 생성하는 것에는 이점이 없습니다.

> [옮긴이] 고속 구독자에 대한 테스트를 위한 예제입니다.

hssub.c : high speed subscriber
```cpp
// high speed subscriber

#include "czmq.h"
#define NBR_WORKERS 3

//  .split publisher task
static void
publisher (void *args, zctx_t *ctx, void *pipe)
{
    //  Prepare publisher
    void *publisher = zsocket_new (ctx, ZMQ_PUB);
    zsocket_bind (publisher, "tcp://*:5556");

    while (true) {
        //  Send current clock (msecs) to subscribers
        char string [20];
        sprintf (string, "%" PRId64, zclock_time ());
        zstr_send (publisher, string);
        char *signal = zstr_recv_nowait (pipe);
        if (signal) {
            free (signal);
            break;
        }
        zclock_sleep (1);            //  1msec wait
    }
    
}

// subscriber task
static void
subscriber (void *args, zctx_t *ctx, void *pipe)
{
    //  Subscribe to everything
    void *subscriber = zsocket_new (ctx, ZMQ_SUB);
    zsocket_set_subscribe (subscriber, "");
    zsocket_connect (subscriber, "tcp://localhost:5556");
    
    //  Socket to send messages to
    void *sender = zsocket_new (ctx, ZMQ_PUSH);
    zsocket_bind (sender, "tcp://*:5557");

    //  Get and send messages
    while (true) {
        char *topic = zstr_recv (subscriber);
        zstr_send(sender, topic);
        free (topic);
    }
}
// worker task
static void
worker(void *args, zctx_t *ctx, void *pipe)
{
    //  Subscribe to everything
    void *worker = zsocket_new (ctx, ZMQ_PULL);
    zsocket_connect (worker, "tcp://localhost:5557");

    //  Get and send messages
    while (true) {
        char *string = zstr_recv (worker);
        printf ("[W%Id]Receive: [%s]\n", (intptr_t)args, string);
        free (string);
    }
}

int main (void)
{
	zctx_t *ctx = zctx_new ();
    void *pubpipe = zthread_fork (ctx, publisher, NULL);
    void *subpipe = zthread_fork (ctx, subscriber, NULL); 
	int worker_nbr;
	for (worker_nbr = 0; worker_nbr < NBR_WORKERS; worker_nbr++){
        zthread_fork (ctx, worker, (void *)(intptr_t)worker_nbr);
		//zclock_sleep(100);
	}
    free (zstr_recv (subpipe));
    zstr_send (pubpipe, "break");
    zclock_sleep (100);
    zctx_destroy (&ctx);
    return 0;
}
```
> [옮긴이] 빌드 및 테스트
 - 3개의 작업자들에게 라운드로빈 형태로 메시지가 전달되는 것을 확인 가능합니다.

~~~{.bash}
PS D:\git_store\zguide-kr\examples\C> cl -EHsc hssub.c libzmq.lib czmq.lib

PS D:\git_store\zguide-kr\examples\C> ./hssub
[W0]Receive: [13242984839620]
[W1]Receive: [13242984839622]
[W2]Receive: [13242984839624]
[W0]Receive: [13242984839626]
[W1]Receive: [13242984839628]
[W2]Receive: [13242984839630]
[W0]Receive: [13242984839632]
[W1]Receive: [13242984839634]
[W2]Receive: [13242984839636]
...
~~~

;## Reliable Pub-Sub (Clone Pattern)

## 신뢰할 수 있는 발행-구독 (복제 패턴)

;As a larger worked example, we'll take the problem of making a reliable pub-sub architecture. We'll develop this in stages. The goal is to allow a set of applications to share some common state. Here are our technical challenges:

좀 더 큰 작업 예제로, 신뢰할 수 있는 발행-구독 아키텍처를 만드는 문제를 보겠습니다. 
우리는 이것을 단계적으로 개발하며 목표는 일련의 응용프로그램들이 일부 공통 상태를 공유하게 합니다. 기술적 도전 과제들은 다음과 같습니다.

;* We have a large set of client applications, say thousands or tens of thousands.
;* They will join and leave the network arbitrarily.
;* These applications must share a single eventually-consistent state.
;* Any application can update the state at any point in time.

* 수천 또는 수만 개의 클라이언트 응용프로그램들이 있습니다.
* 임의로 네트워크에 가입하고 탈퇴합니다.
* 이러한 응용프로그램들은 하나의 최종 일관성 상태를 공유해야 합니다.
* 모든 응용프로그램은 언제든지 상태를 변경할 수 있습니다.

;Let's say that updates are reasonably low-volume. We don't have real time goals. The whole state can fit into memory. Some plausible use cases are:

상태의 변경이 상당히 적은 양이라고 가정하고 실시간 처리 목표는 없습니다. 
전체 상태는 메모리에 저장하고, 일부 사용 사례는 다음과 같습니다.

;* A configuration that is shared by a group of cloud servers.
;* Some game state shared by a group of players.
;* Exchange rate data that is updated in real time and available to applications.

* 클라우드 서버들의 그룹에서 공유하는 구성.
* 플레이어 그룹이 공유하는 일부 게임 상태.
* 환율 데이터가 실시간으로 변경되고 응용프로그램에서 가용함.

;### Centralized Versus Decentralized

### 중앙집중형과 분산형

;A first decision we have to make is whether we work with a central server or not. It makes a big difference in the resulting design. The trade-offs are these:

우리가 해야 할 첫 번째 결정은 중앙 서버 혹은 분산 서버로 작업할지 여부입니다. 
결과적인 설계에서 큰 차이를 만들며, 장단점은 다음과 같습니다.

;* Conceptually, a central server is simpler to understand because networks are not naturally symmetrical. With a central server, we avoid all questions of discovery, bind versus connect, and so on.
;* Generally, a fully-distributed architecture is technically more challenging but ends up with simpler protocols. That is, each node must act as server and client in the right way, which is delicate. When done right, the results are simpler than using a central server. We saw this in the Freelance pattern in Chapter 4 - Reliable Request-Reply Patterns.
;* A central server will become a bottleneck in high-volume use cases. If handling scale in the order of millions of messages a second is required, we should aim for decentralization right away.
;* Ironically, a centralized architecture will scale to more nodes more easily than a decentralized one. That is, it's easier to connect 10,000 nodes to one server than to each other.

* 개념적으로 중앙 서버로 작업하는 것이 네트워크가 태생적으로 비대칭형이기 때문에 이해하기 더 쉽습니다. 중앙 서버를 사용하면 검색, 바인딩과 연결 등과 관련된 모든 질문들을 피할 수 있습니다.
* 일반적으로 완전히 분산된 아키텍처는 기술적으로 더욱 어렵지만 의외로 간단한 통신규약으로 끝납니다. 즉, 각 노드가 올바른 방식으로 서버 및 클라이언트 역할을 해야 하며 이는 섬세합니다. 올바르게 수행하면 중앙 서버를 사용하는 것보다 결과는 더 간단합니다. "4장 - 신뢰할 수 있는 요청-응답 패턴"의 프리랜서 패턴에서 보았습니다.
* 중앙 서버는 대량 데이터 사용 사례에서 병목 현상을 겪을 수 있습니다. 초당 수백만 개의 메시지를 처리해야 한다면, 당장 분산 처리 환경을 목표로 해야 합니다.
* 역설적이게도 중앙집중식 아키텍처는 분산형 아키텍처보다 더 쉽게 더 많은 노드로 확장됩니다. 즉, 하나의 서버에 10,000개의 노드들을 연결하는 것이 노드들 간에 상호 연결하는 것보다 쉽습니다.

;So, for the Clone pattern we'll work with a server that publishes state updates and a set of clients that represent applications.

그래서 복제 패턴의 경우 중앙 서버를 통해 상태 변경을 전송하면 일련의 클라이언트들의 응용프로그램에서 사용합니다.

;### Representing State as Key-Value Pairs

### 상태를 키-값 쌍으로 표시

;We'll develop Clone in stages, solving one problem at a time. First, let's look at how to update a shared state across a set of clients. We need to decide how to represent our state, as well as the updates. The simplest plausible format is a key-value store, where one key-value pair represents an atomic unit of change in the shared state.

우리는 한 번에 하나씩 문제를 해결하면서 단계적으로 복제(Clone)를 개발할 것입니다. 먼저 일련의 클라이언트들에서 공유 상태를 변경하는 방법을 보겠습니다. 우리는 상태를 표현하고 변경한 방법을 결정해야 합니다. 가장 단순한 형식은 키-값 저장소(해시 테이블)로 하나의 키-값 쌍은 공유 상태를 변경하는 기본 단위가 됩니다.

;We have a simple pub-sub example in Chapter 1 - Basics, the weather server and client. Let's change the server to send key-value pairs, and the client to store these in a hash table. This lets us send updates from one server to a set of clients using the classic pub-sub model.

"1장 - 기본"에서 날씨 서버 및 클라이언트 예제로 간단한 발행-구독 패턴으로 구현하였습니다. 이제 서버(wuserver)를 변경하여 키-값 쌍을 보내고 클라이언트에서 해시 테이블에 저장하겠습니다. 이를 통해 고전적인 발행-구독 모델을 사용하여 한 서버에서 일련의 클라이언트들로 변경정보를 보낼 수 있습니다.

;An update is either a new key-value pair, a modified value for an existing key, or a deleted key. We can assume for now that the whole store fits in memory and that applications access it by key, such as by using a hash table or dictionary. For larger stores and some kind of persistence we'd probably store the state in a database, but that's not relevant here.

변경정보는 신규 키-값 쌍이거나 기존 키의 수정된 값 혹은 삭제된 키입니다. 
지금은 전체 저장소가 메모리에 저장할 수 있고 응용프로그램들이 해시 테이블이나 사전을 사용하는 것처럼 키로 접근한다고 가정합니다. 더 큰 저장소와 일종의 지속성이 필요한 경우 상태를 데이터베이스에 저장할 수 있지만 여기서는 관련이 없습니다.

;This is the server:

다음은 서버 코드입니다.

clonesrv1.c: Clone server, Model One in C
```cpp
//  Clone server Model One

#include "kvsimple.c"

int main (void)
{
    //  Prepare our context and publisher socket
    zctx_t *ctx = zctx_new ();
    void *publisher = zsocket_new (ctx, ZMQ_PUB);
    zsocket_bind (publisher, "tcp://*:5556");
    zclock_sleep (200);

    zhash_t *kvmap = zhash_new ();
    int64_t sequence = 0;
    srandom ((unsigned) time (NULL));

    while (!zctx_interrupted) {
        //  Distribute as key-value message
        kvmsg_t *kvmsg = kvmsg_new (++sequence);
        kvmsg_fmt_key  (kvmsg, "%d", randof (10000));
        kvmsg_fmt_body (kvmsg, "%d", randof (1000000));
        kvmsg_send     (kvmsg, publisher);
        kvmsg_store   (&kvmsg, kvmap);
    }
    printf (" Interrupted\n%d messages out\n", (int) sequence);
    zhash_destroy (&kvmap);
    zctx_destroy (&ctx);
    return 0;
}
```

;And here is the client:

다음은 클라이언트 코드입니다.

clonecli1: Clone client, Model One in C
```cpp
//  Clone client Model One

#include "kvsimple.c"

int main (void)
{
    //  Prepare our context and updates socket
    zctx_t *ctx = zctx_new ();
    void *updates = zsocket_new (ctx, ZMQ_SUB);
    zsocket_set_subscribe (updates, "");
    zsocket_connect (updates, "tcp://localhost:5556");

    zhash_t *kvmap = zhash_new ();
    int64_t sequence = 0;

    while (!zctx_interrupted) {
        kvmsg_t *kvmsg = kvmsg_recv (updates);
        if (!kvmsg)
            break;          //  Interrupted
        kvmsg_store (&kvmsg, kvmap);
        sequence++;
    }
    printf (" Interrupted\n%d messages in\n", (int) sequence);
    zhash_destroy (&kvmap);
    zctx_destroy (&ctx);
    return 0;
}
```
> [옮긴이] clonecli.c에서 `while(true)`에 대하여 사용자 인터럽트를 받을 수 있도록 
`while (!zctx_interrupted)`로 변경하였습니다.

> [옮긴이] 빌드 및 테스트

~~~{.bash}
PS D:\git_store\zguide-kr\examples\C> cl -EHsc clonesrv1.c libzmq.lib czmq.lib
PS D:\git_store\zguide-kr\examples\C> cl -EHsc clonecli1.c libzmq.lib czmq.lib

PS D:\git_store\zguide-kr\examples\C> ./clonesrv1
 Interrupted
14938939 messages out

PS D:\git_store\zguide-kr\examples\C> ./clonecli1
 Interrupted
1119580 messages in
~~~

Figure 58 - 상태 변경정보 발행

![Publishing State Updates](images/fig58.svg)

;Here are some things to note about this first model:

첫 번째 모델에 대한 몇 가지 주목할 점은 다음과 같습니다.

;* All the hard work is done in a kvmsg class. This class works with key-value message objects, which are multipart ØMQ messages structured as three frames: a key (a ØMQ string), a sequence number (64-bit value, in network byte order), and a binary body (holds everything else).
;* The server generates messages with a randomized 4-digit key, which lets us simulate a large but not enormous hash table (10K entries).
;* We don't implement deletions in this version: all messages are inserts or updates.
;* The server does a 200 millisecond pause after binding its socket. This is to prevent slow joiner syndrome, where the subscriber loses messages as it connects to the server's socket. We'll remove that in later versions of the Clone code.
;* We'll use the terms publisher and subscriber in the code to refer to sockets. This will help later when we have multiple sockets doing different things.

* 모든 어려운 작업은 `kvmsg` 클래스에서 수행되었습니다. 이 클래스는 키-값 메시지 객체들로 동작하며, 멀티파트(multipart) ØMQ 메시지는 3개의 프레임으로 구성됩니다 : 키(ØMQ 문자열), 순서 번호(네트워크 바이트 순서의 64 비트 값(`int64_t`)), 바이너리 본문(모든 항목을 포함)
* 서버는 임의의 4자리 키(0~9999)로 메시지를 생성하므로, 크지만 방대하지는 않은 해시 테이블(1만 개 항목들)을 시뮬레이션할 수 있습니다.
* 현재 버전에서는 삭제를 구현하지 않습니다. 모든 메시지는 삽입 또는 변경입니다.
* 서버는 소켓을 바인딩하고 200 밀리초 동안 일시 중지합니다. 이는 구독자가 서버의 소켓에 연결할 때 메시지를 유실하는 느린 참여 증후군(slow joiner syndrome)을 방지합니다. 이후 버전의 복제 코드에서는 제거하겠습니다.
* 소켓에 대하여 코드상에서 발행자와 구독자라는 용어를 사용할 것입니다. 이것은 나중에 다른 일을 하는 다중 소켓들로 작업할 때 도움이 됩니다.

;Here is the kvmsg class, in the simplest form that works for now:

다음은 현재 동작하는 가장 간단한 형식의 kvmsg 클래스입니다.

kvsimple.c : Key-value message class in C
```cpp
//  kvsimple class - key-value message class for example applications

#include "kvsimple.h"
#include "zlist.h"

//  Keys are short strings
#define KVMSG_KEY_MAX   255

//  Message is formatted on wire as 3 frames:
//  frame 0: key (ØMQ string)
//  frame 1: sequence (8 bytes, network order)
//  frame 2: body (blob)
#define FRAME_KEY       0
#define FRAME_SEQ       1
#define FRAME_BODY      2
#define KVMSG_FRAMES    3

//  The kvmsg class holds a single key-value message consisting of a
//  list of 0 or more frames:

struct _kvmsg {
    //  Presence indicators for each frame
    int present [KVMSG_FRAMES];
    //  Corresponding ØMQ message frames, if any
    zmq_msg_t frame [KVMSG_FRAMES];
    //  Key, copied into safe C string
    char key [KVMSG_KEY_MAX + 1];
};

//  .split constructor and destructor
//  Here are the constructor and destructor for the class:

//  Constructor, takes a sequence number for the new kvmsg instance:
kvmsg_t *
kvmsg_new (int64_t sequence)
{
    kvmsg_t
        *self;

    self = (kvmsg_t *) zmalloc (sizeof (kvmsg_t));
    kvmsg_set_sequence (self, sequence);
    return self;
}

//  zhash_free_fn callback helper that does the low level destruction:
void
kvmsg_free (void *ptr)
{
    if (ptr) {
        kvmsg_t *self = (kvmsg_t *) ptr;
        //  Destroy message frames if any
        int frame_nbr;
        for (frame_nbr = 0; frame_nbr < KVMSG_FRAMES; frame_nbr++)
            if (self->present [frame_nbr])
                zmq_msg_close (&self->frame [frame_nbr]);

        //  Free object itself
        free (self);
    }
}

//  Destructor
void
kvmsg_destroy (kvmsg_t **self_p)
{
    assert (self_p);
    if (*self_p) {
        kvmsg_free (*self_p);
        *self_p = NULL;
    }
}

//  .split recv method
//  This method reads a key-value message from socket, and returns a new
//  {{kvmsg}} instance:

kvmsg_t *
kvmsg_recv (void *socket)
{
    assert (socket);
    kvmsg_t *self = kvmsg_new (0);

    //  Read all frames off the wire, reject if bogus
    int frame_nbr;
    for (frame_nbr = 0; frame_nbr < KVMSG_FRAMES; frame_nbr++) {
        if (self->present [frame_nbr])
            zmq_msg_close (&self->frame [frame_nbr]);
        zmq_msg_init (&self->frame [frame_nbr]);
        self->present [frame_nbr] = 1;
        if (zmq_msg_recv (&self->frame [frame_nbr], socket, 0) == -1) {
            kvmsg_destroy (&self);
            break;
        }
        //  Verify multipart framing
        int rcvmore = (frame_nbr < KVMSG_FRAMES - 1)? 1: 0;
        if (zsocket_rcvmore (socket) != rcvmore) {
            kvmsg_destroy (&self);
            break;
        }
    }
    return self;
}

//  .split send method
//  This method sends a multiframe key-value message to a socket:

void
kvmsg_send (kvmsg_t *self, void *socket)
{
    assert (self);
    assert (socket);

    int frame_nbr;
    for (frame_nbr = 0; frame_nbr < KVMSG_FRAMES; frame_nbr++) {
        zmq_msg_t copy;
        zmq_msg_init (&copy);
        if (self->present [frame_nbr])
            zmq_msg_copy (&copy, &self->frame [frame_nbr]);
        zmq_msg_send (&copy, socket, 
            (frame_nbr < KVMSG_FRAMES - 1)? ZMQ_SNDMORE: 0);
        zmq_msg_close (&copy);
    }
}

//  .split key methods
//  These methods let the caller get and set the message key, as a
//  fixed string and as a printf formatted string:

char *
kvmsg_key (kvmsg_t *self)
{
    assert (self);
    if (self->present [FRAME_KEY]) {
        if (!*self->key) {
            size_t size = zmq_msg_size (&self->frame [FRAME_KEY]);
            if (size > KVMSG_KEY_MAX)
                size = KVMSG_KEY_MAX;
            memcpy (self->key,
                zmq_msg_data (&self->frame [FRAME_KEY]), size);
            self->key [size] = 0;
        }
        return self->key;
    }
    else
        return NULL;
}

void
kvmsg_set_key (kvmsg_t *self, char *key)
{
    assert (self);
    zmq_msg_t *msg = &self->frame [FRAME_KEY];
    if (self->present [FRAME_KEY])
        zmq_msg_close (msg);
    zmq_msg_init_size (msg, strlen (key));
    memcpy (zmq_msg_data (msg), key, strlen (key));
    self->present [FRAME_KEY] = 1;
}

void
kvmsg_fmt_key (kvmsg_t *self, char *format, ...)
{
    char value [KVMSG_KEY_MAX + 1];
    va_list args;

    assert (self);
    va_start (args, format);
    vsnprintf (value, KVMSG_KEY_MAX, format, args);
    va_end (args);
    kvmsg_set_key (self, value);
}

//  .split sequence methods
//  These two methods let the caller get and set the message sequence number:

int64_t
kvmsg_sequence (kvmsg_t *self)
{
    assert (self);
    if (self->present [FRAME_SEQ]) {
        assert (zmq_msg_size (&self->frame [FRAME_SEQ]) == 8);
        byte *source = zmq_msg_data (&self->frame [FRAME_SEQ]);
        int64_t sequence = ((int64_t) (source [0]) << 56)
                         + ((int64_t) (source [1]) << 48)
                         + ((int64_t) (source [2]) << 40)
                         + ((int64_t) (source [3]) << 32)
                         + ((int64_t) (source [4]) << 24)
                         + ((int64_t) (source [5]) << 16)
                         + ((int64_t) (source [6]) << 8)
                         +  (int64_t) (source [7]);
        return sequence;
    }
    else
        return 0;
}

void
kvmsg_set_sequence (kvmsg_t *self, int64_t sequence)
{
    assert (self);
    zmq_msg_t *msg = &self->frame [FRAME_SEQ];
    if (self->present [FRAME_SEQ])
        zmq_msg_close (msg);
    zmq_msg_init_size (msg, 8);

    byte *source = zmq_msg_data (msg);
    source [0] = (byte) ((sequence >> 56) & 255);
    source [1] = (byte) ((sequence >> 48) & 255);
    source [2] = (byte) ((sequence >> 40) & 255);
    source [3] = (byte) ((sequence >> 32) & 255);
    source [4] = (byte) ((sequence >> 24) & 255);
    source [5] = (byte) ((sequence >> 16) & 255);
    source [6] = (byte) ((sequence >> 8)  & 255);
    source [7] = (byte) ((sequence)       & 255);

    self->present [FRAME_SEQ] = 1;
}

//  .split message body methods
//  These methods let the caller get and set the message body as a
//  fixed string and as a printf formatted string:

byte *
kvmsg_body (kvmsg_t *self)
{
    assert (self);
    if (self->present [FRAME_BODY])
        return (byte *) zmq_msg_data (&self->frame [FRAME_BODY]);
    else
        return NULL;
}

void
kvmsg_set_body (kvmsg_t *self, byte *body, size_t size)
{
    assert (self);
    zmq_msg_t *msg = &self->frame [FRAME_BODY];
    if (self->present [FRAME_BODY])
        zmq_msg_close (msg);
    self->present [FRAME_BODY] = 1;
    zmq_msg_init_size (msg, size);
    memcpy (zmq_msg_data (msg), body, size);
}

void
kvmsg_fmt_body (kvmsg_t *self, char *format, ...)
{
    char value [255 + 1];
    va_list args;

    assert (self);
    va_start (args, format);
    vsnprintf (value, 255, format, args);
    va_end (args);
    kvmsg_set_body (self, (byte *) value, strlen (value));
}

//  .split size method
//  This method returns the body size of the most recently read message,
//  if any exists:

size_t
kvmsg_size (kvmsg_t *self)
{
    assert (self);
    if (self->present [FRAME_BODY])
        return zmq_msg_size (&self->frame [FRAME_BODY]);
    else
        return 0;
}

//  .split store method
//  This method stores the key-value message into a hash map, unless
//  the key and value are both null. It nullifies the {{kvmsg}} reference
//  so that the object is owned by the hash map, not the caller:

void
kvmsg_store (kvmsg_t **self_p, zhash_t *hash)
{
    assert (self_p);
    if (*self_p) {
        kvmsg_t *self = *self_p;
        assert (self);
        if (self->present [FRAME_KEY]
        &&  self->present [FRAME_BODY]) {
            zhash_update (hash, kvmsg_key (self), self);
            zhash_freefn (hash, kvmsg_key (self), kvmsg_free);
        }
        *self_p = NULL;
    }
}

//  .split dump method
//  This method prints the key-value message to stderr for
//  debugging and tracing:

void
kvmsg_dump (kvmsg_t *self)
{
    if (self) {
        if (!self) {
            fprintf (stderr, "NULL");
            return;
        }
        size_t size = kvmsg_size (self);
        byte  *body = kvmsg_body (self);
        fprintf (stderr, "[seq:%" PRId64 "]", kvmsg_sequence (self));
        fprintf (stderr, "[key:%s]", kvmsg_key (self));
        fprintf (stderr, "[size:%zd] ", size);
        int char_nbr;
        for (char_nbr = 0; char_nbr < size; char_nbr++)
            fprintf (stderr, "%02X", body [char_nbr]);
        fprintf (stderr, "\n");
    }
    else
        fprintf (stderr, "NULL message\n");
}

//  .split test method
//  It's good practice to have a self-test method that tests the class; this
//  also shows how it's used in applications:

int
kvmsg_test (int verbose)
{
    kvmsg_t
        *kvmsg;

    printf (" * kvmsg: ");

    //  Prepare our context and sockets
    zctx_t *ctx = zctx_new ();
    void *output = zsocket_new (ctx, ZMQ_DEALER);
    int rc = zmq_bind (output, "inproc://kvmsg_selftest");
    assert (rc == 0);
    void *input = zsocket_new (ctx, ZMQ_DEALER);
    rc = zmq_connect (input, "inproc://kvmsg_selftest");
    assert (rc == 0);

    zhash_t *kvmap = zhash_new ();

    //  Test send and receive of simple message
    kvmsg = kvmsg_new (1);
    kvmsg_set_key  (kvmsg, "key");
    kvmsg_set_body (kvmsg, (byte *) "body", 4);
    if (verbose)
        kvmsg_dump (kvmsg);
    kvmsg_send (kvmsg, output);
    kvmsg_store (&kvmsg, kvmap);

    kvmsg = kvmsg_recv (input);
    if (verbose)
        kvmsg_dump (kvmsg);
    assert (streq (kvmsg_key (kvmsg), "key"));
    kvmsg_store (&kvmsg, kvmap);

    //  Shutdown and destroy all objects
    zhash_destroy (&kvmap);
    zctx_destroy (&ctx);

    printf ("OK\n");
    return 0;
}
```
> [옮긴이] "kvsimpler.c"의 `kvmsg_test()`에서 ipc를 사용하고 있으나 원도우 환경에서는 동작할 수 없어 inproc로 변경하여 테스트를 수행합니다.(inproc는 원도우 및 Linux에서 동작 가능)

```cpp
// 변경전
    int rc = zmq_bind (output, "ipc://kvmsg_selftest.ipc");
    assert (rc == 0);
    void *input = zsocket_new (ctx, ZMQ_DEALER);
    rc = zmq_connect (input, "ipc://kvmsg_selftest.ipc");
// 변경후
    int rc = zmq_bind (output, "inproc://kvmsg_selftest");
    assert (rc == 0);
    void *input = zsocket_new (ctx, ZMQ_DEALER);
    rc = zmq_connect (input, "inproc://kvmsg_selftest");
```
> [옮긴이] "kvsimpler"에 대한 테스트를 수행하기 위한 "kvsimtest.c"는 다음과 같습니다.

```cpp
#include "kvsimple.c"

void main()
{
    kvmsg_test(1);
}
```
> [옮긴이] 빌드 및 테스트

~~~{.bash}
PS D:\git_store\zguide-kr\examples\C> cl -EHsc kvsimtest.c libzmq.lib czmq.lib

PS D:\git_store\zguide-kr\examples\C> ./kvsimtest
 * kvmsg: [seq:1][key:key][size:4] 626F6479
[seq:1][key:key][size:4] 626F6479
OK
~~~

;Later, we'll make a more sophisticated kvmsg class that will work in real applications.

나중에 실제 응용프로그램에서 동작하는 보다 정교한 kvmsg 클래스를 만들겠습니다.

;Both the server and client maintain hash tables, but this first model only works properly if we start all clients before the server and the clients never crash. That's very artificial.

서버와 클라이언트 모두 해시 테이블을 유지하지만, 첫 번째 모델은 서버를 시작하기 전에 모든 클라이언트들을 구동하고 클라이언트들이 충돌하지 않는 경우에만 제대로 작동합니다. 
그것은 매우 인위적입니다.

;### Getting an Out-of-Band Snapshot

### 대역 외 스냅샷 얻기

;So now we have our second problem: how to deal with late-joining clients or clients that crash and then restart.

이제 두 번째 문제가 있습니다. 늦게 가입하는 클라이언트들과 혹은 장애조치 이후 재시작한 클라이언트들을 처리하는 방법입니다.

;In order to allow a late (or recovering) client to catch up with a server, it has to get a snapshot of the server's state. Just as we've reduced "message" to mean "a sequenced key-value pair", we can reduce "state" to mean "a hash table". To get the server state, a client opens a DEALER socket and asks for it explicitly.

늦게 가입(또는 재시작)한 클라이언트들에 대하여 서버가 이미 발행한 메시지들을 받을 수 있게 하려면 서버 상태의 스냅샷을 얻어야 합니다. "메시지"의 의미를 "순서화된 키-값 쌍"으로 줄인 것처럼 "상태 정보"의 의미도 "해시 테이블" 줄일 수 있습니다. 서버 상태 정보를 얻기 위해 클라이언트는 DEALER 소켓을 열고 명시적으로 요청합니다.

;To make this work, we have to solve a problem of timing. Getting a state snapshot will take a certain time, possibly fairly long if the snapshot is large. We need to correctly apply updates to the snapshot. But the server won't know when to start sending us updates. One way would be to start subscribing, get a first update, and then ask for "state for update N". This would require the server storing one snapshot for each update, which isn't practical.

이 작업을 수행하기 위해 타이밍 문제를 해결해야 합니다. 상태 스냅샷을 얻기는 일정 시간이 걸리며 스냅샷이 크면 상당히 오래 걸릴 수 있습니다. 
스냅샷에 올바르게 변경정보를 적용해야 하지만 서버는 변경정보 전송을 언제 시작할지 모릅니다.
한 가지 방법은 클라이언트가 구독을 시작하고 첫 번째 변경정보를 받은 다음 "변경정보 N에 대한 상태(state for update N)"를 요청하는 것입니다. 이렇게 하려면 서버가 각 변경정보에 대해 하나의 스냅샷을 저장해야 하므로 실용적이지 않습니다.

그림 59 - 상태 복제

![State Replication](images/fig59.svg)

;So we will do the synchronization in the client, as follows:

따라서 클라이언트의 동기화는 다음과 같이 수행합니다.

;* The client first subscribes to updates and then makes a state request. This guarantees that the state is going to be newer than the oldest update it has.
;* The client waits for the server to reply with state, and meanwhile queues all updates. It does this simply by not reading them: ØMQ keeps them queued on the socket queue.
;* When the client receives its state update, it begins once again to read updates. However, it discards any updates that are older than the state update. So if the state update includes updates up to 200, the client will discard updates up to 201.
;* The client then applies updates to its own state snapshot.

* 클라이언트는 먼저 변경정보들을 구독한 다음 상태 요청을 하면, 상태가 이전 오래된 변경정보보다 최신이 됩니다. 
* 클라이언트는 서버가 상태로 응답할 때까지 대기하고 모든 변경정보를 대기열에 넣습니다. 
변경정보를 대기열에 넣기만 하고 처리하지 않도록 하는 것은 대기열을 읽지 않음으로써 가능합니다 : ØMQ는 변경정보들을 소켓 대기열에 넣어 보관합니다.
* 클라이언트가 상태 변경을 받으면 대기열에 넣어 두었던 변경정보 읽기를 다시 시작합니다. 그러나 상태 변경 시점보다 오래된 변경정보들은 모두 삭제됩니다. 따라서 상태 변경 전에 대기열에 최대 200 개의 변경정보가 포함된 경우 클라이언트는 최대 201개의 변경정보를 삭제합니다.
* 그런 다음 클라이언트는 자체 상태 스냅샷에 변경정보를 적용합니다.

;It's a simple model that exploits ØMQ's own internal queues. Here's the server:

다음의 서버 예제는 ØMQ의 자체 내부 대기열을 이용하는 단순한 모델입니다. 

clonesrv2.c : Clone server, Model Two in C
```cpp
//  Clone server - Model Two

//  Lets us build this source without creating a library
#include "kvsimple.c"

static int s_send_single (const char *key, void *data, void *args);
static void state_manager (void *args, zctx_t *ctx, void *pipe);

int main (void)
{
    //  Prepare our context and sockets
    zctx_t *ctx = zctx_new ();
    void *publisher = zsocket_new (ctx, ZMQ_PUB);
    zsocket_bind (publisher, "tcp://*:5557");

    int64_t sequence = 0;
    srandom ((unsigned) time (NULL));

    //  Start state manager and wait for synchronization signal
    void *updates = zthread_fork (ctx, state_manager, NULL);
    free (zstr_recv (updates));

    while (!zctx_interrupted) {
        //  Distribute as key-value message
        kvmsg_t *kvmsg = kvmsg_new (++sequence);
        kvmsg_fmt_key  (kvmsg, "%d", randof (10000));
        kvmsg_fmt_body (kvmsg, "%d", randof (1000000));
        kvmsg_send     (kvmsg, publisher);
        kvmsg_send     (kvmsg, updates);
        kvmsg_destroy (&kvmsg);
    }
    printf (" Interrupted\n%d messages out\n", (int) sequence);
    zctx_destroy (&ctx);
    return 0;
}

//  Routing information for a key-value snapshot
typedef struct {
    void *socket;           //  ROUTER socket to send to
    zframe_t *identity;     //  Identity of peer who requested state
} kvroute_t;

//  Send one state snapshot key-value pair to a socket
//  Hash item data is our kvmsg object, ready to send
static int
s_send_single (const char *key, void *data, void *args)
{
    kvroute_t *kvroute = (kvroute_t *) args;
    //  Send identity of recipient first
    zframe_send (&kvroute->identity,
        kvroute->socket, ZFRAME_MORE + ZFRAME_REUSE);
    kvmsg_t *kvmsg = (kvmsg_t *) data;
    kvmsg_send (kvmsg, kvroute->socket);
    return 0;
}

//  .split state manager
//  The state manager task maintains the state and handles requests from
//  clients for snapshots:

static void
state_manager (void *args, zctx_t *ctx, void *pipe)
{
    zhash_t *kvmap = zhash_new ();

    zstr_send (pipe, "READY");
    void *snapshot = zsocket_new (ctx, ZMQ_ROUTER);
    zsocket_bind (snapshot, "tcp://*:5556");

    zmq_pollitem_t items [] = {
        { pipe, 0, ZMQ_POLLIN, 0 },
        { snapshot, 0, ZMQ_POLLIN, 0 }
    };
    int64_t sequence = 0;       //  Current snapshot version number
    while (!zctx_interrupted) {
        int rc = zmq_poll (items, 2, -1);
        if (rc == -1 && errno == ETERM)
            break;              //  Context has been shut down

        //  Apply state update from main thread
        if (items [0].revents & ZMQ_POLLIN) {
            kvmsg_t *kvmsg = kvmsg_recv (pipe);
            if (!kvmsg)
                break;          //  Interrupted
            sequence = kvmsg_sequence (kvmsg);
            kvmsg_store (&kvmsg, kvmap);
        }
        //  Execute state snapshot request
        if (items [1].revents & ZMQ_POLLIN) {
            zframe_t *identity = zframe_recv (snapshot);
            if (!identity)
                break;          //  Interrupted

            //  Request is in second frame of message
            char *request = zstr_recv (snapshot);
            if (streq (request, "ICANHAZ?"))
                free (request);
            else {
                printf ("E: bad request, aborting\n");
                break;
            }
            //  Send state snapshot to client
            kvroute_t routing = { snapshot, identity };

            //  For each entry in kvmap, send kvmsg to client
            zhash_foreach (kvmap, s_send_single, &routing);

            //  Now send END message with sequence number
            printf ("Sending state shapshot=%d\n", (int) sequence);
            zframe_send (&identity, snapshot, ZFRAME_MORE);
            kvmsg_t *kvmsg = kvmsg_new (sequence);
            kvmsg_set_key  (kvmsg, "KTHXBAI");
            kvmsg_set_body (kvmsg, (byte *) "", 0);
            kvmsg_send     (kvmsg, snapshot);
            kvmsg_destroy (&kvmsg);
        }
    }
    zhash_destroy (&kvmap);
}
```

;And here is the client:

clonecli2.c : Clone client, Model Two in C
```cpp
//  Clone client - Model Two

//  Lets us build this source without creating a library
#include "kvsimple.c"

int main (void)
{
    //  Prepare our context and subscriber
    zctx_t *ctx = zctx_new ();
    void *snapshot = zsocket_new (ctx, ZMQ_DEALER);
    zsocket_connect (snapshot, "tcp://localhost:5556");
    void *subscriber = zsocket_new (ctx, ZMQ_SUB);
    zsocket_set_subscribe (subscriber, "");
    zsocket_connect (subscriber, "tcp://localhost:5557");

    zhash_t *kvmap = zhash_new ();

    //  Get state snapshot
    int64_t sequence = 0;
    zstr_send (snapshot, "ICANHAZ?");
    while (true) {
        kvmsg_t *kvmsg = kvmsg_recv (snapshot);
        if (!kvmsg)
            break;          //  Interrupted
        if (streq (kvmsg_key (kvmsg), "KTHXBAI")) {
            sequence = kvmsg_sequence (kvmsg);
            printf ("Received snapshot=%d\n", (int) sequence);
            kvmsg_destroy (&kvmsg);
            break;          //  Done
        }
        kvmsg_store (&kvmsg, kvmap);
    }
    //  Now apply pending updates, discard out-of-sequence messages
    while (!zctx_interrupted) {
        kvmsg_t *kvmsg = kvmsg_recv (subscriber);
        if (!kvmsg)
            break;          //  Interrupted
        if (kvmsg_sequence (kvmsg) > sequence) {
            sequence = kvmsg_sequence (kvmsg);
            kvmsg_store (&kvmsg, kvmap);
        }
        else
            kvmsg_destroy (&kvmsg);
    }
    zhash_destroy (&kvmap);
    zctx_destroy (&ctx);
    return 0;
}
```
> [옮긴이] clonecli2에서 "ICANHAZ?" 메시지를 clonesrv2로 보내면 서버는 해시 테이블에 저장한 변경정보들을 `s_send_single()`통하여 모두 전송하고 "KTHXBAI" 메시지를 전송합니다.
clonecli2에서 "KTHXBAI"을 받으면 해당 sequence를 기준으로 clonesrv2에서 발행된 변경정보의 sequence와 비교하여 이후의 것들만 받아 해시 테이블에 보관합니다.(이전 정보는 폐기)

> [옮긴이] "ICANHAZ?"는 "I Can has?"(가져도 될까요?)이며 "KTHXBAI"는 "Ok, Thank you, goodbye"(예, 고마워요, 잘 있어요)를 의미합니다.

> [옮긴이] 빌드 및 테스트

~~~{.bash}
PS D:\git_store\zguide-kr\examples\C> cl -EHsc clonesrv2.c libzmq.lib czmq.lib
PS D:\git_store\zguide-kr\examples\C> cl -EHsc clonecli2.c libzmq.lib czmq.lib

PS D:\git_store\zguide-kr\examples\C> ./clonesrv2
Sending state shapshot=1401876

PS D:\git_store\zguide-kr\examples\C> ./clonecli2
Received snapshot=1401876
~~~

;Here are some things to note about these two programs:

2개의 프로그램에서 몇 가지 주목할 사항은 다음과 같습니다.

;* The server uses two tasks. One thread produces the updates (randomly) and sends these to the main PUB socket, while the other thread handles state requests on the ROUTER socket. The two communicate across PAIR sockets over an inproc:// connection.
;* The client is really simple. In C, it consists of about fifty lines of code. A lot of the heavy lifting is done in the kvmsg class. Even so, the basic Clone pattern is easier to implement than it seemed at first.
;* We don't use anything fancy for serializing the state. The hash table holds a set of kvmsg objects, and the server sends these, as a batch of messages, to the client requesting state. If multiple clients request state at once, each will get a different snapshot.
;* We assume that the client has exactly one server to talk to. The server must be running; we do not try to solve the question of what happens if the server crashes.

* 서버는 2가지 작업을 수행합니다. 하나의 스레드(clonesrv2)는 변경정보(무작위로)를 생성하여 메인 PUB 소켓으로 보내고, 다른 스레드(state_manager)는 파이프(PAIR)에 변경정보를 받아 해시 테이블에 보관하고 ROUTER 소켓에서 클라이언트의 상태 요청들을 처리합니다. 메인 스레드(clonesrv2)와 자식 스레드(state_manager)는 `inproc://` 연결을 통해 PAIR 소켓을 통해 통신합니다.
* 클라이언트는 정말 간단합니다. C 언어에서는 약 50 줄의 코드로 구성됩니다. 많은 무거운 작업이 kvmsg(kvsimple) 클래스에서 수행됩니다. 그럼에도 불구하고 기본 복제 패턴은 처음에 보였던 것보다 구현하기가 쉽습니다.
* 우리는 상태를 직렬화하기 위해 화려한 것을 사용하지 않습니다. 해시 테이블은 일련의 kvmsg 객체를 보유하고 서버는 이러한 객체를 일련의 메시지로 클라이언트 요청(ICANHAZ) 시에 보냅니다. 여러 클라이언트들이 한 번에 상태를 요청하면, 각각 다른 스냅샷을 얻습니다.
* 우리는 클라이언트에 정확히 하나의 서버가 있다고 가정하며 서버가 실행 중이어야 합니다. 서버에 장애가 발생하면 어떻게 되는지는 여기서 다루지 않습니다.

;Right now, these two programs don't do anything real, but they correctly synchronize state. It's a neat example of how to mix different patterns: PAIR-PAIR, PUB-SUB, and ROUTER-DEALER.

현재, 2개 프로그램은 실제 작업을 수행하지 않지만 상태를 올바르게 동기화합니다. 깔끔한 예제로 PAIR-PAIR, PUB-SUB 및 ROUTER-DEALER와 같은 다양한 패턴을 혼합하는 방법을 다루었습니다.

;### Republishing Updates from Clients

### 클라이언트들로부터 변경정보 재발행

;In our second model, changes to the key-value store came from the server itself. This is a centralized model that is useful, for example if we have a central configuration file we want to distribute, with local caching on each node. A more interesting model takes updates from clients, not the server. The server thus becomes a stateless broker. This gives us some benefits:

두 번째 모델에서 키-값 저장소에 대한 변경 사항은 서버 자체적으로 나왔습니다. 중앙집중식 모델에서는 유용하며, 예를 들어 서버에 중앙 구성 파일을 두고 각 노드들의 로컬 저장소에 사용하기 위해 배포합니다. 더 흥미로운 모델은 서버가 아닌 클라이언트로부터 변경정보들을 받는 것이며, 이럴 경우 서버는 상태 비저장(stateless) 브로커가 되며, 몇 가지 이점을 제공합니다.

;* We're less worried about the reliability of the server. If it crashes, we can start a new instance and feed it new values.
;* We can use the key-value store to share knowledge between active peers.

* 우리는 서버의 안정성에 대해 덜 걱정합니다. 서버에 충돌이 발생하면 재시작하여 클라이언트에 신규 값을 제공할 수 있습니다.
* 키-값 저장소를 사용하여 활성 동료 간에 지식(저장소)을 공유할 수 있습니다.

;To send updates from clients back to the server, we could use a variety of socket patterns. The simplest plausible solution is a PUSH-PULL combination.

클라이언트에서 다시 서버로 변경정보를 보내기 위해, 다양한 소켓 패턴을 사용할 수 있지만 가장 간단한 솔루션은 PUSH-PULL 조합입니다.

;Why don't we allow clients to publish updates directly to each other? While this would reduce latency, it would remove the guarantee of consistency. You can't get consistent shared state if you allow the order of updates to change depending on who receives them. Say we have two clients, changing different keys. This will work fine. But if the two clients try to change the same key at roughly the same time, they'll end up with different notions of its value.

클라이언트들이 서로에게 직접 변경정보를 발행하지 않는 이유는 지연 시간(latency)이 줄어들지만 일관성(consistency)을 보장할 수 없기 때문입니다. 
변경정보를 받는 사람에 따라 변경정보 순서를 변경한다면 일관된 공유 상태를 얻을 수 없습니다. 
서로 다른 키들을 변경하는 두 개의 클라이언트들이 있다고 가정하면 잘 작동하지만 두 개의 클라이언트들이 거의 동시에 동일한 키를 변경하려고 하면 동일한 키에 다른 값으로 변경되어 일관성이 없어집니다.

;There are a few strategies for obtaining consistency when changes happen in multiple places at once. We'll use the approach of centralizing all change. No matter the precise timing of the changes that clients make, they are all pushed through the server, which enforces a single sequence according to the order in which it gets updates.

여러 위치에서 동시 변경이 발생할 때 일관성을 유지하기 위한 몇 가지 전략들이 있습니다. 
우리는 모든 변경 사항을 중앙 집중화하는 접근 방식을 사용할 것입니다. 클라이언트가 변경하는 정확한 시간에 관계없이, 모든 변경 사항은 서버를 통해 전달되므로 변경정보를 받는 순서에 따라 단일 순서가 적용됩니다.

그림 60 - 변경정보 재발행

![Republishing Updates](images/fig60.svg)

;By mediating all changes, the server can also add a unique sequence number to all updates. With unique sequencing, clients can detect the nastier failures, including network congestion and queue overflow. If a client discovers that its incoming message stream has a hole, it can take action. It seems sensible that the client contact the server and ask for the missing messages, but in practice that isn't useful. If there are holes, they're caused by network stress, and adding more stress to the network will make things worse. All the client can do is warn its users that it is "unable to continue", stop, and not restart until someone has manually checked the cause of the problem.

모든 변경 사항을 조정하기 위해, 서버는 모든 변경정보에 고유한 순서 번호를 추가할 수 있습니다. 고유한 순서 번호를 통해 클라이언트는 네트워크 정체(cogestion) 및 대기열 오버플로우(overflow)를 포함하여 심각한 장애를 감지할 수 있습니다. 클라이언트는 수신 메시지 스트림에 구멍(순서 번호가 연결되지 않는 부분)이 있음을 발견하면 조치를 취할 수 있습니다. 클라이언트가 서버에 접속하여 누락된 메시지를 요청하는 것이 합리적으로 보이지만 실제로는 유용하지 않습니다. 만약 구멍 네트워크 처리 부하에 따른 스트레스로 인한 것이라면, 네트워크에 더 많은 스트레스를 가하면 상황이 악화됩니다. 클라이언트가 할 수 있는 일은 사용자에게 "계속할 수 없음"이라고 경고하고 중지하고, 누군가가 문제의 원인을 수동으로 확인할 때까지 재시작하지 않는 것입니다.

;We'll now generate state updates in the client. Here's the server:

클라이언트에서 상태 변경들을 생성하겠습니다. 
다음은 서버의 코드입니다.

clonesrv3.c : Clone server, Model Three in C
```cpp
//  Clone server - Model Three

//  Lets us build this source without creating a library
#include "kvsimple.c"

//  Routing information for a key-value snapshot
typedef struct {
    void *socket;           //  ROUTER socket to send to
    zframe_t *identity;     //  Identity of peer who requested state
} kvroute_t;

//  Send one state snapshot key-value pair to a socket
//  Hash item data is our kvmsg object, ready to send
static int
s_send_single (const char *key, void *data, void *args)
{
    kvroute_t *kvroute = (kvroute_t *) args;
    //  Send identity of recipient first
    zframe_send (&kvroute->identity,
        kvroute->socket, ZFRAME_MORE + ZFRAME_REUSE);
    kvmsg_t *kvmsg = (kvmsg_t *) data;
    kvmsg_send (kvmsg, kvroute->socket);
    return 0;
}

int main (void)
{
    //  Prepare our context and sockets
    zctx_t *ctx = zctx_new ();
    void *snapshot = zsocket_new (ctx, ZMQ_ROUTER);
    zsocket_bind (snapshot, "tcp://*:5556");
    void *publisher = zsocket_new (ctx, ZMQ_PUB);
    zsocket_bind (publisher, "tcp://*:5557");
    void *collector = zsocket_new (ctx, ZMQ_PULL);
    zsocket_bind (collector, "tcp://*:5558");

    //  .split body of main task
    //  The body of the main task collects updates from clients and
    //  publishes them back out to clients:

    int64_t sequence = 0;
    zhash_t *kvmap = zhash_new ();

    zmq_pollitem_t items [] = {
        { collector, 0, ZMQ_POLLIN, 0 },
        { snapshot, 0, ZMQ_POLLIN, 0 }
    };
    while (!zctx_interrupted) {
        int rc = zmq_poll (items, 2, 1000 * ZMQ_POLL_MSEC);

        //  Apply state update sent from client
        if (items [0].revents & ZMQ_POLLIN) {
            kvmsg_t *kvmsg = kvmsg_recv (collector);
            if (!kvmsg)
                break;          //  Interrupted
            kvmsg_set_sequence (kvmsg, ++sequence);
            kvmsg_send (kvmsg, publisher);
            kvmsg_store (&kvmsg, kvmap);
            printf ("I: publishing update %5d\n", (int) sequence);
        }
        //  Execute state snapshot request
        if (items [1].revents & ZMQ_POLLIN) {
            zframe_t *identity = zframe_recv (snapshot);
            if (!identity)
                break;          //  Interrupted

            //  Request is in second frame of message
            char *request = zstr_recv (snapshot);
            if (streq (request, "ICANHAZ?"))
                free (request);
            else {
                printf ("E: bad request, aborting\n");
                break;
            }
            //  Send state snapshot to client
            kvroute_t routing = { snapshot, identity };

            //  For each entry in kvmap, send kvmsg to client
            zhash_foreach (kvmap, s_send_single, &routing);

            //  Now send END message with sequence number
            printf ("I: sending shapshot=%d\n", (int) sequence);
            zframe_send (&identity, snapshot, ZFRAME_MORE);
            kvmsg_t *kvmsg = kvmsg_new (sequence);
            kvmsg_set_key  (kvmsg, "KTHXBAI");
            kvmsg_set_body (kvmsg, (byte *) "", 0);
            kvmsg_send     (kvmsg, snapshot);
            kvmsg_destroy (&kvmsg);
        }
    }
    printf (" Interrupted\n%d messages handled\n", (int) sequence);
    zhash_destroy (&kvmap);
    zctx_destroy (&ctx);

    return 0;
}
```

;And here is the client:

다음은 클라이언트의 코드입니다.

clonecli3: Clone client, Model Three in C
```cpp
//  Clone client - Model Three

//  Lets us build this source without creating a library
#include "kvsimple.c"

int main (void)
{
    //  Prepare our context and subscriber
    zctx_t *ctx = zctx_new ();
    void *snapshot = zsocket_new (ctx, ZMQ_DEALER);
    zsocket_connect (snapshot, "tcp://localhost:5556");
    void *subscriber = zsocket_new (ctx, ZMQ_SUB);
    zsocket_set_subscribe (subscriber, "");
    zsocket_connect (subscriber, "tcp://localhost:5557");
    void *publisher = zsocket_new (ctx, ZMQ_PUSH);
    zsocket_connect (publisher, "tcp://localhost:5558");

    zhash_t *kvmap = zhash_new ();
    srandom ((unsigned) time (NULL));

    //  .split getting a state snapshot
    //  We first request a state snapshot:
    int64_t sequence = 0;
    zstr_send (snapshot, "ICANHAZ?");
    while (true) {
        kvmsg_t *kvmsg = kvmsg_recv (snapshot);
        if (!kvmsg)
            break;          //  Interrupted
        if (streq (kvmsg_key (kvmsg), "KTHXBAI")) {
            sequence = kvmsg_sequence (kvmsg);
            printf ("I: received snapshot=%d\n", (int) sequence);
            kvmsg_destroy (&kvmsg);
            break;          //  Done
        }
        kvmsg_store (&kvmsg, kvmap);
    }
    //  .split processing state updates
    //  Now we wait for updates from the server and every so often, we
    //  send a random key-value update to the server:
    
    int64_t alarm = zclock_time () + 1000;
    while (!zctx_interrupted) {
        zmq_pollitem_t items [] = { { subscriber, 0, ZMQ_POLLIN, 0 } };
        int tickless = (int) ((alarm - zclock_time ()));
        if (tickless < 0)
            tickless = 0;
        int rc = zmq_poll (items, 1, tickless * ZMQ_POLL_MSEC);
        if (rc == -1)
            break;              //  Context has been shut down

        if (items [0].revents & ZMQ_POLLIN) {
            kvmsg_t *kvmsg = kvmsg_recv (subscriber);
            if (!kvmsg)
                break;          //  Interrupted

            //  Discard out-of-sequence kvmsgs, incl. heartbeats
            if (kvmsg_sequence (kvmsg) > sequence) {
                sequence = kvmsg_sequence (kvmsg);
                kvmsg_store (&kvmsg, kvmap);
                printf ("I: received update=%d\n", (int) sequence);
            }
            else
                kvmsg_destroy (&kvmsg);
        }
        //  If we timed out, generate a random kvmsg
        if (zclock_time () >= alarm) {
            kvmsg_t *kvmsg = kvmsg_new (0);
            kvmsg_fmt_key  (kvmsg, "%d", randof (10000));
            kvmsg_fmt_body (kvmsg, "%d", randof (1000000));
            kvmsg_send     (kvmsg, publisher);
            kvmsg_destroy (&kvmsg);
            alarm = zclock_time () + 1000;
        }
    }
    printf (" Interrupted\n%d messages in\n", (int) sequence);
    zhash_destroy (&kvmap);
    zctx_destroy (&ctx);
    return 0;
}
```
> [옮긴이] clonecli3에서 1초마다 보내는 변경정보를 clonesrv3은 클라이언트들에 발행하며 clonecli3 중지하면 clonesrv3도 더 이상 변경정보를 발행하지 않습니다.

> [옮긴이] 빌드 및 테스트

~~~{.bash}
PS D:\git_store\zguide-kr\examples\C> cl -EHsc clonesrv3.c libzmq.lib czmq.lib
PS D:\git_store\zguide-kr\examples\C> cl -EHsc clonecli3.c libzmq.lib czmq.lib

PS D:\git_store\zguide-kr\examples\C> ./clonesrv3
I: sending shapshot=0
I: publishing update     1
I: publishing update     2
I: publishing update     3
I: sending snapshot=3
I: publishing update     4
I: publishing update     5
...

PS D:\git_store\zguide-kr\examples\C> ./clonecli3
I: received snapshot=0
I: received update=1
I: received update=2
I: received update=3
I: received update=4
I: received update=5
...

PS D:\git_store\zguide-kr\examples\C> ./clonecli3
I: received snapshot=3
I: received update=4
I: received update=5
...
~~~
;Here are some things to note about this third design:

세 번째 설계에서 몇 가지 주목할 점은 다음과 같습니다.

;* The server has collapsed to a single task. It manages a PULL socket for incoming updates, a ROUTER socket for state requests, and a PUB socket for outgoing updates.
;* The client uses a simple tickless timer to send a random update to the server once a second. In a real implementation, we would drive updates from application code.

* 서버가 단일 작업(clonesrv3)으로 축소되었으며 클라이언트로부터 수신되는 변경정보를 위한 PULL 소켓, 상태 요청 및 스냅샷 전달을 위한 위한 ROUTER 소켓, 클라이언트에게 변경정보 송신을 위한 PUB 소켓을 관리합니다.
* 클라이언트는 간단한 무지연(tickless) 타이머를 사용하여 1초에 한 번씩 서버에 무작위 업데이트를 보냅니다. 실제 구현에서는 응용프로그램 코드에서 업데이트를 유도합니다.

;### Working with Subtrees

### 하위트리와 작업

;As we grow the number of clients, the size of our shared store will also grow. It stops being reasonable to send everything to every client. This is the classic story with pub-sub: when you have a very small number of clients, you can send every message to all clients. As you grow the architecture, this becomes inefficient. Clients specialize in different areas.

클라이언트들의 수가 증가하면 공유 스토어의 규모도 커질 것입니다. 모든 클라이언트에게 모든 스냅샷을 보내는 것은 합리적이지 않습니다. 이것은 발행-구독의 고전적인 이야기입니다 : 클라이언트들의 수가 매우 적으면 모든 메시지들을 모든 클라이언트들에게 보낼 수 있습니다. 아키텍처를 커지면 비효율적이 되며 클라이언트는 다양한 영역에 전문화되어 있습니다.

;So even when working with a shared store, some clients will want to work only with a part of that store, which we call a subtree. The client has to request the subtree when it makes a state request, and it must specify the same subtree when it subscribes to updates.

따라서 서버의 공유 저장소로 작업할 때에도 일부 클라이언트들은 해당 저장소의 일부에서만 작업하기를 원하여, 일부 저장소를 하위트리(subtree)라고 합니다. 클라이언트는 상태 요청 시 하위 트리를 요청해야 하며 변경정보들을 구독할 때 동일한 하위트리를 지정해야 합니다.

;There are a couple of common syntaxes for trees. One is the path hierarchy, and another is the topic tree. These look like this:

트리에 대한 몇 가지 공통적인 문법들이 있으며, 하나는 경로 계층(path hierarchy)이고 다른 하나는 토픽 트리(topic tree)입니다. 다음과 같이 보입니다.

;* Path hierarchy: /some/list/of/paths
;* Topic tree: some.list.of.topics

* 경로 계층 : /some/list/of/paths
* 토픽 트리 : some.list.of.topics

;We'll use the path hierarchy, and extend our client and server so that a client can work with a single subtree. Once you see how to work with a single subtree you'll be able to extend this yourself to handle multiple subtrees, if your use case demands it.

예제에서는 경로 계층을 사용하고, 클라이언트와 서버를 확장하여 클라이언트가 단일 하위트리로 작업하게 합니다. 단일 하위트리로 작업하는 방법을 알게 되면 용도에 따라 다중 하위트리들을 처리하도록 확장할 수 있습니다.

;Here's the server implementing subtrees, a small variation on Model Three:

다음은 모델 3의 작은 변형으로 하위트리를 구현하는 서버의 코드입니다.

clonesrv4.c : Clone server, Model Four in C
```cpp
//  Clone server - Model Four

//  Lets us build this source without creating a library
#include "kvsimple.c"

//  Routing information for a key-value snapshot
typedef struct {
    void *socket;           //  ROUTER socket to send to
    zframe_t *identity;     //  Identity of peer who requested state
    char *subtree;          //  Client subtree specification
} kvroute_t;

//  Send one state snapshot key-value pair to a socket
//  Hash item data is our kvmsg object, ready to send
static int
s_send_single (const char *key, void *data, void *args)
{
    kvroute_t *kvroute = (kvroute_t *) args;
    kvmsg_t *kvmsg = (kvmsg_t *) data;
    if (strlen (kvroute->subtree) <= strlen (kvmsg_key (kvmsg))
    &&  memcmp (kvroute->subtree,
                kvmsg_key (kvmsg), strlen (kvroute->subtree)) == 0) {
        //  Send identity of recipient first
        zframe_send (&kvroute->identity,
            kvroute->socket, ZFRAME_MORE + ZFRAME_REUSE);
        kvmsg_send (kvmsg, kvroute->socket);
    }
    return 0;
}

//  The main task is identical to clonesrv3 except for where it
//  handles subtrees.
//  .skip

int main (void)
{
    //  Prepare our context and sockets
    zctx_t *ctx = zctx_new ();
    void *snapshot = zsocket_new (ctx, ZMQ_ROUTER);
    zsocket_bind (snapshot, "tcp://*:5556");
    void *publisher = zsocket_new (ctx, ZMQ_PUB);
    zsocket_bind (publisher, "tcp://*:5557");
    void *collector = zsocket_new (ctx, ZMQ_PULL);
    zsocket_bind (collector, "tcp://*:5558");

    int64_t sequence = 0;
    zhash_t *kvmap = zhash_new ();

    zmq_pollitem_t items [] = {
        { collector, 0, ZMQ_POLLIN, 0 },
        { snapshot, 0, ZMQ_POLLIN, 0 }
    };
    while (!zctx_interrupted) {
        int rc = zmq_poll (items, 2, 1000 * ZMQ_POLL_MSEC);

        //  Apply state update sent from client
        if (items [0].revents & ZMQ_POLLIN) {
            kvmsg_t *kvmsg = kvmsg_recv (collector);
            if (!kvmsg)
                break;          //  Interrupted
            kvmsg_set_sequence (kvmsg, ++sequence);
            kvmsg_send (kvmsg, publisher);
            kvmsg_dump(kvmsg);
            kvmsg_store (&kvmsg, kvmap);
            printf ("I: publishing update %5d\n", (int) sequence);
       }
        //  Execute state snapshot request
        if (items [1].revents & ZMQ_POLLIN) {
            zframe_t *identity = zframe_recv (snapshot);
            if (!identity)
                break;          //  Interrupted
            //  .until
            //  Request is in second frame of message
            char *request = zstr_recv (snapshot);
            char *subtree = NULL;
            if (streq (request, "ICANHAZ?")) {
                free (request);
                subtree = zstr_recv (snapshot);
            }
            //  .skip
            else {
                printf ("E: bad request, aborting\n");
                break;
            }
            //  .until
            //  Send state snapshot to client
            kvroute_t routing = { snapshot, identity, subtree };
            //  .skip

            //  For each entry in kvmap, send kvmsg to client
            zhash_foreach (kvmap, s_send_single, &routing);

            //  .until
            //  Now send END message with sequence number
            printf ("I: sending shapshot=%d\n", (int) sequence);
            zframe_send (&identity, snapshot, ZFRAME_MORE);
            kvmsg_t *kvmsg = kvmsg_new (sequence);
            kvmsg_set_key  (kvmsg, "KTHXBAI");
            kvmsg_set_body (kvmsg, (byte *) subtree, 0);
            kvmsg_send     (kvmsg, snapshot);
            kvmsg_destroy (&kvmsg);
            free (subtree);
        }
    }
    //  .skip
    printf (" Interrupted\n%d messages handled\n", (int) sequence);
    zhash_destroy (&kvmap);
    zctx_destroy (&ctx);

    return 0;
}
```

;And here is the corresponding client:

하위트리의 저장소의 내용을 구독하는 클라이언트의 코드입니다.

clonecli4.c : Clone client, Model Four in C
```cpp
//  Clone client - Model Four

//  Lets us build this source without creating a library
#include "kvsimple.c"

//  This client is identical to clonecli3 except for where we
//  handles subtrees.
#define SUBTREE "/client/"
//  .skip

int main (void)
{
    //  Prepare our context and subscriber
    zctx_t *ctx = zctx_new ();
    void *snapshot = zsocket_new (ctx, ZMQ_DEALER);
    zsocket_connect (snapshot, "tcp://localhost:5556");
    void *subscriber = zsocket_new (ctx, ZMQ_SUB);
    zsocket_set_subscribe (subscriber, "");
    //  .until
    zsocket_connect (subscriber, "tcp://localhost:5557");
    zsocket_set_subscribe (subscriber, SUBTREE);
    //  .skip
    void *publisher = zsocket_new (ctx, ZMQ_PUSH);
    zsocket_connect (publisher, "tcp://localhost:5558");

    zhash_t *kvmap = zhash_new ();
    srandom ((unsigned) time (NULL));

    //  .until
    //  We first request a state snapshot:
    int64_t sequence = 0;
    zstr_sendm (snapshot, "ICANHAZ?");
    zstr_send  (snapshot, SUBTREE);
    //  .skip
    while (true) {
        kvmsg_t *kvmsg = kvmsg_recv (snapshot);
        if (!kvmsg)
            break;          //  Interrupted
        if (streq (kvmsg_key (kvmsg), "KTHXBAI")) {
            sequence = kvmsg_sequence (kvmsg);
            printf ("I: received snapshot=%d\n", (int) sequence);
            kvmsg_destroy (&kvmsg);
            break;          //  Done
        }
        kvmsg_store (&kvmsg, kvmap);
    }
    int64_t alarm = zclock_time () + 1000;
    while (!zctx_interrupted) {
        zmq_pollitem_t items [] = { { subscriber, 0, ZMQ_POLLIN, 0 } };
        int tickless = (int) ((alarm - zclock_time ()));
        if (tickless < 0)
            tickless = 0;
        int rc = zmq_poll (items, 1, tickless * ZMQ_POLL_MSEC);
        if (rc == -1)
            break;              //  Context has been shut down

        if (items [0].revents & ZMQ_POLLIN) {
            kvmsg_t *kvmsg = kvmsg_recv (subscriber);
            if (!kvmsg)
                break;          //  Interrupted

            //  Discard out-of-sequence kvmsgs, incl. heartbeats
            if (kvmsg_sequence (kvmsg) > sequence) {
                sequence = kvmsg_sequence (kvmsg);
                kvmsg_dump(kvmsg);
                kvmsg_store (&kvmsg, kvmap);
                printf ("I: received update=%d\n", (int) sequence);

            }
            else
                kvmsg_destroy (&kvmsg);
        }
        //  .until
        //  If we timed out, generate a random kvmsg
        if (zclock_time () >= alarm) {
            kvmsg_t *kvmsg = kvmsg_new (0);
            kvmsg_fmt_key  (kvmsg, "%s%d", SUBTREE, randof (10000));
            kvmsg_fmt_body (kvmsg, "%d", randof (1000000));
            kvmsg_send     (kvmsg, publisher);
            kvmsg_destroy (&kvmsg);
            alarm = zclock_time () + 1000;
        }
        //  .skip
    }
    printf (" Interrupted\n%d messages in\n", (int) sequence);
    zhash_destroy (&kvmap);
    zctx_destroy (&ctx);
    return 0;
}
```
> [옮긴이] 빌드 및 테스트
- clonecli4에서 필터링을 통해 "SUBTREE"가 포함되어 발행(publish)된 kvmsg 객체를 받아 해시 테이블에 저정하며, 1초 간격으로 상태 요청에 사용될 kvmsg의 key에 "SUBTREE"을 포함하여 보냅니다.

~~~{.bash}
PS D:\git_store\zguide-kr\examples\C> cl -EHsc clonesrv4.c libzmq.lib czmq.lib
PS D:\git_store\zguide-kr\examples\C> cl -EHsc clonecli4.c libzmq.lib czmq.lib

PS D:\git_store\zguide-kr\examples\C> ./clonesrv4
I: sending shapshot=0
I: publishing update     1
I: publishing update     2
I: publishing update     3
I: publishing update     4
...

PS D:\git_store\zguide-kr\examples\C> ./clonecli4
I: received snapshot=0
I: received update=1
I: received update=2
I: received update=3
I: received update=4
...
~~~
> [옮긴이] `kvm_dump()`을 통하여 메시지 내용을 확인하면 다음과 같습니다.

~~~{.bash}
PS D:\git_store\zguide-kr\examples\C> ./clonesrv4
I: sending shapshot=0
[seq:1][key:/client/5174][size:6] 393939323337
I: publishing update     1
[seq:2][key:/client/2364][size:6] 373230383235
I: publishing update     2
[seq:3][key:/client/4486][size:6] 393134363732
I: publishing update     3
[seq:4][key:/client/4894][size:6] 353832373333
I: publishing update     4
...

PS D:\git_store\zguide-kr\examples\C> ./clonecli4
I: received snapshot=0
[seq:1][key:/client/5174][size:6] 393939323337
I: received update=1
[seq:2][key:/client/2364][size:6] 373230383235
I: received update=2
[seq:3][key:/client/4486][size:6] 393134363732
I: received update=3
[seq:4][key:/client/4894][size:6] 353832373333
I: received update=4
~~~

;### Ephemeral Values

### 임시값들

;An ephemeral value is one that expires automatically unless regularly refreshed. If you think of Clone being used for a registration service, then ephemeral values would let you do dynamic values. A node joins the network, publishes its address, and refreshes this regularly. If the node dies, its address eventually gets removed.

임시값은 정기적으로 갱신(refresh)되지 없으면 자동으로 만료(expire)되는 값입니다. 등록 서비스에 복제가 사용된다고 생각하면, 임시값을 동적 값으로 사용할 수 있습니다. 노드는 네트워크에 가입하고 주소를 게시하고 이를 정기적으로 갱신합니다. 노드가 죽으면 그 주소는 결국 제거됩니다.

;The usual abstraction for ephemeral values is to attach them to a session, and delete them when the session ends. In Clone, sessions would be defined by clients, and would end if the client died. A simpler alternative is to attach a time to live (TTL) to ephemeral values, which the server uses to expire values that haven't been refreshed in time.

임시값들에 대한 일반적인 추상화는 이를 세션에 연결하고 세션이 종료될 때 삭제하는 것입니다. 
복제에서 세션은 클라이언트들에 의해 정의되며 클라이언트가 죽으면 종료됩니다. 
더 간단한 대안은 유효시간(TTL(Time to Live))을 임시값에 포함시켜 서버에서 TTL에 정해진 시간 동안 값이 갱신되지 않을 경우 만료하는 데 사용합니다.

;A good design principle that I use whenever possible is to not invent concepts that are not absolutely essential. If we have very large numbers of ephemeral values, sessions will offer better performance. If we use a handful of ephemeral values, it's fine to set a TTL on each one. If we use masses of ephemeral values, it's more efficient to attach them to sessions and expire them in bulk. This isn't a problem we face at this stage, and may never face, so sessions go out the window.

좋은 디자인 원칙은 가능한 본질이 아닌 개념을 만들지 않는 것입니다.
매우 많은 수의 임시값들이 있는 경우 세션은 더 나은 성능을 제공합니다. 소수의 임시값들을 사용하는 경우 각 값에 TTL을 설정하는 것이 좋습니다. 대량의 임시값들을 사용하는 경우 이를 세션에 포함하고 한꺼번에 만료하는 것이 더 효율적입니다. 이번 단계의 주제에서는 세션을 제외하겠습니다.

;Now we will implement ephemeral values. First, we need a way to encode the TTL in the key-value message. We could add a frame. The problem with using ØMQ frames for properties is that each time we want to add a new property, we have to change the message structure. It breaks compatibility. So let's add a properties frame to the message, and write the code to let us get and put property values.

이제 임시값을 구현합니다. 먼저 키-값 메시지에서 TTL을 인코딩하는 방법이 필요하며 프레임으로 추가할 수 있습니다.  속성으로 ØMQ 프레임을 사용할 때의 문제점은 매번 신규 속성을 추가할 때마다 메시지 구조를 변경할 경우 ØMQ 버전 간의 호환성을 깨뜨립니다. 따라서 메시지에 속성 프레임을 추가하고 속성값을 가져오고 입력할 수 있는 코드를 작성해 보겠습니다.

;Next, we need a way to say, "delete this value". Up until now, servers and clients have always blindly inserted or updated new values into their hash table. We'll say that if the value is empty, that means "delete this key".

다음으로, "이 값을 삭제하십시오"라고 말하는 방법이 필요합니다. 지금까지는 서버와 클라이언트는 항상 맹목적으로 해시 테이블에 신규 값을 넣거나 기존 값을 변경했습니다. 
앞으로는 해시 테이블에서 값이 비어 있으면 "키 삭제"를 의미하게 하겠습니다.

;Here's a more complete version of the kvmsg class, which implements the properties frame (and adds a UUID frame, which we'll need later on). It also handles empty values by deleting the key from the hash, if necessary:

다음은 기존 `kvsimple` 보다 좀 더 완벽한 버전의 `kvmsg` 클래스이며, 속성들 프레임(나중에 필요할 UUID 프레임을 추가)을 구현하였습니다. 또한 필요한 경우 해시 테이블에서 키(key)를 삭제함으로 빈 값(empty value)을 처리합니다.

kvmsg.c : 키-값 메시지 클래스
```cpp
//  kvmsg class - key-value message class for example applications

#include "kvmsg.h"
#ifdef _WIN32
#pragma comment(lib, "rpcrt4.lib")  // UuidCreate - Minimum supported OS Win 2000
#include <windows.h>
#else
#include <uuid/uuid.h>
#endif
#include "zlist.h"

//  Keys are short strings
#define KVMSG_KEY_MAX   255

//  Message is formatted on wire as 5 frames:
//  frame 0: key (ØMQ string)
//  frame 1: sequence (8 bytes, network order)
//  frame 2: uuid (blob, 16 bytes)
//  frame 3: properties (ØMQ string)
//  frame 4: body (blob)
#define FRAME_KEY       0
#define FRAME_SEQ       1
#define FRAME_UUID      2
#define FRAME_PROPS     3
#define FRAME_BODY      4
#define KVMSG_FRAMES    5

//  Structure of our class
struct _kvmsg {
    //  Presence indicators for each frame
    int present [KVMSG_FRAMES];
    //  Corresponding ØMQ message frames, if any
    zmq_msg_t frame [KVMSG_FRAMES];
    //  Key, copied into safe C string
    char key [KVMSG_KEY_MAX + 1];
    //  List of properties, as name=value strings
    zlist_t *props;
    size_t props_size;
};

//  .split property encoding
//  These two helpers serialize a list of properties to and from a
//  message frame:

static void
s_encode_props (kvmsg_t *self)
{
    zmq_msg_t *msg = &self->frame [FRAME_PROPS];
    if (self->present [FRAME_PROPS])
        zmq_msg_close (msg);

    zmq_msg_init_size (msg, self->props_size);
    char *prop = zlist_first (self->props);
    char *dest = (char *) zmq_msg_data (msg);
    while (prop) {
        strcpy (dest, prop);
        dest += strlen (prop);
        *dest++ = '\n';
        prop = zlist_next (self->props);
    }
    self->present [FRAME_PROPS] = 1;
}

static void
s_decode_props (kvmsg_t *self)
{
    zmq_msg_t *msg = &self->frame [FRAME_PROPS];
    self->props_size = 0;
    while (zlist_size (self->props))
        free (zlist_pop (self->props));

    size_t remainder = zmq_msg_size (msg);
    char *prop = (char *) zmq_msg_data (msg);
    char *eoln = memchr (prop, '\n', remainder);
    while (eoln) {
        *eoln = 0;
        zlist_append (self->props, strdup (prop));
        self->props_size += strlen (prop) + 1;
        remainder -= strlen (prop) + 1;
        prop = eoln + 1;
        eoln = memchr (prop, '\n', remainder);
    }
}

//  .split constructor and destructor
//  Here are the constructor and destructor for the class:

//  Constructor, takes a sequence number for the new kvmsg instance:
kvmsg_t *
kvmsg_new (int64_t sequence)
{
    kvmsg_t
        *self;

    self = (kvmsg_t *) zmalloc (sizeof (kvmsg_t));
    self->props = zlist_new ();
    kvmsg_set_sequence (self, sequence);
    return self;
}

//  zhash_free_fn callback helper that does the low level destruction:
void
kvmsg_free (void *ptr)
{
    if (ptr) {
        kvmsg_t *self = (kvmsg_t *) ptr;
        //  Destroy message frames if any
        int frame_nbr;
        for (frame_nbr = 0; frame_nbr < KVMSG_FRAMES; frame_nbr++)
            if (self->present [frame_nbr])
                zmq_msg_close (&self->frame [frame_nbr]);

        //  Destroy property list
        while (zlist_size (self->props))
            free (zlist_pop (self->props));
        zlist_destroy (&self->props);

        //  Free object itself
        free (self);
    }
}

//  Destructor
void
kvmsg_destroy (kvmsg_t **self_p)
{
    assert (self_p);
    if (*self_p) {
        kvmsg_free (*self_p);
        *self_p = NULL;
    }
}

//  .split recv method
//  This method reads a key-value message from the socket and returns a 
//  new {{kvmsg}} instance:

kvmsg_t *
kvmsg_recv (void *socket)
{
    //  This method is almost unchanged from kvsimple
    //  .skip
    assert (socket);
    kvmsg_t *self = kvmsg_new (0);

    //  Read all frames off the wire, reject if bogus
    int frame_nbr;
    for (frame_nbr = 0; frame_nbr < KVMSG_FRAMES; frame_nbr++) {
        if (self->present [frame_nbr])
            zmq_msg_close (&self->frame [frame_nbr]);
        zmq_msg_init (&self->frame [frame_nbr]);
        self->present [frame_nbr] = 1;
        if (zmq_msg_recv (&self->frame [frame_nbr], socket, 0) == -1) {
            kvmsg_destroy (&self);
            break;
        }
        //  Verify multipart framing
        int rcvmore = (frame_nbr < KVMSG_FRAMES - 1)? 1: 0;
        if (zsocket_rcvmore (socket) != rcvmore) {
            kvmsg_destroy (&self);
            break;
        }
    }
    //  .until
    if (self)
        s_decode_props (self);
    return self;
}

//  Send key-value message to socket; any empty frames are sent as such.
void
kvmsg_send (kvmsg_t *self, void *socket)
{
    assert (self);
    assert (socket);

    s_encode_props (self);
    //  The rest of the method is unchanged from kvsimple
    //  .skip
    int frame_nbr;
    for (frame_nbr = 0; frame_nbr < KVMSG_FRAMES; frame_nbr++) {
        zmq_msg_t copy;
        zmq_msg_init (&copy);
        if (self->present [frame_nbr])
            zmq_msg_copy (&copy, &self->frame [frame_nbr]);
        zmq_msg_send (&copy, socket,
            (frame_nbr < KVMSG_FRAMES - 1)? ZMQ_SNDMORE: 0);
        zmq_msg_close (&copy);
    }
}
//  .until

//  .split dup method
//  This method duplicates a {{kvmsg}} instance, returns the new instance:

kvmsg_t *
kvmsg_dup (kvmsg_t *self)
{
    kvmsg_t *kvmsg = kvmsg_new (0);
    int frame_nbr;
    for (frame_nbr = 0; frame_nbr < KVMSG_FRAMES; frame_nbr++) {
        if (self->present [frame_nbr]) {
            zmq_msg_t *src = &self->frame [frame_nbr];
            zmq_msg_t *dst = &kvmsg->frame [frame_nbr];
            zmq_msg_init_size (dst, zmq_msg_size (src));
            memcpy (zmq_msg_data (dst),
                    zmq_msg_data (src), zmq_msg_size (src));
            kvmsg->present [frame_nbr] = 1;
        }
    }
    kvmsg->props_size = zlist_size (self->props);
    char *prop = (char *) zlist_first (self->props);
    while (prop) {
        zlist_append (kvmsg->props, strdup (prop));
        prop = (char *) zlist_next (self->props);
    }
    return kvmsg;
}

//  The key, sequence, body, and size methods are the same as in kvsimple.
//  .skip

//  Return key from last read message, if any, else NULL
char *
kvmsg_key (kvmsg_t *self)
{
    assert (self);
    if (self->present [FRAME_KEY]) {
        if (!*self->key) {
            size_t size = zmq_msg_size (&self->frame [FRAME_KEY]);
            if (size > KVMSG_KEY_MAX)
                size = KVMSG_KEY_MAX;
            memcpy (self->key,
                zmq_msg_data (&self->frame [FRAME_KEY]), size);
            self->key [size] = 0;
        }
        return self->key;
    }
    else
        return NULL;
}

//  Set message key as provided
void
kvmsg_set_key (kvmsg_t *self, char *key)
{
    assert (self);
    zmq_msg_t *msg = &self->frame [FRAME_KEY];
    if (self->present [FRAME_KEY])
        zmq_msg_close (msg);
    zmq_msg_init_size (msg, strlen (key));
    memcpy (zmq_msg_data (msg), key, strlen (key));
    self->present [FRAME_KEY] = 1;
}

//  Set message key using printf format
void
kvmsg_fmt_key (kvmsg_t *self, char *format, ...)
{
    char value [KVMSG_KEY_MAX + 1];
    va_list args;

    assert (self);
    va_start (args, format);
    vsnprintf (value, KVMSG_KEY_MAX, format, args);
    va_end (args);
    kvmsg_set_key (self, value);
}

//  Return sequence nbr from last read message, if any
int64_t
kvmsg_sequence (kvmsg_t *self)
{
    assert (self);
    if (self->present [FRAME_SEQ]) {
        assert (zmq_msg_size (&self->frame [FRAME_SEQ]) == 8);
        byte *source = zmq_msg_data (&self->frame [FRAME_SEQ]);
        int64_t sequence = ((int64_t) (source [0]) << 56)
                         + ((int64_t) (source [1]) << 48)
                         + ((int64_t) (source [2]) << 40)
                         + ((int64_t) (source [3]) << 32)
                         + ((int64_t) (source [4]) << 24)
                         + ((int64_t) (source [5]) << 16)
                         + ((int64_t) (source [6]) << 8)
                         +  (int64_t) (source [7]);
        return sequence;
    }
    else
        return 0;
}

//  Set message sequence number
void
kvmsg_set_sequence (kvmsg_t *self, int64_t sequence)
{
    assert (self);
    zmq_msg_t *msg = &self->frame [FRAME_SEQ];
    if (self->present [FRAME_SEQ])
        zmq_msg_close (msg);
    zmq_msg_init_size (msg, 8);

    byte *source = zmq_msg_data (msg);
    source [0] = (byte) ((sequence >> 56) & 255);
    source [1] = (byte) ((sequence >> 48) & 255);
    source [2] = (byte) ((sequence >> 40) & 255);
    source [3] = (byte) ((sequence >> 32) & 255);
    source [4] = (byte) ((sequence >> 24) & 255);
    source [5] = (byte) ((sequence >> 16) & 255);
    source [6] = (byte) ((sequence >> 8)  & 255);
    source [7] = (byte) ((sequence)       & 255);

    self->present [FRAME_SEQ] = 1;
}

//  Return body from last read message, if any, else NULL
byte *
kvmsg_body (kvmsg_t *self)
{
    assert (self);
    if (self->present [FRAME_BODY])
        return (byte *) zmq_msg_data (&self->frame [FRAME_BODY]);
    else
        return NULL;
}

//  Set message body
void
kvmsg_set_body (kvmsg_t *self, byte *body, size_t size)
{
    assert (self);
    zmq_msg_t *msg = &self->frame [FRAME_BODY];
    if (self->present [FRAME_BODY])
        zmq_msg_close (msg);
    self->present [FRAME_BODY] = 1;
    zmq_msg_init_size (msg, size);
    memcpy (zmq_msg_data (msg), body, size);
}

//  Set message body using printf format
void
kvmsg_fmt_body (kvmsg_t *self, char *format, ...)
{
    char value [255 + 1];
    va_list args;

    assert (self);
    va_start (args, format);
    vsnprintf (value, 255, format, args);
    va_end (args);
    kvmsg_set_body (self, (byte *) value, strlen (value));
}

//  Return body size from last read message, if any, else zero
size_t
kvmsg_size (kvmsg_t *self)
{
    assert (self);
    if (self->present [FRAME_BODY])
        return zmq_msg_size (&self->frame [FRAME_BODY]);
    else
        return 0;
}
//  .until

//  .split UUID methods
//  These methods get and set the UUID for the key-value message:

byte *
kvmsg_uuid (kvmsg_t *self)
{
    assert (self);
    if (self->present [FRAME_UUID]
    &&  zmq_msg_size (&self->frame [FRAME_UUID]) == sizeof (uuid_t))
        return (byte *) zmq_msg_data (&self->frame [FRAME_UUID]);
    else
        return NULL;
}

#ifdef _WIN32
//  Sets the UUID to a randomly generated value
void
kvmsg_set_uuid (kvmsg_t *self)
{
    assert (self);
    zmq_msg_t *msg = &self->frame [FRAME_UUID];
    uuid_t uuid;
    UuidCreate(&uuid);
    if (self->present [FRAME_UUID])
        zmq_msg_close (msg);
    zmq_msg_init_size (msg, sizeof (uuid_t));
    memcpy (zmq_msg_data (msg), &uuid, sizeof (uuid_t));
    self->present [FRAME_UUID] = 1;
}
#else
void
kvmsg_set_uuid (kvmsg_t *self)
{
    assert (self);
    zmq_msg_t *msg = &self->frame [FRAME_UUID];
    uuid_t uuid;
    uuid_generate (uuid);
    if (self->present [FRAME_UUID])
        zmq_msg_close (msg);
    zmq_msg_init_size (msg, sizeof (uuid));
    memcpy (zmq_msg_data (msg), uuid, sizeof (uuid));
    self->present [FRAME_UUID] = 1;
}
#endif

//  .split property methods
//  These methods get and set a specified message property:

//  Get message property, return "" if no such property is defined.
char *
kvmsg_get_prop (kvmsg_t *self, char *name)
{
    assert (strchr (name, '=') == NULL);
    char *prop = zlist_first (self->props);
    size_t namelen = strlen (name);
    while (prop) {
        if (strlen (prop) > namelen
        &&  memcmp (prop, name, namelen) == 0
        &&  prop [namelen] == '=')
            return prop + namelen + 1;
        prop = zlist_next (self->props);
    }
    return "";
}

//  Set message property. Property name cannot contain '='. Max length of
//  value is 255 chars.
void
kvmsg_set_prop (kvmsg_t *self, char *name, char *format, ...)
{
    assert (strchr (name, '=') == NULL);

    char value [255 + 1];
    va_list args;
    assert (self);
    va_start (args, format);
    vsnprintf (value, 255, format, args);
    va_end (args);

    //  Allocate name=value string
    char *prop = malloc (strlen (name) + strlen (value) + 2);

    //  Remove existing property if any
    sprintf (prop, "%s=", name);
    char *existing = zlist_first (self->props);
    while (existing) {
        if (memcmp (prop, existing, strlen (prop)) == 0) {
            self->props_size -= strlen (existing) + 1;
            zlist_remove (self->props, existing);
            free (existing);
            break;
        }
        existing = zlist_next (self->props);
    }
    //  Add new name=value property string
    strcat (prop, value);
    zlist_append (self->props, prop);
    self->props_size += strlen (prop) + 1;
}

//  .split store method
//  This method stores the key-value message into a hash map, unless
//  the key and value are both null. It nullifies the {{kvmsg}} reference
//  so that the object is owned by the hash map, not the caller:

void
kvmsg_store (kvmsg_t **self_p, zhash_t *hash)
{
    assert (self_p);
    if (*self_p) {
        kvmsg_t *self = *self_p;
        assert (self);
        if (kvmsg_size (self)) {
            if (self->present [FRAME_KEY]
            &&  self->present [FRAME_BODY]) {
                zhash_update (hash, kvmsg_key (self), self);
                zhash_freefn (hash, kvmsg_key (self), kvmsg_free);
            }
        }
        else
            zhash_delete (hash, kvmsg_key (self));

        *self_p = NULL;
    }
}

//  .split dump method
//  This method extends the {{kvsimple}} implementation with support for
//  message properties:

void
kvmsg_dump (kvmsg_t *self)
{
    //  .skip
    if (self) {
        if (!self) {
            fprintf (stderr, "NULL");
            return;
        }
        size_t size = kvmsg_size (self);
        byte  *body = kvmsg_body (self);
        fprintf (stderr, "[seq:%" PRId64 "]", kvmsg_sequence (self));
        fprintf (stderr, "[key:%s]", kvmsg_key (self));
        //  .until
        fprintf (stderr, "[size:%zd] ", size);
        if (zlist_size (self->props)) {
            fprintf (stderr, "[");
            char *prop = zlist_first (self->props);
            while (prop) {
                fprintf (stderr, "%s;", prop);
                prop = zlist_next (self->props);
            }
            fprintf (stderr, "]");
        }
        //  .skip
        int char_nbr;
        for (char_nbr = 0; char_nbr < size; char_nbr++)
            fprintf (stderr, "%02X", body [char_nbr]);
        fprintf (stderr, "\n");
    }
    else
        fprintf (stderr, "NULL message\n");
}
//  .until

//  .split test method
//  This method is the same as in {{kvsimple}} with added support
//  for the uuid and property features of {{kvmsg}}:

int
kvmsg_test (int verbose)
{
    //  .skip
    kvmsg_t
        *kvmsg;

    printf (" * kvmsg: ");

    //  Prepare our context and sockets
    zctx_t *ctx = zctx_new ();
    void *output = zsocket_new (ctx, ZMQ_DEALER);
    int rc = zmq_bind (output, "inproc://kvmsg_selftest");
    assert (rc == 0);
    void *input = zsocket_new (ctx, ZMQ_DEALER);
    rc = zmq_connect (input, "inproc://kvmsg_selftest");
    assert (rc == 0);

    zhash_t *kvmap = zhash_new ();

    //  .until
    //  Test send and receive of simple message
    kvmsg = kvmsg_new (1);
    kvmsg_set_key  (kvmsg, "key");
    kvmsg_set_uuid (kvmsg);
    kvmsg_set_body (kvmsg, (byte *) "body", 4);
    if (verbose)
        kvmsg_dump (kvmsg);
    kvmsg_send (kvmsg, output);
    kvmsg_store (&kvmsg, kvmap);

    kvmsg = kvmsg_recv (input);
    if (verbose){
        kvmsg_dump (kvmsg);
        byte *uuid = kvmsg_uuid(kvmsg);
        int size = sizeof(uuid);
        fprintf (stderr, "UUID :");
        for (int char_nbr = 0; char_nbr < size; char_nbr++)
            fprintf (stderr, "%02X", uuid [char_nbr]);
        fprintf (stderr, "\n");
    }
    assert (streq (kvmsg_key (kvmsg), "key"));
    kvmsg_store (&kvmsg, kvmap);

    //  Test send and receive of message with properties
    kvmsg = kvmsg_new (2);
    kvmsg_set_prop (kvmsg, "prop1", "value1");
    kvmsg_set_prop (kvmsg, "prop2", "value1");
    kvmsg_set_prop (kvmsg, "prop2", "value2");
    kvmsg_set_key  (kvmsg, "key");
    kvmsg_set_uuid (kvmsg);
    kvmsg_set_body (kvmsg, (byte *) "body", 4);
    assert (streq (kvmsg_get_prop (kvmsg, "prop2"), "value2"));
    if (verbose)
        kvmsg_dump (kvmsg);
    kvmsg_send (kvmsg, output);
    kvmsg_destroy (&kvmsg);

    kvmsg = kvmsg_recv (input);
    if (verbose)
        kvmsg_dump (kvmsg);
    assert (streq (kvmsg_key (kvmsg), "key"));
    assert (streq (kvmsg_get_prop (kvmsg, "prop2"), "value2"));
    kvmsg_destroy (&kvmsg);
    //  .skip
    //  Shutdown and destroy all objects
    zhash_destroy (&kvmap);
    zctx_destroy (&ctx);

    printf ("OK\n");
    return 0;
}
//  .until
```
> [옮긴이] "kvmsg.c"는 원도우에서 실행되지 않아 수정이 필요합니다.
1. ipc 전송 방법을 inproc로 변경합니다.
  - 변경전 : "ipc://kvmsg_selftest.ipc"
  - 변경후 : "inproc://kvmsg_selftest"
2. uuid를 원도우 환경에서 사용할 수 있도록 4장 "titanic.c" 예제를 참조한다.

```cpp
#ifdef _WIN32
#pragma comment(lib, "rpcrt4.lib")  // UuidCreate - Minimum supported OS Win 2000
#include <windows.h>
#else
#include <uuid/uuid.h>
#endif
...
#ifdef _WIN32
//  Sets the UUID to a randomly generated value
void
kvmsg_set_uuid (kvmsg_t *self)
{
    assert (self);
    zmq_msg_t *msg = &self->frame [FRAME_UUID];
    uuid_t uuid;
    UuidCreate(&uuid);
    if (self->present [FRAME_UUID])
        zmq_msg_close (msg);
    zmq_msg_init_size (msg, sizeof (uuid_t));
    memcpy (zmq_msg_data (msg), &uuid, sizeof (uuid_t));
    self->present [FRAME_UUID] = 1;
}
#else
void
kvmsg_set_uuid (kvmsg_t *self)
{
    assert (self);
    zmq_msg_t *msg = &self->frame [FRAME_UUID];
    uuid_t uuid;
    uuid_generate (uuid);
    if (self->present [FRAME_UUID])
        zmq_msg_close (msg);
    zmq_msg_init_size (msg, sizeof (uuid));
    memcpy (zmq_msg_data (msg), uuid, sizeof (uuid));
    self->present [FRAME_UUID] = 1;
}
#endif
```

> [옮긴이] 테스트를 위하여 "kvmsgtest.c" 코드는 다음과 같습니다.

```cpp
#include "kvmsg.c"

void main()
{
    kvmsg_test(1);
}
```
> [옮긴이] 빌드 및 테스트 

~~~{.bash}
PS D:\git_store\zguide-kr\examples\C> cl -EHsc kvmsgtest.c libzmq.lib czmq.lib

PS D:\git_store\zguide-kr\examples\C> ./kvmsgtest
 * kvmsg: [seq:1][key:key][size:4] 626F6479
[seq:1][key:key][size:4] 626F6479
[seq:2][key:key][size:4] [prop1=value1;prop2=value2;]626F6479
[seq:2][key:key][size:4] [prop1=value1;prop2=value2;]626F6479
UUID :5301DB3A6170784D
OK
~~~

;The Model Five client is almost identical to Model Four. It uses the full kvmsg class now, and sets a randomized ttl property (measured in seconds) on each message:

모델 5 클라이언트는 모델 4와 거의 동일합니다. 이제 전체 kvmsg 클래스를 사용하고 각 메시지에 임의의 ttl 속성(초 단위로 측정)을 설정합니다.

```cpp
kvmsg_set_prop (kvmsg, "ttl", "%d", randof (30));
```

;### Using a Reactor

### 리엑터 사용

;Until now, we have used a poll loop in the server. In this next model of the server, we switch to using a reactor. In C, we use CZMQ's zloop class. Using a reactor makes the code more verbose, but easier to understand and build out because each piece of the server is handled by a separate reactor handler.

지금까지 우리는 서버에서 폴(poll) 루프를 사용했습니다. 다음 서버 모델에서는 리엑터로 전환하여 사용합니다. C 언어에서는 CZMQ의 zloop 클래스를 사용합니다. 리액터를 사용하면 코드가 더 장황해지지만 서버의 각 부분이 개별 리엑터 핸들러에 의해 처리되기 때문에 쉽게 이해할 수 있습니다.

;We use a single thread and pass a server object around to the reactor handlers. We could have organized the server as multiple threads, each handling one socket or timer, but that works better when threads don't have to share data. In this case all work is centered around the server's hashmap, so one thread is simpler.

단일 스레드를 사용하고 리엑터 핸들러에 서버 객체를 전달합니다. 서버를 여러 개의 스레드들로 구성하여 각 스레드는 하나의 소켓 또는 타이머를 처리하지만 스레드가 데이터를 공유하지 않을 경우 더 잘 작동합니다. 이 경우 모든 작업은 서버의 해시 맵을 중심으로 이루어지며 하나의 스레드가 더 간단합니다.

;There are three reactor handlers:
3개의 리엑터 핸들러는 다음과 같습니다.

;* One to handle snapshot requests coming on the ROUTER socket;
;* One to handle incoming updates from clients, coming on the PULL socket;
;* One to expire ephemeral values that have passed their TTL.

* 하나는 ROUTER 소켓을 통해 오는 스냅샷 요청들을 처리합니다.
* 하나는 PULL 소켓을 통해 오는 클라이언트들의 변경정보를 처리합니다.
* 하나는 TTL이 경과된 임시값을 만료 처리(값을 공백으로 처리)합니다.

clonesrv5.c : Clone server, Model Five in C
```cpp
//  Clone server - Model Five

//  Lets us build this source without creating a library
#include "kvmsg.c"

//  zloop reactor handlers
static int s_snapshots (zloop_t *loop, zmq_pollitem_t *poller, void *args);
static int s_collector (zloop_t *loop, zmq_pollitem_t *poller, void *args);
static int s_flush_ttl (zloop_t *loop, int timer_id, void *args);

//  Our server is defined by these properties
typedef struct {
    zctx_t *ctx;                //  Context wrapper
    zhash_t *kvmap;             //  Key-value store
    zloop_t *loop;              //  zloop reactor
    int port;                   //  Main port we're working on
    int64_t sequence;           //  How many updates we're at
    void *snapshot;             //  Handle snapshot requests
    void *publisher;            //  Publish updates to clients
    void *collector;            //  Collect updates from clients
} clonesrv_t;

int main (void)
{
    clonesrv_t *self = (clonesrv_t *) zmalloc (sizeof (clonesrv_t));
    self->port = 5556;
    self->ctx = zctx_new ();
    self->kvmap = zhash_new ();
    self->loop = zloop_new ();
    zloop_set_verbose (self->loop, false);

    //  Set up our clone server sockets
    self->snapshot  = zsocket_new (self->ctx, ZMQ_ROUTER);
    zsocket_bind (self->snapshot,  "tcp://*:%d", self->port);
    self->publisher = zsocket_new (self->ctx, ZMQ_PUB);
    zsocket_bind (self->publisher, "tcp://*:%d", self->port + 1);
    self->collector = zsocket_new (self->ctx, ZMQ_PULL);
    zsocket_bind (self->collector, "tcp://*:%d", self->port + 2);

    //  Register our handlers with reactor
    zmq_pollitem_t poller = { 0, 0, ZMQ_POLLIN };
    poller.socket = self->snapshot;
    zloop_poller (self->loop, &poller, s_snapshots, self);
    poller.socket = self->collector;
    zloop_poller (self->loop, &poller, s_collector, self);
    zloop_timer (self->loop, 1000, 0, s_flush_ttl, self);

    //  Run reactor until process interrupted
    zloop_start (self->loop);

    zloop_destroy (&self->loop);
    zhash_destroy (&self->kvmap);
    zctx_destroy (&self->ctx);
    free (self);
    return 0;
}

//  .split send snapshots
//  We handle ICANHAZ? requests by sending snapshot data to the
//  client that requested it:

//  Routing information for a key-value snapshot
typedef struct {
    void *socket;           //  ROUTER socket to send to
    zframe_t *identity;     //  Identity of peer who requested state
    char *subtree;          //  Client subtree specification
} kvroute_t;

//  We call this function for each key-value pair in our hash table
static int
s_send_single (const char *key, void *data, void *args)
{
    kvroute_t *kvroute = (kvroute_t *) args;
    kvmsg_t *kvmsg = (kvmsg_t *) data;
    if (strlen (kvroute->subtree) <= strlen (kvmsg_key (kvmsg))
    &&  memcmp (kvroute->subtree,
                kvmsg_key (kvmsg), strlen (kvroute->subtree)) == 0) {
        zframe_send (&kvroute->identity,    //  Choose recipient
            kvroute->socket, ZFRAME_MORE + ZFRAME_REUSE);
        kvmsg_send (kvmsg, kvroute->socket);
    }
    return 0;
}

//  .split snapshot handler
//  This is the reactor handler for the snapshot socket; it accepts
//  just the ICANHAZ? request and replies with a state snapshot ending
//  with a KTHXBAI message:

static int
s_snapshots (zloop_t *loop, zmq_pollitem_t *poller, void *args)
{
    clonesrv_t *self = (clonesrv_t *) args;

    zframe_t *identity = zframe_recv (poller->socket);
    if (identity) {
        //  Request is in second frame of message
        char *request = zstr_recv (poller->socket);
        char *subtree = NULL;
        if (streq (request, "ICANHAZ?")) {
            free (request);
            subtree = zstr_recv (poller->socket);
        }
        else
            printf ("E: bad request, aborting\n");

        if (subtree) {
            //  Send state socket to client
            kvroute_t routing = { poller->socket, identity, subtree };
            zhash_foreach (self->kvmap, s_send_single, &routing);

            //  Now send END message with sequence number
            zclock_log ("I: sending shapshot=%d", (int) self->sequence);
            zframe_send (&identity, poller->socket, ZFRAME_MORE);
            kvmsg_t *kvmsg = kvmsg_new (self->sequence);
            kvmsg_set_key  (kvmsg, "KTHXBAI");
            kvmsg_set_body (kvmsg, (byte *) subtree, 0);
            kvmsg_send     (kvmsg, poller->socket);
            kvmsg_destroy (&kvmsg);
            free (subtree);
        }
        zframe_destroy(&identity);
    }
    return 0;
}

//  .split collect updates
//  We store each update with a new sequence number, and if necessary, a
//  time-to-live. We publish updates immediately on our publisher socket:

static int
s_collector (zloop_t *loop, zmq_pollitem_t *poller, void *args)
{
    clonesrv_t *self = (clonesrv_t *) args;

    kvmsg_t *kvmsg = kvmsg_recv (poller->socket);
    if (kvmsg) {
        kvmsg_set_sequence (kvmsg, ++self->sequence);
        kvmsg_send (kvmsg, self->publisher);
        int ttl = atoi (kvmsg_get_prop (kvmsg, "ttl"));
        if (ttl)
            kvmsg_set_prop (kvmsg, "ttl",
                "%" PRId64, zclock_time () + ttl * 1000);
        kvmsg_store (&kvmsg, self->kvmap);
        zclock_log ("I: publishing update=%d", (int) self->sequence);
    }
    return 0;
}

//  .split flush ephemeral values
//  At regular intervals, we flush ephemeral values that have expired. This
//  could be slow on very large data sets:

//  If key-value pair has expired, delete it and publish the
//  fact to listening clients.
static int
s_flush_single (const char *key, void *data, void *args)
{
    clonesrv_t *self = (clonesrv_t *) args;

    kvmsg_t *kvmsg = (kvmsg_t *) data;
    int64_t ttl;
    sscanf (kvmsg_get_prop (kvmsg, "ttl"), "%" PRId64, &ttl);
    if (ttl && zclock_time () >= ttl) {
        kvmsg_set_sequence (kvmsg, ++self->sequence);
        kvmsg_set_body (kvmsg, (byte *) "", 0);
        kvmsg_send (kvmsg, self->publisher);
        kvmsg_store (&kvmsg, self->kvmap);
        zclock_log ("I: publishing delete=%d", (int) self->sequence);
    }
    return 0;
}

static int
s_flush_ttl (zloop_t *loop, int timer_id, void *args)
{
    clonesrv_t *self = (clonesrv_t *) args;
    if (self->kvmap)
        zhash_foreach (self->kvmap, s_flush_single, args);
    return 0;
}
```

kvmsg클래스에 TTL 속성을 부가한 클라이언트 소스는 다음과 같습니다.

clonecli5.c : Clone Client, Model Five in C
```cpp
//  Clone client - Model Five

//  Lets us build this source without creating a library
#include "kvmsg.c"
#define SUBTREE "/client/"

int main (void)
{
    //  Prepare our context and subscriber
    zctx_t *ctx = zctx_new ();
    void *snapshot = zsocket_new (ctx, ZMQ_DEALER);
    zsocket_connect (snapshot, "tcp://localhost:5556");
    void *subscriber = zsocket_new (ctx, ZMQ_SUB);
    zsocket_set_subscribe (subscriber, "");
    zsocket_connect (subscriber, "tcp://localhost:5557");
    zsocket_set_subscribe (subscriber, SUBTREE);
    void *publisher = zsocket_new (ctx, ZMQ_PUSH);
    zsocket_connect (publisher, "tcp://localhost:5558");

    zhash_t *kvmap = zhash_new ();
    srandom ((unsigned) time (NULL));

    //  Get state snapshot
    int64_t sequence = 0;
    zstr_sendm (snapshot, "ICANHAZ?");
    zstr_send  (snapshot, SUBTREE);
    while (true) {
        kvmsg_t *kvmsg = kvmsg_recv (snapshot);
        if (!kvmsg)
            break;          //  Interrupted
        if (streq (kvmsg_key (kvmsg), "KTHXBAI")) {
            sequence = kvmsg_sequence (kvmsg);
            printf ("I: received snapshot=%d\n", (int) sequence);
            kvmsg_destroy (&kvmsg);
            break;          //  Done
        }
        kvmsg_store (&kvmsg, kvmap);
    }
    int64_t alarm = zclock_time () + 1000;
    while (!zctx_interrupted) {
        zmq_pollitem_t items [] = { { subscriber, 0, ZMQ_POLLIN, 0 } };
        int tickless = (int) ((alarm - zclock_time ()));
        if (tickless < 0)
            tickless = 0;
        int rc = zmq_poll (items, 1, tickless * ZMQ_POLL_MSEC);
        if (rc == -1)
            break;              //  Context has been shut down

        if (items [0].revents & ZMQ_POLLIN) {
            kvmsg_t *kvmsg = kvmsg_recv (subscriber);
            if (!kvmsg)
                break;          //  Interrupted

            //  Discard out-of-sequence kvmsgs, incl. heartbeats
            if (kvmsg_sequence (kvmsg) > sequence) {
                sequence = kvmsg_sequence (kvmsg);
                kvmsg_store (&kvmsg, kvmap);
                printf ("I: received update=%d\n", (int) sequence);
            }
            else
                kvmsg_destroy (&kvmsg);
        }
        //  If we timed out, generate a random kvmsg
        if (zclock_time () >= alarm) {
            kvmsg_t *kvmsg = kvmsg_new (0);
            kvmsg_fmt_key  (kvmsg, "%s%d", SUBTREE, randof (10000));
            kvmsg_fmt_body (kvmsg, "%d", randof (1000000));
            kvmsg_set_prop (kvmsg, "ttl", "%d", randof (30));
            kvmsg_send     (kvmsg, publisher);
            kvmsg_destroy (&kvmsg);
            alarm = zclock_time () + 1000;
        }
    }
    printf (" Interrupted\n%d messages in\n", (int) sequence);
    zhash_destroy (&kvmap);
    zctx_destroy (&ctx);
    return 0;
}
```
> [옮긴이] 빌드 및 테스트

~~~{.bash}
PS D:\git_store\zguide-kr\examples\C> cl -EHsc clonesrv5.c libzmq.lib czmq.lib
PS D:\git_store\zguide-kr\examples\C> cl -EHsc clonecli5.c libzmq.lib czmq.lib

PS D:\git_store\zguide-kr\examples\C> ./clonesrv5
20-08-29 07:27:56 I: sending shapshot=0
20-08-29 07:27:57 I: publishing update=1
20-08-29 07:27:58 I: publishing update=2
...
20-08-29 07:28:08 I: publishing update=12
20-08-29 07:28:08 I: publishing delete=13
20-08-29 07:28:09 I: publishing update=14
20-08-29 07:28:10 I: publishing update=15
20-08-29 07:28:11 I: publishing update=16
20-08-29 07:28:12 I: publishing update=17
20-08-29 07:28:12 I: publishing delete=18
...

PS D:\git_store\zguide-kr\examples\C> ./clonecli5
I: received snapshot=0
I: received update=1
I: received update=2
I: received update=3
I: received update=4
...
~~~

;### Adding the Binary Star Pattern for Reliability

### 신뢰성을 위한 바이너리 스타 패턴 추가

;The Clone models we've explored up to now have been relatively simple. Now we're going to get into unpleasantly complex territory, which has me getting up for another espresso. You should appreciate that making "reliable" messaging is complex enough that you always need to ask, "Do we actually need this?" before jumping into it. If you can get away with unreliable or with "good enough" reliability, you can make a huge win in terms of cost and complexity. Sure, you may lose some data now and then. It is often a good trade-off. Having said, that, and… sips… because the espresso is really good, let's jump in.

지금까지 살펴본 복제 모델은 비교적 간단했습니다. 이제 우리는 불쾌하고 복잡한 영역에 들어가서 다른 에스프레소를 마시게 될 것입니다. 
"신뢰성 있는" 메시징을 만드는 것이 복잡한 만큼 "실제로 이것이 필요한가?"라는 의문이 있어야 합니다. "신뢰성이 없는" 혹은 "충분히 좋은 신뢰성을 가진" 상황에서 벗어날 수 있다면 비용과 복잡성 측면에서 큰 승리를 한 것입니다. 물론 때때로 일부 데이터가 유실할 수 있지만 좋은 절충안입니다. 하지만 한 모금의 에스프레소는 정말 좋습니다. 복잡성의 세계로 뛰어들어 가겠습니다. 

;As you play with the last model, you'll stop and restart the server. It might look like it recovers, but of course it's applying updates to an empty state instead of the proper current state. Any new client joining the network will only get the latest updates instead of the full historical record.

이전 모델(리엑터와 TTL)로 작업할 때 서버를 중지하고 재시작하면 복구된 것처럼 보이지만 물론 적절한 현재 상태 대신 빈 상태에 변경정보들을 반영합니다. 네트워크에 가입하는 모든 신규 클라이언트는 전체 이력 데이터 대신 최신 변경정보만 받습니다.

;What we want is a way for the server to recover from being killed, or crashing. We also need to provide backup in case the server is out of commission for any length of time. When someone asks for "reliability", ask them to list the failures they want to handle. In our case, these are:

우리가 서버가 죽거나 충돌하면 복구하는 방법이 필요합니다. 또한 서버가 일정 시간 동안 작동하지 않는 경우 백업을 제공해야 합니다. 누군가 "신뢰성" 구현이 필요하면 우선 처리해야 하는 장애들에 대한 목록 요구해야 하며, 우리의 경우 다음과 같습니다.

;* The server process crashes and is automatically or manually restarted. The process loses its state and has to get it back from somewhere.
;* The server machine dies and is offline for a significant time. Clients have to switch to an alternate server somewhere.
;* The server process or machine gets disconnected from the network, e.g., a switch dies or a datacenter gets knocked out. It may come back at some point, but in the meantime clients need an alternate server.

* 서버 프로세스가 충돌하여 자동 또는 수동으로 재시작됩니다. 프로세스는 상태를 잃고 어딘가에서 다시 가져와야 합니다.
* 서버 머신(하드웨어)이 죽고 상당한 시간 동안 오프라인 상태입니다. 클라이언트는 어딘가의 대체 서버로 전환해야 합니다.
* 서버 프로세스 또는 머신이 네트워크에서 연결 해제됩니다(예 : 스위치가 죽거나 데이터센터가 무응답). 어제든지 정상화될 수 있지만 그동안 클라이언트들은 대체 서버가 필요합니다.

;Our first step is to add a second server. We can use the Binary Star pattern from Chapter 4 - Reliable Request-Reply Patterns to organize these into primary and backup. Binary Star is a reactor, so it's useful that we already refactored the last server model into a reactor style.

첫 번째 단계는 대체 서버를 추가하는 것입니다."4장 - 신뢰할 수 있는 요청-응답 패턴"의 바이너리 스타 패턴을 사용하여 기본 및 백업으로 구성할 수 있습니다. 바이너리 스타는 리엑터를 사용하므로 이전 서버 모델(리엑터+TTL)을 재구성하기에는 유용합니다.

;We need to ensure that updates are not lost if the primary server crashes. The simplest technique is to send them to both servers. The backup server can then act as a client, and keep its state synchronized by receiving updates as all clients do. It'll also get new updates from clients. It can't yet store these in its hash table, but it can hold onto them for a while.

기본 서버가 충돌하더라도 변경정보들이 손실되지 않도록 해야 합니다. 가장 간단한 방법은 클라이언트의 상태 변경정보를 2개 서버(기본-백업)에 모두 보내는 것입니다. 그런 다음 백업 서버는 클라이언트 역할을 수행하며, 모든 클라이언트가 수행하는 것처럼 서버로부터 변경정보들을 수신하여 상태를 동기화합니다. 또한 백업 서버는 클라이언트로부터 새로운 변경정보를 받지만 해시테 이블에 저장하지 않고 잠시 동안 가지고 있습니다.

;So, Model Six introduces the following changes over Model Five:

따라서 모델 6은 모델 5에 비해 다음과 같은 변경 사항이 적용되었습니다.

;* We use a pub-sub flow instead of a push-pull flow for client updates sent to the servers. This takes care of fanning out the updates to both servers. Otherwise we'd have to use two DEALER sockets.
;* We add heartbeats to server updates (to clients), so that a client can detect when the primary server has died. It can then switch over to the backup server.
;* We connect the two servers using the Binary Star bstar reactor class. Binary Star relies on the clients to vote by making an explicit request to the server they consider active. We'll use snapshot requests as the voting mechanism.
;* We make all update messages uniquely identifiable by adding a UUID field. The client generates this, and the server propagates it back on republished updates.
;* The passive server keeps a "pending list" of updates that it has received from clients, but not yet from the active server; or updates it's received from the active server, but not yet from the clients. The list is ordered from oldest to newest, so that it is easy to remove updates off the head.

* 클라이언트가 서버로 전송하는 상태 변경정보에 PUSH-PULL 패턴 대신 PUB-SUB 패턴을 사용합니다. 이렇게 하면 2대의 서버로 변경정보가 동일하게 펼쳐져 갈 수 있습니다. 그렇지 않으면 2개의 DEALER 소켓들을 사용해야 합니다.
* 서버 변경정보 메시지(클라이언트로 가는)에 심박을 추가하여, 클라이언트가 기본 서버의 죽음을 감지하게 하여, 클라이언트가 백업 서버로 전환하게 합니다.
* 2대의 서버를 바이너리 스타 bstar 리엑터 클래스를 사용하여 연결합니다. 바이너리 스타는 클라이언트들이 활성화 상태로 여기는 서버에 명시적인 요청하는 투표에 의존합니다. 투표 메커니즘으로 스냅샷 요청들을 사용할 것입니다.
* 모든 변경정보 메시지에 고유의 식별자로 UUID 필드를 추가하였습니다. 클라이언트는 상태 변경정보에서 생성하고, 서버는 다시 클라이언트에서 변경정보를 받아하여 전파합니다.
* 비활성 서버는 "보류 목록(Pending list)"에 
  - 클라이언트에서 수신되었지만 아직 활성 서버에서는 수신되지 않은 변경정보들을 저장하거나, 
  - 활성 서버에서는 수신되었지만 클라이언트에서는 아직 수신되지 않은 변경정보들을 저장합니다.
  - 보류 목록은 오래된 것부터 최신 순서로 정렬되어 있으므로 최신 것부터 변경정보들을 쉽게 제거할 수 있습니다.

그림 61 - 복제 클라이언트 유한 상태 머신

![Clone Client Finite State Machine](images/fig61.svg)

;It's useful to design the client logic as a finite state machine. The client cycles through three states:

클라이언트 로직을 유한 상태 머신으로 설계하는 것이 유용합니다. 클라이언트는 다음 3가지 상태(INITIAL, SYNCING, ACTIVE)를 순환합니다.
> [옮긴이] 기존 클라이언트 예제에서도 클라이언트가 시작되면 서버에 상태 요청(ICANHAZ)하여 snapshot 받아 오면, 클라이언트에 snapshot을 반영하고 서버에 완료(KTHXBAI)를 보낸 이후,  클라이언트에서 상태 변경정보를 보내면 서버가 각 클라이언트에서 변경정보를 전송하도록 하였습니다.

;* The client opens and connects its sockets, and then requests a snapshot from the first server. To avoid request storms, it will ask any given server only twice. One request might get lost, which would be bad luck. Two getting lost would be carelessness.
;* The client waits for a reply (snapshot data) from the current server, and if it gets it, it stores it. If there is no reply within some timeout, it fails over to the next server.
;* When the client has gotten its snapshot, it waits for and processes updates. Again, if it doesn't hear anything from the server within some timeout, it fails over to the next server.

* [INITIAL] 클라이언트는 소켓(SUB, DEALER, PUB)을 열고 연결하고, 첫 번째 서버에서 스냅샷을 요청합니다. 클라이언트들의 요청 폭주를 방지하기 위해 주어진 서버에 2번만 요청합니다. 하나의 요청이 유실되는 것은 불행이지만, 두 번째까지 유실되는 것은 부주의입니다.
* [SYNCING] 클라이언트는 서버로부터 응답(스냅샷 데이터)을 기다렸다가 수신하여 저장합니다. 정해진 제한 시간 내에 서버로부터 응답이 없으면 다음 서버로 장애조치합니다.
* [ACTIVE] 클라이언트가 스냅샷을 받고, 서버의 변경정보들을 기다렸다가 처리합니다. 
다시 정해진 제한 시간(timeout)에 서버로부터 응답이 없으면 다음 서버로 장애조치합니다.

;The client loops forever. It's quite likely during startup or failover that some clients may be trying to talk to the primary server while others are trying to talk to the backup server. The Binary Star state machine handles this, hopefully accurately. It's hard to prove software correct; instead we hammer it until we can't prove it wrong.

클라이언트는 영원히 반복됩니다. 시작 또는 장애조치 중에 일부 클라이언트들은 기본 서버와 통신을 시도하고 다른 클라이언트들은 백업 서버와 통신을 시도할 것입니다. 바이너리 스타 상태 머신은 이것을 정확하게 처리합니다. 소프트웨어가 정확하다는 것을 증명하는 것은 어렵지만 대신에 우리는 소프트웨어가 틀렸다는 것을 증명할 수 없을 때까지 두드려 봅니다.

;Failover happens as follows:

장애조치는 다음과 같이 이루어집니다. 

;* The client detects that primary server is no longer sending heartbeats, and concludes that it has died. The client connects to the backup server and requests a new state snapshot.
;* The backup server starts to receive snapshot requests from clients, and detects that primary server has gone, so it takes over as primary.
;* The backup server applies its pending list to its own hash table, and then starts to process state snapshot requests.

* 클라이언트는 기본 서버가 더 이상 심박를 보내지 않음을 감지하고 죽었다는 결론을 내립니다. 클라이언트는 백업 서버에 연결하고 신규 상태 스냅샷을 요청합니다.
* 백업 서버는 클라이언트로부터 스냅샷 요청을 받기 시작하고 기본 서버의 죽음을 감지하여 기본 서버로 인계합니다.
* 백업 서버는 "보류 목록"을 자신의 해시 테이블에 적용한 다음, 클라이언트의 상태 스냅샷 요청을 처리하기 시작합니다.

;When the primary server comes back online, it will:

기본 서버가 다시 온라인 상태가 되면 다음을 수행합니다.

;* Start up as passive server, and connect to the backup server as a Clone client.
;* Start to receive updates from clients, via its SUB socket.

* 비활성 상태로 서버를 시작하고, 복제 클라이언트로 백업 서버에 연결합니다.
* SUB 소켓을 통해 클라이언트들로부터 변경정보들을 수신하기 시작합니다.

;We make a few assumptions:

몇 가지 가정들은 다음과 같습니다.

;* At least one server will keep running. If both servers crash, we lose all server state and there's no way to recover it.
;* Multiple clients do not update the same hash keys at the same time. Client updates will arrive at the two servers in a different order. Therefore, the backup server may apply updates from its pending list in a different order than the primary server would or did. Updates from one client will always arrive in the same order on both servers, so that is safe.

* 하나 이상의 서버가 계속 실행됩니다. 2대의 서버들이 충돌하면 모든 서버 상태가 유실되고 복구할 방법이 없습니다.
* 여러 클라이언트들이 동일한 해시 테이블의 키를 동시에 변경하지 않습니다. 클라이언트들의 변경정보들은 다른 순서로 2대 서버들에 도달합니다. 따라서 백업 서버는 기본 서버와 다른 순서의 변경정보들을 보류 목록(Pending List)의 반영할 수 있습니다. 한 클라이언트의 변경정보들은 항상 동일한 순서로 2대 서버들에 도달하므로 안전합니다.

;Thus the architecture for our high-availability server pair using the Binary Star pattern has two servers and a set of clients that talk to both servers.

따라서 바이너리 스티 패턴을 사용하는 고가용성 서버 쌍의 아키텍처에는 2대의 서버와 서버들과 통신하는 일련의 클라이언트들이 있습니다.

그림 62 - 고가용성 복제 서버 쌍

![High-availability Clone Server Pair](images/fig62.svg)

;Here is the sixth and last model of the Clone server:

여기에 6번째로 복제 서버의 마지막 모델(모델 6)의 코드입니다.

clonesrv6.c : Clone server, Model Six in C

```cpp
//  Clone server Model Six

//  Lets us build this source without creating a library
#include "bstar.c"
#include "kvmsg.c"

//  .split definitions
//  We define a set of reactor handlers and our server object structure:

//  Bstar reactor handlers
static int
    s_snapshots   (zloop_t *loop, zmq_pollitem_t *poller, void *args);
static int
    s_collector   (zloop_t *loop, zmq_pollitem_t *poller, void *args);
static int
    s_flush_ttl   (zloop_t *loop, int timer_id, void *args);
static int
    s_send_hugz   (zloop_t *loop, int timer_id, void *args);
static int
    s_new_active  (zloop_t *loop, zmq_pollitem_t *poller, void *args);
static int
    s_new_passive (zloop_t *loop, zmq_pollitem_t *poller, void *args);
static int
    s_subscriber  (zloop_t *loop, zmq_pollitem_t *poller, void *args);

//  Our server is defined by these properties
typedef struct {
    zctx_t *ctx;                //  Context wrapper
    zhash_t *kvmap;             //  Key-value store
    bstar_t *bstar;             //  Bstar reactor core
    int64_t sequence;           //  How many updates we're at
    int port;                   //  Main port we're working on
    int peer;                   //  Main port of our peer
    void *publisher;            //  Publish updates and hugz
    void *collector;            //  Collect updates from clients
    void *subscriber;           //  Get updates from peer
    zlist_t *pending;           //  Pending updates from clients
    bool primary;               //  true if we're primary
    bool active;                //  true if we're active
    bool passive;               //  true if we're passive
} clonesrv_t;

//  .split main task setup
//  The main task parses the command line to decide whether to start
//  as a primary or backup server. We're using the Binary Star pattern
//  for reliability. This interconnects the two servers so they can
//  agree on which one is primary and which one is backup. To allow the
//  two servers to run on the same box, we use different ports for 
//  primary and backup. Ports 5003/5004 are used to interconnect the 
//  servers. Ports 5556/5566 are used to receive voting events (snapshot 
//  requests in the clone pattern). Ports 5557/5567 are used by the 
//  publisher, and ports 5558/5568 are used by the collector:

int main (int argc, char *argv [])
{
    clonesrv_t *self = (clonesrv_t *) zmalloc (sizeof (clonesrv_t));
    if (argc == 2 && streq (argv [1], "-p")) {
        zclock_log ("I: primary active, waiting for backup (passive)");
        self->bstar = bstar_new (BSTAR_PRIMARY, "tcp://*:5003",
                                 "tcp://localhost:5004");
        bstar_voter (self->bstar, "tcp://*:5556",
                     ZMQ_ROUTER, s_snapshots, self);
        self->port = 5556;
        self->peer = 5566;
        self->primary = true;
    }
    else
    if (argc == 2 && streq (argv [1], "-b")) {
        zclock_log ("I: backup passive, waiting for primary (active)");
        self->bstar = bstar_new (BSTAR_BACKUP, "tcp://*:5004",
                                 "tcp://localhost:5003");
        bstar_voter (self->bstar, "tcp://*:5566",
                     ZMQ_ROUTER, s_snapshots, self);
        self->port = 5566;
        self->peer = 5556;
        self->primary = false;
    }
    else {
        printf ("Usage: clonesrv6 { -p | -b }\n");
        free (self);
        exit (0);
    }
    //  Primary server will become first active
    if (self->primary)
        self->kvmap = zhash_new ();

    self->ctx = zctx_new ();
    self->pending = zlist_new ();
    bstar_set_verbose (self->bstar, true);

    //  Set up our clone server sockets
    self->publisher = zsocket_new (self->ctx, ZMQ_PUB);
    self->collector = zsocket_new (self->ctx, ZMQ_SUB);
    zsocket_set_subscribe (self->collector, "");
    zsocket_bind (self->publisher, "tcp://*:%d", self->port + 1);
    zsocket_bind (self->collector, "tcp://*:%d", self->port + 2);

    //  Set up our own clone client interface to peer
    self->subscriber = zsocket_new (self->ctx, ZMQ_SUB);
    zsocket_set_subscribe (self->subscriber, "");
    zsocket_connect (self->subscriber,
                     "tcp://localhost:%d", self->peer + 1);

    //  .split main task body
    //  After we've setup our sockets, we register our binary star
    //  event handlers, and then start the bstar reactor. This finishes
    //  when the user presses Ctrl-C or when the process receives a SIGINT
    //  interrupt:

    //  Register state change handlers
    bstar_new_active (self->bstar, s_new_active, self);
    bstar_new_passive (self->bstar, s_new_passive, self);

    //  Register our other handlers with the bstar reactor
    zmq_pollitem_t poller = { self->collector, 0, ZMQ_POLLIN };
    zloop_poller (bstar_zloop (self->bstar), &poller, s_collector, self);
    zloop_timer  (bstar_zloop (self->bstar), 1000, 0, s_flush_ttl, self);
    zloop_timer  (bstar_zloop (self->bstar), 1000, 0, s_send_hugz, self);

    //  Start the bstar reactor
    bstar_start (self->bstar);

    //  Interrupted, so shut down
    while (zlist_size (self->pending)) {
        kvmsg_t *kvmsg = (kvmsg_t *) zlist_pop (self->pending);
        kvmsg_destroy (&kvmsg);
    }
    zlist_destroy (&self->pending);
    bstar_destroy (&self->bstar);
    zhash_destroy (&self->kvmap);
    zctx_destroy (&self->ctx);
    free (self);

    return 0;
}

//  We handle ICANHAZ? requests exactly as in the clonesrv5 example.
//  .skip

//  Routing information for a key-value snapshot
typedef struct {
    void *socket;           //  ROUTER socket to send to
    zframe_t *identity;     //  Identity of peer who requested state
    char *subtree;          //  Client subtree specification
} kvroute_t;

//  Send one state snapshot key-value pair to a socket
//  Hash item data is our kvmsg object, ready to send
static int
s_send_single (const char *key, void *data, void *args)
{
    kvroute_t *kvroute = (kvroute_t *) args;
    kvmsg_t *kvmsg = (kvmsg_t *) data;
    if (strlen (kvroute->subtree) <= strlen (kvmsg_key (kvmsg))
    &&  memcmp (kvroute->subtree,
                kvmsg_key (kvmsg), strlen (kvroute->subtree)) == 0) {
        zframe_send (&kvroute->identity,    //  Choose recipient
            kvroute->socket, ZFRAME_MORE + ZFRAME_REUSE);
        kvmsg_send (kvmsg, kvroute->socket);
    }
    return 0;
}

static int
s_snapshots (zloop_t *loop, zmq_pollitem_t *poller, void *args)
{
    clonesrv_t *self = (clonesrv_t *) args;

    zframe_t *identity = zframe_recv (poller->socket);
    if (identity) {
        //  Request is in second frame of message
        char *request = zstr_recv (poller->socket);
        char *subtree = NULL;
        if (streq (request, "ICANHAZ?")) {
            free (request);
            subtree = zstr_recv (poller->socket);
        }
        else
            printf ("E: bad request, aborting\n");

        if (subtree) {
            //  Send state socket to client
            kvroute_t routing = { poller->socket, identity, subtree };
            zhash_foreach (self->kvmap, s_send_single, &routing);

            //  Now send END message with sequence number
            zclock_log ("I: sending shapshot=%d", (int) self->sequence);
            zframe_send (&identity, poller->socket, ZFRAME_MORE);
            kvmsg_t *kvmsg = kvmsg_new (self->sequence);
            kvmsg_set_key  (kvmsg, "KTHXBAI");
            kvmsg_set_body (kvmsg, (byte *) subtree, 0);
            kvmsg_send     (kvmsg, poller->socket);
            kvmsg_destroy (&kvmsg);
            free (subtree);
        }
        zframe_destroy(&identity);
    }
    return 0;
}
//  .until

//  .split collect updates
//  The collector is more complex than in the clonesrv5 example because the 
//  way it processes updates depends on whether we're active or passive. 
//  The active applies them immediately to its kvmap, whereas the passive 
//  queues them as pending:

//  If message was already on pending list, remove it and return true,
//  else return false.
static int
s_was_pending (clonesrv_t *self, kvmsg_t *kvmsg)
{
    kvmsg_t *held = (kvmsg_t *) zlist_first (self->pending);
    while (held) {
        if (memcmp (kvmsg_uuid (kvmsg),
                    kvmsg_uuid (held), sizeof (uuid_t)) == 0) {
            zlist_remove (self->pending, held);
            return true;
        }
        held = (kvmsg_t *) zlist_next (self->pending);
    }
    return false;
}

static int
s_collector (zloop_t *loop, zmq_pollitem_t *poller, void *args)
{
    clonesrv_t *self = (clonesrv_t *) args;

    kvmsg_t *kvmsg = kvmsg_recv (poller->socket);
    if (kvmsg) {
        if (self->active) {
            kvmsg_set_sequence (kvmsg, ++self->sequence);
            kvmsg_send (kvmsg, self->publisher);
            int ttl = atoi (kvmsg_get_prop (kvmsg, "ttl"));
            if (ttl)
                kvmsg_set_prop (kvmsg, "ttl",
                    "%" PRId64, zclock_time () + ttl * 1000);
            kvmsg_store (&kvmsg, self->kvmap);
            zclock_log ("I: publishing update=%d", (int) self->sequence);
        }
        else {
            //  If we already got message from active, drop it, else
            //  hold on pending list
            if (s_was_pending (self, kvmsg))
                kvmsg_destroy (&kvmsg);
            else
                zlist_append (self->pending, kvmsg);
        }
    }
    return 0;
}

//  We purge ephemeral values using exactly the same code as in
//  the previous clonesrv5 example.
//  .skip
//  If key-value pair has expired, delete it and publish the
//  fact to listening clients.
static int
s_flush_single (const char *key, void *data, void *args)
{
    clonesrv_t *self = (clonesrv_t *) args;

    kvmsg_t *kvmsg = (kvmsg_t *) data;
    int64_t ttl;
    sscanf (kvmsg_get_prop (kvmsg, "ttl"), "%" PRId64, &ttl);
    if (ttl && zclock_time () >= ttl) {
        kvmsg_set_sequence (kvmsg, ++self->sequence);
        kvmsg_set_body (kvmsg, (byte *) "", 0);
        kvmsg_send (kvmsg, self->publisher);
        kvmsg_store (&kvmsg, self->kvmap);
        zclock_log ("I: publishing delete=%d", (int) self->sequence);
    }
    return 0;
}

static int
s_flush_ttl (zloop_t *loop, int timer_id, void *args)
{
    clonesrv_t *self = (clonesrv_t *) args;
    if (self->kvmap)
        zhash_foreach (self->kvmap, s_flush_single, args);
    return 0;
}
//  .until

//  .split heartbeating
//  We send a HUGZ message once a second to all subscribers so that they
//  can detect if our server dies. They'll then switch over to the backup
//  server, which will become active:

static int
s_send_hugz (zloop_t *loop, int timer_id, void *args)
{
    clonesrv_t *self = (clonesrv_t *) args;

    kvmsg_t *kvmsg = kvmsg_new (self->sequence);
    kvmsg_set_key  (kvmsg, "HUGZ");
    kvmsg_set_body (kvmsg, (byte *) "", 0);
    kvmsg_send     (kvmsg, self->publisher);
    kvmsg_destroy (&kvmsg);

    return 0;
}

//  .split handling state changes
//  When we switch from passive to active, we apply our pending list so that
//  our kvmap is up-to-date. When we switch to passive, we wipe our kvmap
//  and grab a new snapshot from the active server:

static int
s_new_active (zloop_t *loop, zmq_pollitem_t *unused, void *args)
{
    clonesrv_t *self = (clonesrv_t *) args;

    self->active = true;
    self->passive = false;

    //  Stop subscribing to updates
    zmq_pollitem_t poller = { self->subscriber, 0, ZMQ_POLLIN };
    zloop_poller_end (bstar_zloop (self->bstar), &poller);

    //  Apply pending list to own hash table
    while (zlist_size (self->pending)) {
        kvmsg_t *kvmsg = (kvmsg_t *) zlist_pop (self->pending);
        kvmsg_set_sequence (kvmsg, ++self->sequence);
        kvmsg_send (kvmsg, self->publisher);
        kvmsg_store (&kvmsg, self->kvmap);
        zclock_log ("I: publishing pending=%d", (int) self->sequence);
    }
    return 0;
}

static int
s_new_passive (zloop_t *loop, zmq_pollitem_t *unused, void *args)
{
    clonesrv_t *self = (clonesrv_t *) args;

    zhash_destroy (&self->kvmap);
    self->active = false;
    self->passive = true;

    //  Start subscribing to updates
    zmq_pollitem_t poller = { self->subscriber, 0, ZMQ_POLLIN };
    zloop_poller (bstar_zloop (self->bstar), &poller, s_subscriber, self);

    return 0;
}

//  .split subscriber handler
//  When we get an update, we create a new kvmap if necessary, and then
//  add our update to our kvmap. We're always passive in this case:

static int
s_subscriber (zloop_t *loop, zmq_pollitem_t *poller, void *args)
{
    clonesrv_t *self = (clonesrv_t *) args;
    //  Get state snapshot if necessary
    if (self->kvmap == NULL) {
        self->kvmap = zhash_new ();
        void *snapshot = zsocket_new (self->ctx, ZMQ_DEALER);
        zsocket_connect (snapshot, "tcp://localhost:%d", self->peer);
        zclock_log ("I: asking for snapshot from: tcp://localhost:%d",
                    self->peer);
        zstr_sendm (snapshot, "ICANHAZ?");
        zstr_send (snapshot, ""); // blank subtree to get all
        while (true) {
            kvmsg_t *kvmsg = kvmsg_recv (snapshot);
            if (!kvmsg)
                break;          //  Interrupted
            if (streq (kvmsg_key (kvmsg), "KTHXBAI")) {
                self->sequence = kvmsg_sequence (kvmsg);
                kvmsg_destroy (&kvmsg);
                break;          //  Done
            }
            kvmsg_store (&kvmsg, self->kvmap);
        }
        zclock_log ("I: received snapshot=%d", (int) self->sequence);
        zsocket_destroy (self->ctx, snapshot);
    }
    //  Find and remove update off pending list
    kvmsg_t *kvmsg = kvmsg_recv (poller->socket);
    if (!kvmsg)
        return 0;

    if (strneq (kvmsg_key (kvmsg), "HUGZ")) {
        if (!s_was_pending (self, kvmsg)) {
            //  If active update came before client update, flip it
            //  around, store active update (with sequence) on pending
            //  list and use to clear client update when it comes later
            zlist_append (self->pending, kvmsg_dup (kvmsg));
        }
        //  If update is more recent than our kvmap, apply it
        if (kvmsg_sequence (kvmsg) > self->sequence) {
            self->sequence = kvmsg_sequence (kvmsg);
            kvmsg_store (&kvmsg, self->kvmap);
            zclock_log ("I: received update=%d", (int) self->sequence);
        }
        else
            kvmsg_destroy (&kvmsg);
    }
    else
        kvmsg_destroy (&kvmsg);

    return 0;
}
```

;This model is only a few hundred lines of code, but it took quite a while to get working. To be accurate, building Model Six took about a full week of "Sweet god, this is just too complex for an example" hacking. We've assembled pretty much everything and the kitchen sink into this small application. We have failover, ephemeral values, subtrees, and so on. What surprised me was that the up-front design was pretty accurate. Still the details of writing and debugging so many socket flows is quite challenging.

이 모델은 수백 줄의 코드에 불과하지만 작동하기 위해 많은 시간이 소요되었습니다. 정확히 말하면 모델 6을 구현하는데 대략 1주일이 걸렸으며 "신이시여, 이건 예제로 하기에는 너무 복잡합니다." 하며 삽질하였습니다. 우리는 작은 응용프로그램에 지금까지 적용한 거의 개념들을 모아 조립하였습니다. 바이너리 스타, TTL, 유한 상태 머신, 리엑터, 장애조치, 임시값, 하위트리 등이 있습니다. 나를 놀라게 한 것은 앞선 설계가 꽤 정확하다는 것입니다. 여전히 많은 소켓 흐름을 작성하고 디버깅하는 것은 매우 어렵습니다.

;The reactor-based design removes a lot of the grunt work from the code, and what remains is simpler and easier to understand. We reuse the bstar reactor from Chapter 4 - Reliable Request-Reply Patterns. The whole server runs as one thread, so there's no inter-thread weirdness going on—just a structure pointer (self) passed around to all handlers, which can do their thing happily. One nice side effect of using reactors is that the code, being less tightly integrated into a poll loop, is much easier to reuse. Large chunks of Model Six are taken from Model Five.

리엑터 기반 설계는 코드에서 많은 지저분한 작업을 제거하여, 남은 것을 더 간단하고 이해하기 쉽게 합니다. "4장 - 신뢰할 수 있는 요청-응답 패턴"의 bstar 리액터를 재사용하였습니다. 전체 서버가 하나의 스레드로 실행되므로 스레드 간 이상함이 일어나지 않습니다 - 모든 핸들러들에 전달된 구조체 포인터(self)만으로 즐거운 작업을 수행합니다. 리엑터 사용 시 하나의 좋은 효과는 폴링 루프와 약연결(loosely coupled)된 코드는 재사용하기 훨씬 쉽다는 것입니다. 모델 6의 큰 덩어리는 모델 5에서 가져왔습니다.

;I built it piece by piece, and got each piece working properly before going onto the next one. Because there are four or five main socket flows, that meant quite a lot of debugging and testing. I debugged just by dumping messages to the console. Don't use classic debuggers to step through ØMQ applications; you need to see the message flows to make any sense of what is going on.

각 기능들을 하나씩 하나씩 구현하여, 다음 기능으로 넘어가기 전에 각 기능이 정상적으로 동작하게 하였습니다. 4~5개의 주요 소켓 흐름이 있기 때문에 많은 디버깅과 테스트가 필요했습니다. 디버깅은 컴퓨터 화면에 메시지를 덤프하여 수행하였습니다. ØMQ 응용프로그램을 단계별로 진행하기 위해 고적적인 디버거(gdb 등)를 사용하지 마십시오. 무슨 일이 일어나고 있는지 이해하려면 메시지 흐름을 확인해야 합니다.

;For testing, I always try to use Valgrind, which catches memory leaks and invalid memory accesses. In C, this is a major concern, as you can't delegate to a garbage collector. Using proper and consistent abstractions like kvmsg and CZMQ helps enormously.

테스트를 위해 항상 메모리 누수와 잘못된 메모리 참조를 잡기 위해 Valgrind를 사용하였습니다. 
C 언어의 주요 관심사로 컴파일러에서 가비지(garbage) 수집을 하지 않아 코드상에서 명시적으로 메모리 해제를 않으면 메모리 누수가 발생할 수 있습니다. 이때 kvmsg 및 CZMQ와 같은 적절하고 일관된 추상화를 사용하면 엄청난 도움이 됩니다.

> [옮긴이] "clonecli6.c" 상태(INITIAL, SYNCING, ACTIVE)를 PUSH-PULL을 PUB-SUB로 변경한 클라이언트의 마지막 모델(모델 6)로 다음 주제에서 설명합니다.

> [옮긴이] 빌드 및 테스트

~~~{.bash}
PS D:\git_store\zguide-kr\examples\C> cl -EHsc clonesrv6.c libzmq.lib czmq.lib
PS D:\git_store\zguide-kr\examples\C> cl -EHsc clonecli6.c libzmq.lib czmq.lib

PS D:\git_store\zguide-kr\examples\C> ./clonesrv6 -p
20-08-29 16:13:24 I: primary active, waiting for backup (passive)
D: 20-08-29 16:13:24 zloop: register SUB poller (000001EEAEB11AC0, 0)
D: 20-08-29 16:13:24 zloop: register timer id=2 delay=1000 times=0
D: 20-08-29 16:13:24 zloop: register timer id=3 delay=1000 times=0
D: 20-08-29 16:13:24 zloop polling for 986 msec
D: 20-08-29 16:13:25 zloop: call timer handler id=1
...
20-08-29 16:13:41 I: connected to backup (passive), ready as active
D: 20-08-29 16:13:41 zloop: cancel SUB poller (000001EEAE518160, 0)
D: 20-08-29 16:13:41 zloop polling for 346 msec
D: 20-08-29 16:13:41 zloop: call timer handler id=1
...
D: 20-08-29 16:14:03 zloop: interrupted

PS D:\git_store\zguide-kr\examples\C> ./clonesrv6 -b
20-08-29 16:13:40 I: backup passive, waiting for primary (active)
D: 20-08-29 16:13:40 zloop: register SUB poller (000001F9CDC93690, 0)
D: 20-08-29 16:13:40 zloop: register timer id=2 delay=1000 times=0
D: 20-08-29 16:13:40 zloop: register timer id=3 delay=1000 times=0
D: 20-08-29 16:13:40 zloop polling for 985 msec
D: 20-08-29 16:13:40 zloop: call SUB socket handler (000001F9CD681270, 0)
...
D: 20-08-29 16:13:41 zloop: call SUB socket handler (000001F9CD681270, 0)
20-08-29 16:13:41 I: connected to primary (active), ready as passive
...
0-08-29 16:14:07 I: failover successful, ready as active
D: 20-08-29 16:14:07 zloop: cancel SUB poller (000001F9CD694FA0, 0)

PS D:\git_store\zguide-kr\examples\C> ./clonecli6
20-08-29 16:13:51 I: adding server tcp://localhost:5556...
20-08-29 16:13:51 I: waiting for server at tcp://localhost:5556...
20-08-29 16:13:51 I: adding server tcp://localhost:5566...
20-08-29 16:13:51 I: received from tcp://localhost:5556 snapshot=0
20-08-29 16:13:52 I: received from tcp://localhost:5556 update=1
20-08-29 16:13:53 I: received from tcp://localhost:5556 update=2
...
20-08-29 16:14:07 I: server at tcp://localhost:5556 didn't give HUGZ
20-08-29 16:14:07 I: waiting for server at tcp://localhost:5566...
~~~

;### The Clustered Hashmap Protocol

### 클러스터된 해시맵 통신규약

;While the server is pretty much a mashup of the previous model plus the Binary Star pattern, the client is quite a lot more complex. But before we get to that, let's look at the final protocol. I've written this up as a specification on the ØMQ RFC website as the Clustered Hashmap Protocol.

모델 6 서버는 이전 모델(모델 5(임시값 + TTL))에 바이너리 스타 패턴을 매우 많이 혼합시켰지만, 모델 6 클라이언트는 좀 더 많이 복잡합니다. 클라이언트에 대해 알아보기 전에 최종 통신규약을 보면, 클러스터된 해시맵 통신규약으로 ØMQ RFC 사이트에 사양서로 작성해 두었습니다.
>[옮긴이] [클러스터된 해시 맵 통신규약](https://rfc.zeromq.org/spec/12/)은 일련의 클라이언트들 간에 공유하기 위한 클러스터 전반의 키-값 해시 맵과 메커니즘을 정의합니다.

;Roughly, there are two ways to design a complex protocol such as this one. One way is to separate each flow into its own set of sockets. This is the approach we used here. The advantage is that each flow is simple and clean. The disadvantage is that managing multiple socket flows at once can be quite complex. Using a reactor makes it simpler, but still, it makes a lot of moving pieces that have to fit together correctly.

대략적으로 이와 같은 복잡한 통신규약을 설계하는 방법에는 두 가지가 있습니다. 첫 번째 방법은 각 흐름을 자체 소켓 집합으로 분리하는 것입니다. 이것이 우리가 여기서 사용한 접근 방식입니다. 장점은 각 흐름이 단순하고 깔끔하다는 것입니다. 단점은 한 번에 다중 소켓 흐름을 관리하는 것이 매우 복잡할 수 있습니다. 리엑터를 사용하면 더 단순해지지만 그래도 정상적으로 동작하게 하기 위해 함께 조정할 부분들이 많습니다.

;The second way to make such a protocol is to use a single socket pair for everything. In this case, I'd have used ROUTER for the server and DEALER for the clients, and then done everything over that connection. It makes for a more complex protocol but at least the complexity is all in one place. In Chapter 7 - Advanced Architecture using ØMQ we'll look at an example of a protocol done over a ROUTER-DEALER combination.

두 번째 방법은 모든 것에 단일 소켓 쌍을 사용하는 것입니다. 이 경우 서버에는 ROUTER를, 클라이언트들에는 DEALER를 사용하여 해당 연결상에서 모든 작업을 수행했습니다. 
이러한 방법은 더 복잡한 통신규약을 만들지만 적어도 복잡성은 모두 한곳에 집중되어 있습니다. "7장 - ØMQ 활용한 고급 아키텍처"에서 ROUTER-DEALER 조합을 통해 수행되는 통신규약의 예를 살펴보겠습니다.

;Let's take a look at the CHP specification. Note that "SHOULD", "MUST" and "MAY" are key words we use in protocol specifications to indicate requirement levels.

[CHP](https://rfc.zeromq.org/spec/12/) 사양서를 살펴보겠습니다. "SHOULD", "MUST"및 "MAY"는 통신규약 사양서에서 요구 사항 수준을 나타내기 위해 사용되는 핵심 용어라는 점에 주의하십시오.

;#### Goals
#### 목표

;CHP is meant to provide a basis for reliable pub-sub across a cluster of clients connected over a ØMQ network. It defines a "hashmap" abstraction consisting of key-value pairs. Any client can modify any key-value pair at any time, and changes are propagated to all clients. A client can join the network at any time.

CHP는 ØMQ 네트워크에 연결된 클라이언트들의 클러스터에서 신뢰할 수 있는 발행-구독에 대한 기반을 제공합니다. "해시 맵" 추상화를 키-값 쌍으로 구성하여 정의합니다. 모든 클라이언트들은 언제든지 키-값 쌍을 수정할 수 있으며, 변경 사항은 모든 클라이언트들에게 전파됩니다. 클라이언트는 언제든지 네트워크에 참여할 수 있습니다.

;#### Architecture
#### 아키텍처

;CHP connects a set of client applications and a set of servers. Clients connect to the server. Clients do not see each other. Clients can come and go arbitrarily.

CHP는 일련의 클라이언트와 서버 응용프로그램들을 연결합니다. 클라이언트들은 서버에 연결합니다. 클라이언트는 서로를 보지 못합니다. 클라이언트들은 언제든지 클러스터에 들어오고 나갈 수 있습니다.

;#### Ports and Connections
#### 포트들과 접속들

;The server MUST open three ports as follows:
;* A SNAPSHOT port (ØMQ ROUTER socket) at port number P.
;* A PUBLISHER port (ØMQ PUB socket) at port number P + 1.
;* A COLLECTOR port (ØMQ SUB socket) at port number P + 2.

서버는 3개의 포트를 오픈하며 다음과 같습니다.
* 스냅샷 포트(ØMQ ROUTER 소켓)로 포트 번호는 P.
* 발행자 포트(ØMQ PUB 소켓)로 포트 번호는 P+1.
* 수집자 포트(ØMQ SUB 소켓)로 포트 번호는 P+2.

;The client SHOULD open at least two connections:
;* A SNAPSHOT connection (ØMQ DEALER socket) to port number P.
;* A SUBSCRIBER connection (ØMQ SUB socket) to port number P + 1.
;The client MAY open a third connection, if it wants to update the hashmap:
;* A PUBLISHER connection (ØMQ PUB socket) to port number P + 2.

클라이언트는 적어도 2개의 연결들이 오픈되어야 합니다.
* 스냅샷 연결(ØMQ DEALER 소켓)로 포트 번호는 P.
* 구독자 연결(ØMQ SUB 소켓)로 포트 번호는 P+1.
클라이언트는 3번째 연결을하여 해시맵 변경을 서버로 전달하면, 서버에서 각 클라이언트들에게 전파합니다.
* 발행자 연결(ØMQ PUB 소켓)으로 포트 번호는 P+2

;This extra frame is not shown in the commands explained below.

명령(SUBTREE, SET, GET, CONNECT)에 있는 추가 프레임은 아래에서 설명합니다.

;#### State Synchronization

#### 상태 동기화

;The client MUST start by sending a ICANHAZ command to its snapshot connection. This command consists of two frames as follows:

; 클라이언트는 스냅샷 연결(포트 번호 P)에 "ICANHAZ?" 명령을 전송하여 시작해야 합니다. 이 명령은 다음과 같은 두 개의 프레임으로 구성됩니다.

~~~{.bash}
[클라이언트] ICANHAZ 명령
-----------------------------------
Frame 0: "ICANHAZ?"
Frame 1: 하위트리 사양서(예 : "/client/")
~~~

;Both frames are ØMQ strings. The subtree specification MAY be empty. If not empty, it consists of a slash followed by one or more path segments, ending in a slash.
;The server MUST respond to a ICANHAZ command by sending zero or more KVSYNC commands to its snapshot port, followed with a KTHXBAI command. The server MUST prefix each command with the identity of the client, as provided by ØMQ with the ICANHAZ command. The KVSYNC command specifies a single key-value pair as follows:

2개 프레임들은 모두 ØMQ 문자열입니다. 하위트리 사양은 공백("")일 수 있지만 
공백이 아닐 경우 (`clone_subtree()`로 설정) 슬래시(/) 이후 구성되는 경로명이며, 슬래시(/)로 끝납니다.(예 : "/client/")
서버는 스냅샷 포트(ROUTER)로부터 "ICANHAZ?" 받고 0개 이상의 KVSYNC 명령으로 스냅샷 포트에 kvmsg를 전송 한 다음 "KTHXBAI" 명령을 전송합니다. 서버는 "ICANHAZ?" 명령에 의해 제공되는 클라이언트 식별자(ID)를 각 메시지들의 선두에 부여합니다. KVSYNC 명령은 다음과 같이 단일 키-값 쌍을 지정합니다.

~~~{.bash}
[섭버] KVSYNC 명령
-----------------------------------
Frame 0: 키, ØMQ 문자열
Frame 1: 순서 번호, 8 바이트 네트워크 순서
Frame 2: <공백>                --> UUID
Frame 3: <공백>                --> PROPERTIES
Frame 4: 값, 이진대형객체(Blob : Binary Large Object)
~~~

;The sequence number has no significance and may be zero.
The KTHXBAI command takes this form:

순서 번호는 중요하지 않으며 0일 수도 있습니다.
KTHXBAI 명령은 다음과 같은 형태입니다.

~~~{.bash}
[서버] KTHXBAI 명령
-----------------------------------
Frame 0: "KTHXBAI"
Frame 1: 순서 번호, 8 바이트 네트워크 순서
Frame 2: <공백>
Frame 3: <공백>
Frame 4: 서브트리 사양서(예 : "/client/")
~~~

;The sequence number MUST be the highest sequence number of the KVSYNC commands previously sent.
;When the client has received a KTHXBAI command, it SHOULD start to receive messages from its subscriber connection and apply them.

순서 번호는 이전에 보낸 KVSYNC 명령의 가장 높은 순서 번호입니다.
클라이언트의 스냅샷 소켓(DEALER)를 통해 KTHXBAI 명령을 수신하면, 구독자 연결(SUB)에서 메시지를 수신하고 적용하기 시작합니다(SHOULD).

;#### Server-to-Client Updates

#### 서버에서 클라이언트로 전달되는 변경정보들

;When the server has an update for its hashmap it MUST broadcast this on its publisher socket as a KVPUB command. The KVPUB command has this form:

서버에 해시 맵에 대한 변경정보가 있을 때 발행자 소켓(PUB)에 KVPUB 명령으로 브로드캐스트 해야 합니다. KVPUB 명령의 형식은 다음과 같습니다.

~~~{.bash}
[서버] KVPUB 명령
-----------------------------------
Frame 0: 키, ØMQ 문자열
Frame 1: 순서 번호, 8 바이트 네트워크 순서
Frame 2: UUID, 16 바이트
Frame 3: 추가 속성들, ØMQ 문자열
Frame 4: 값, 이진대형객체(Blob : Binary Large Object)
~~~

;The sequence number MUST be strictly incremental. The client MUST discard any KVPUB commands whose sequence numbers are not strictly greater than the last KTHXBAI or KVPUB command received.
;The UUID is optional and frame 2 MAY be empty (size zero). The properties field is formatted as zero or more instances of "name=value" followed by a newline character. If the key-value pair has no properties, the properties field is empty.
;If the value is empty, the client SHOULD delete its key-value entry with the specified key.

순서 번호는 반드시 증가해야 합니다. 클라이언트는 순서 번호가 마지막으로 수신된 KTHXBAI 또는 KVPUB 명령보다 크지 않은 모든 KVPUB 명령을 폐기됩니다.
UUID는 선택 사항이며 "Frame 2"는 공백일 수 있습니다(크기 0). 추가 속성들은 0개 이상의 "name 
= value"형태와 뒤에 개행 문자로 구성됩니다. 키-값 쌍에 추가 속성들이 없는 경우 속성 필드는 공백입니다.

;In the absence of other updates the server SHOULD send a HUGZ command at regular intervals, e.g., once per second. The HUGZ command has this format:

다른 변경정보들이 없는 경우 서버는 일정한 간격(예 : 1초당 한번)으로 HUGZ 명령을 발행자 소켓(PUB)으로 보내야 합니다. HUGZ 명령의 형식은 다음과 같습니다.
> [옮긴이] HUGZ는 상대편 서버 및 클라이언트들에게 보냅니다.

~~~{.bash}
[서버] HUGZ 명령
-----------------------------------
Frame 0: "HUGZ"
Frame 1: 00000000
Frame 2: <공백>
Frame 3: <공백>
Frame 4: <공백>
~~~

;The client MAY treat the absence of HUGZ as an indicator that the server has crashed (see Reliability below).

클라이언트는 일정한 간격으로 HUGZ 메시지가 없을 경우, 서버에 장애가 발생했다는 표시로 사용됩니다(아래 안정성 참조).

;#### Client-to-Server Updates
#### 클라이언트에서 서버로 전달되는 변경정보들

;When the client has an update for its hashmap, it MAY send this to the server via its publisher connection as a KVSET command. The KVSET command has this form:

클라이언트가 해시 맵에 대한 변경정보를 가지고 있을 때, KVSET 명령으로 발행자 연결(PUB)을 통해 변경정보를 서버에 보낼 수 있습니다. KVSET 명령의 형식은 다음과 같습니다.

~~~{.bash}
[클라이언트] KVSET 명령
-----------------------------------
Frame 0: 키, ØMQ 문자열
Frame 1: 순서 번호, 8 바이트 네트워크 순서
Frame 2: UUID, 16 바이트
Frame 3: 추가 속성들, ØMQ 문자열
Frame 4: 값, 이진대형객체(Blob : Binary Large Object)
~~~

;The sequence number has no significance and may be zero. The UUID SHOULD be a universally unique identifier, if a reliable server architecture is used.
If the value is empty, the server MUST delete its key-value entry with the specified key.
The server SHOULD accept the following properties:
;* ttl: specifies a time-to-live in seconds. If the KVSET command has a ttl property, the server SHOULD delete the key-value pair and broadcast a KVPUB with an empty value in order to delete this from all clients when the TTL has expired.

순서 번호는 중요하지 않으며 0일 수 있습니다. UUID는 고유한 식별자로 신뢰성 있는 서버 아키텍처에서 사용됩니다. 만약 값이 공백(서버에서 TTL에 의한 값을 공백으로 치환)이면, 서버는 해당 키에 대한 키-값 항목을 삭제해야 합니다.
서버는 다음 속성을 수락해야 합니다.
* TTL(Time to Live) : 유효시간(초)을 지정합니다. KVSET 명령에 ttl 속성이 있는 경우, 서버는 키-값 쌍을 삭제하고 값이 비어있는 kvsmg를 KVPUB 명령을 통해 클라이언트들로 전송해야 하며, 클라이언트에서는 TTL이 만료되었을 때 삭제합니다.

;#### Reliability
#### 안정성

;CHP may be used in a dual-server configuration where a backup server takes over if the primary server fails. CHP does not specify the mechanisms used for this failover but the Binary Star pattern may be helpful.
To assist server reliability, the client MAY:

CHP는 기본 서버가 실패할 경우 백업 서버가 인계받는 이중 서버 구성으로 사용할 수 있습니다. CHP는 이런 장애조치에 사용되는 메커니즘을 정의하지 않지만 바이너리 스타 패턴이 도움이 될 수 있습니다.
서버 안정성을 지원하기 위해 클라이언트는 다음을 수행할 수 있습니다.

;* Set a UUID in every KVSET command.
;* Detect the lack of HUGZ over a time period and use this as an indicator that the current server has failed.
;* Connect to a backup server and re-request a state synchronization.

* 모든 KVSET 명령에서 UUID를 설정합니다.
* 일정 기간 동안 HUGZ 메시지 부재 감지하고 이를 현재 서버가 실패했음을 나타내는 표시로 사용합니다.
* 백업 서버에 연결하고 상태 동기화(ICANHAZ~KTXBAI)를 다시 요청합니다.

;#### Scalability and Performance
#### 확장성 및 성능

;CHP is designed to be scalable to large numbers (thousands) of clients, limited only by system resources on the broker. Because all updates pass through a single server, the overall throughput will be limited to some millions of updates per second at peak, and probably less.

CHP는 브로커의 많은 수(수천)의 클라이언트들로 확장 가능하도록 설계되었으며, 단지 브로커상의 시스템 자원에 의해 제한받습니다.
모든 클라이언트들의 변경정보들이 단일 서버를 통과하기 때문에, 전체 처리량은 피크시 초당 수백만 개의 변경정보들로 제한될 수 있으며, 아마도 더 적을 수 있습니다.

;#### Security
#### 보안

;CHP does not implement any authentication, access control, or encryption mechanisms and should not be used in any deployment where these are required.

CHP는 인증, 접근 제어 또는 암호화 메커니즘을 구현하지 않았으며, 이러한 메커니즘이 필요한 환경에서 사용해서는 안됩니다.

;### Building a Multithreaded Stack and API

### 멀티스레드 스택과 API 구축

;The client stack we've used so far isn't smart enough to handle this protocol properly. As soon as we start doing heartbeats, we need a client stack that can run in a background thread. In the Freelance pattern at the end of Chapter 4 - Reliable Request-Reply Patterns we used a multithreaded API but didn't explain it in detail. It turns out that multithreaded APIs are quite useful when you start to make more complex ØMQ protocols like CHP.

지금까지 사용한 클라이언트 스택은 CHP 통신규약을 제대로 처리할 만큼 똑똑하지 않습니다. 심박을 시작하자마자 백그라운드 스레드에서 실행할 수 있는 클라이언트 스택이 필요합니다. "4장 - 신뢰할 수 있는 요청-응답 패턴"의 마지막 있는 프리렌서 패턴에서 멀티스레드 API를 사용했지만 자세히 설명하지 않았습니다. 
멀티스레드 API는 CHP와 같은 복잡한 ØMQ 통신규약을 구현할 때 매우 유용합니다.

그림 63 - 멀티스레드 API

![Multithreaded API](images/fig63.svg)

;If you make a nontrivial protocol and you expect applications to implement it properly, most developers will get it wrong most of the time. You're going to be left with a lot of unhappy people complaining that your protocol is too complex, too fragile, and too hard to use. Whereas if you give them a simple API to call, you have some chance of them buying in.

중요한 통신규약을 만들고 응용프로그램에서 제대로 구현될 것으로 기대한다면, 대부분의 개발자는  잘못된 길로 갈 수가 있습니다. 당신은 통신규약이 너무 복잡하고, 너무 섬세하며, 너무 사용하기 어렵다고 불평하는 많은 불행한 사람들과 함께 남겨질 것입니다. 
개발자들에게 호출할 간단한 API를 제공할 수 있다면, 당신은 API를 구매하려 할 것입니다.

;Our multithreaded API consists of a frontend object and a background agent, connected by two PAIR sockets. Connecting two PAIR sockets like this is so useful that your high-level binding should probably do what CZMQ does, which is package a "create new thread with a pipe that I can use to send messages to it" method.

멜티스레드 API는 2개의 PAIR 소켓으로 연결된 프론트엔드 개체와 백그라운드 에이전트로 구성됩니다. 이와 같이 2개의 PAIR 소켓을 연결하는 것은 매우 유용하여 ØMQ의 C 개발 언어에서 제공하는 고수준의 바인딩인 CZMQ에서 수행합니다. 이것은 "신규 스레드를 생성 시에 메시지를 보내는 데 사용할 수 있는 파이프를 사용"하는 방법입니다.

> [옮긴이] "clone.c"에서 아래와 같이 사용합니다.
 - self->pipe = zthread_fork (self->ctx, clone_agent, NULL);

;The multithreaded APIs that we see in this book all take the same form:

본 가이드에서 볼 수 있는 멀티스레드 API는 모두 동일한 형식을 가집니다.

;* The constructor for the object (clone_new) creates a context and starts a background thread connected with a pipe. It holds onto one end of the pipe so it can send commands to the background thread.
;* The background thread starts an agent that is essentially a zmq_poll loop reading from the pipe socket and any other sockets (here, the DEALER and SUB sockets).
;* The main application thread and the background thread now communicate only via ØMQ messages. By convention, the frontend sends string commands so that each method on the class turns into a message sent to the backend agent, like this:

* 객체의 생성자(`clone_new()`)는 컨텍스트(Context)를 생성하고 파이프로 연결된 백그라운드 스레드(`clone_agent()`)를 시작합니다. 파이프의 한쪽 끝을 잡고 있으므로 백그라운드 스레드에 명령을 보낼 수 있습니다.
* 백그라운드 스레드(`clone_agent()`)는 에이전트를 시작하여 기본적으로 `zmq_poll` 루프를 통하여 파이프 소켓과 다른 소켓들(DEALER, SUB 소켓)을 읽습니다.
* 메인 응용프로그램 스레드와 백그라운드 스레드는 ØMQ 메시지를 통하여 통신합니다. 규칙에 따라 프론트엔드는 다음과 같이 문자열 명령(CONNECT, GET, SET 등)을 보내면 클래스의 각 메서드가 백엔드 에이전트에 전송되는 메시지로 전환(`agent_control_message()`)합니다.

```cpp
void
clone_connect (clone_t *self, char *address, char *service)
{
    assert (self);
    zmsg_t *msg = zmsg_new ();
    zmsg_addstr (msg, "CONNECT");
    zmsg_addstr (msg, address);
    zmsg_addstr (msg, service);
    zmsg_send (&msg, self->pipe);
}
```

;* If the method needs a return code, it can wait for a reply message from the agent.
;* If the agent needs to send asynchronous events back to the frontend, we add a recv method to the class, which waits for messages on the frontend pipe.
;* We may want to expose the frontend pipe socket handle to allow the class to be integrated into further poll loops. Otherwise any recv method would block the application.

* 메서드에 반환 코드가 필요한 경우 에이전트의 응답 메시지를 기다릴 수 있습니다.
* 에이전트가 비동기 이벤트들을 프론트엔드로 다시 보내야 하는 경우, `recv()` 메서드를 추가하여 프론트엔드 파이프에서 메시지를 기다립니다.
* 프론트엔드 파이프 소켓 핸들을 노출하여 추가 폴링 루프에 통합할 수 있습니다. 그렇지 않으면 모든 `recv()` 메서드가 응용프로그램을 차단합니다.

;The clone class has the same structure as the flcliapi class from Chapter 4 - Reliable Request-Reply Patterns and adds the logic from the last model of the Clone client. Without ØMQ, this kind of multithreaded API design would be weeks of really hard work. With ØMQ, it was a day or two of work.

`clone` 클래스는 "4장 - 신뢰할 수 있는 요청-응답 패턴"의 프리랜서 패턴의 `flcliapi` 클래스와 동일한 구조를 가지며 복제 클라이언트의 마지막 모델(모델 5(임시값 + TTL))에서 고가용성 기능을 추가합니다. ØMQ가 없으면 이런 종류의 멀티스레드 API 설계는 몇 주동안 정말 힘든 작업이 될 것입니다. ØMQ에서는 하루나 이틀 정도의 작업이었습니다.

;The actual API methods for the clone class are quite simple:

`clone` 클래스의 실제 API 메서드는 매우 간단합니다.
```cpp
//  Create a new clone class instance
clone_t *
    clone_new (void);

//  Destroy a clone class instance
void
    clone_destroy (clone_t **self_p);

//  Define the subtree, if any, for this clone class
void
    clone_subtree (clone_t *self, char *subtree);

//  Connect the clone class to one server
void
    clone_connect (clone_t *self, char *address, char *service);

//  Set a value in the shared hashmap
void
    clone_set (clone_t *self, char *key, char *value, int ttl);

//  Get a value from the shared hashmap
char *
    clone_get (clone_t *self, char *key);
```

;So here is Model Six of the clone client, which has now become just a thin shell using the clone class:

복제 클라이언트의 모델 6 코드가 있으며 `clone` 클래스를 사용하여 얇은 껍질에 불과하게 되었습니다.

clonecli6.c : Clone client, Model Six in C
```cpp
//  Clone client Model Six

//  Lets us build this source without creating a library
#include "clone.c"
#define SUBTREE "/client/"

int main (void)
{
    //  Create distributed hash instance
    clone_t *clone = clone_new ();

    //  Specify configuration
    clone_subtree (clone, SUBTREE);
    clone_connect (clone, "tcp://localhost", "5556");
    clone_connect (clone, "tcp://localhost", "5566");

    //  Set random tuples into the distributed hash
    while (!zctx_interrupted) {
        //  Set random value, check it was stored
        char key [255];
        char value [10];
        sprintf (key, "%s%d", SUBTREE, randof (10000));
        sprintf (value, "%d", randof (1000000));
        clone_set (clone, key, value, randof (30));
        sleep (1);
    }
    clone_destroy (&clone);
    return 0;
}
```

;Note the connect method, which specifies one server endpoint. Under the hood, we're in fact talking to three ports. However, as the CHP protocol says, the three ports are on consecutive port numbers:

하나의 서버 단말을 지정하는 연결 방법에 유의하십시오. 내부적으로 우리는 실제로 3개의 포트들과 통신을 필요하며, CHP 통신규약에서 알 수 있듯이 3개의 포트는 연속 포트 번호입니다.

;* The server state router (ROUTER) is at port P.
;* The server updates publisher (PUB) is at port P + 1.
;* The server updates subscriber (SUB) is at port P + 2.

* 서버 상태(snapshot) 라우터(ROUTER 소켓)는 포트 번호 P.
* 서버 변경정보 발행자(PUB)는 포트 번호 P + 1.
* 서버 변경정보 구독자(SUB)는 포트 번호 P + 2.

;So we can fold the three connections into one logical operation (which we implement as three separate ØMQ connect calls).

따라서 3개의 연결을 하나의 논리적 기능(이는 3개의 개별 ØMQ 연결 호출로 구현)으로 수행할 수 있습니다.

;Let's end with the source code for the clone stack. This is a complex piece of code, but easier to understand when you break it into the frontend object class and the backend agent. The frontend sends string commands ("SUBTREE", "CONNECT", "SET", "GET") to the agent, which handles these commands as well as talking to the server(s). Here is the agent's logic:

복제 스택에 대한 소스 코드로 마무리하겠습니다. 이것은 복잡한 코드이지만 프론트엔드 객체 클래스와 백엔드 에이전트로 분리하면 이해하기 더 쉽습니다. 
프론트엔드는 문자열 명령들("SUBTREE", "CONNECT", "SET", "GET")을 에이전트에 전송합니다. 
에이전트는 이러한 명령들을 처리하고 서버와 통신합니다. 에이전트의 처리 로직은 다음과 같습니다.

;1. Start up by getting a snapshot from the first server
;2. When we get a snapshot switch to reading from the subscriber socket.
;3. If we don't get a snapshot then fail over to the second server.
;4. Poll on the pipe and the subscriber socket.
;5. If we got input on the pipe, handle the control message from the frontend object.
;6. If we got input on the subscriber, store or apply the update.
;7. If we didn't get anything from the server within a certain time, fail over.
;8. Repeat until the process is interrupted by Ctrl-C.

1. 첫 번째 서버에서 스냅샷을 가져와서 시작
2. 구독자 소켓에서 읽기로 전환하여 서버로부터 스냅샷을 받을 때
3. 서버로부터 스냅샷을 받지 못하면 두 번째 서버로 장애조치합니다.
4. 파이프(pipe)와 구독자(SUB) 소켓에서 폴링합니다.
5. 파이프(pipe)에 입력이 있으면 프론트엔드 개체에서 전달된 제어 메시지("SUBTREE", "CONNECT", "SET", "GET")를 처리합니다.
6. 구독자(SUB)에 대한 입력이 있으면 변경정보를 저장하거나 적용하십시오.
7. 일정 시간 내에 서버에서 아무것도 받지 못한다면 장애조치합니다.
8. Ctrl-C로 프로세스가 중단될 때까지 반복합니다.

; And here is the actual clone class implementation:

아래는 실제 `clone` 클래스가 구현된 코드입니다.

clone.c: Clone class in C
```cpp
//  clone class - Clone client API stack (multithreaded)

#include "clone.h"
//  If no server replies within this time, abandon request
#define GLOBAL_TIMEOUT  4000    //  msecs

//  =====================================================================
//  Synchronous part, works in our application thread

//  Structure of our class

struct _clone_t {
    zctx_t *ctx;                //  Our context wrapper
    void *pipe;                 //  Pipe through to clone agent
};

//  This is the thread that handles our real clone class
static void clone_agent (void *args, zctx_t *ctx, void *pipe);

//  .split constructor and destructor
//  Here are the constructor and destructor for the clone class. Note that
//  we create a context specifically for the pipe that connects our
//  frontend to the backend agent:

clone_t *
clone_new (void)
{
    clone_t
        *self;

    self = (clone_t *) zmalloc (sizeof (clone_t));
    self->ctx = zctx_new ();
    self->pipe = zthread_fork (self->ctx, clone_agent, NULL);
    return self;
}

void
clone_destroy (clone_t **self_p)
{
    assert (self_p);
    if (*self_p) {
        clone_t *self = *self_p;
        zctx_destroy (&self->ctx);
        free (self);
        *self_p = NULL;
    }
}

//  .split subtree method
//  Specify subtree for snapshot and updates, which we must do before
//  connecting to a server as the subtree specification is sent as the
//  first command to the server. Sends a [SUBTREE][subtree] command to
//  the agent:

void clone_subtree (clone_t *self, char *subtree)
{
    assert (self);
    zmsg_t *msg = zmsg_new ();
    zmsg_addstr (msg, "SUBTREE");
    zmsg_addstr (msg, subtree);
    zmsg_send (&msg, self->pipe);
}

//  .split connect method
//  Connect to a new server endpoint. We can connect to at most two
//  servers. Sends [CONNECT][endpoint][service] to the agent:

void
clone_connect (clone_t *self, char *address, char *service)
{
    assert (self);
    zmsg_t *msg = zmsg_new ();
    zmsg_addstr (msg, "CONNECT");
    zmsg_addstr (msg, address);
    zmsg_addstr (msg, service);
    zmsg_send (&msg, self->pipe);
}

//  .split set method
//  Set a new value in the shared hashmap. Sends a [SET][key][value][ttl]
//  command through to the agent which does the actual work:

void
clone_set (clone_t *self, char *key, char *value, int ttl)
{
    char ttlstr [10];
    sprintf (ttlstr, "%d", ttl);

    assert (self);
    zmsg_t *msg = zmsg_new ();
    zmsg_addstr (msg, "SET");
    zmsg_addstr (msg, key);
    zmsg_addstr (msg, value);
    zmsg_addstr (msg, ttlstr);
    zmsg_send (&msg, self->pipe);
}

//  .split get method
//  Look up value in distributed hash table. Sends [GET][key] to the agent and
//  waits for a value response. If there is no value available, will eventually
//  return NULL:

char *
clone_get (clone_t *self, char *key)
{
    assert (self);
    assert (key);
    zmsg_t *msg = zmsg_new ();
    zmsg_addstr (msg, "GET");
    zmsg_addstr (msg, key);
    zmsg_send (&msg, self->pipe);

    zmsg_t *reply = zmsg_recv (self->pipe);
    if (reply) {
        char *value = zmsg_popstr (reply);
        zmsg_destroy (&reply);
        return value;
    }
    return NULL;
}

//  .split working with servers
//  The backend agent manages a set of servers, which we implement using
//  our simple class model:

typedef struct {
    char *address;              //  Server address
    int port;                   //  Server port
    void *snapshot;             //  Snapshot socket
    void *subscriber;           //  Incoming updates
    uint64_t expiry;            //  When server expires
    uint requests;              //  How many snapshot requests made?
} server_t;

static server_t *
server_new (zctx_t *ctx, char *address, int port, char *subtree)
{
    server_t *self = (server_t *) zmalloc (sizeof (server_t));

    zclock_log ("I: adding server %s:%d...", address, port);
    self->address = strdup (address);
    self->port = port;

    self->snapshot = zsocket_new (ctx, ZMQ_DEALER);
    zsocket_connect (self->snapshot, "%s:%d", address, port);
    self->subscriber = zsocket_new (ctx, ZMQ_SUB);
    zsocket_connect (self->subscriber, "%s:%d", address, port + 1);
    zsocket_set_subscribe (self->subscriber, subtree);
    zsocket_set_subscribe (self->subscriber, "HUGZ");
    return self;
}

static void
server_destroy (server_t **self_p)
{
    assert (self_p);
    if (*self_p) {
        server_t *self = *self_p;
        free (self->address);
        free (self);
        *self_p = NULL;
    }
}

//  .split backend agent class
//  Here is the implementation of the backend agent itself:

//  Number of servers to which we will talk to
#define SERVER_MAX      2

//  Server considered dead if silent for this long
#define SERVER_TTL      5000    //  msecs

//  States we can be in
#define STATE_INITIAL       0   //  Before asking server for state
#define STATE_SYNCING       1   //  Getting state from server
#define STATE_ACTIVE        2   //  Getting new updates from server

typedef struct {
    zctx_t *ctx;                //  Context wrapper
    void *pipe;                 //  Pipe back to application
    zhash_t *kvmap;             //  Actual key/value table
    char *subtree;              //  Subtree specification, if any
    server_t *server [SERVER_MAX];
    uint nbr_servers;           //  0 to SERVER_MAX
    uint state;                 //  Current state
    uint cur_server;            //  If active, server 0 or 1
    int64_t sequence;           //  Last kvmsg processed
    void *publisher;            //  Outgoing updates
} agent_t;

static agent_t *
agent_new (zctx_t *ctx, void *pipe)
{
    agent_t *self = (agent_t *) zmalloc (sizeof (agent_t));
    self->ctx = ctx;
    self->pipe = pipe;
    self->kvmap = zhash_new ();
    self->subtree = strdup ("");
    self->state = STATE_INITIAL;
    self->publisher = zsocket_new (self->ctx, ZMQ_PUB);
    return self;
}

static void
agent_destroy (agent_t **self_p)
{
    assert (self_p);
    if (*self_p) {
        agent_t *self = *self_p;
        int server_nbr;
        for (server_nbr = 0; server_nbr < self->nbr_servers; server_nbr++)
            server_destroy (&self->server [server_nbr]);
        zhash_destroy (&self->kvmap);
        free (self->subtree);
        free (self);
        *self_p = NULL;
    }
}

//  .split handling a control message
//  Here we handle the different control messages from the frontend;
//  SUBTREE, CONNECT, SET, and GET:

static int
agent_control_message (agent_t *self)
{
    zmsg_t *msg = zmsg_recv (self->pipe);
    char *command = zmsg_popstr (msg);
    if (command == NULL)
        return -1;      //  Interrupted

    if (streq (command, "SUBTREE")) {
        free (self->subtree);
        self->subtree = zmsg_popstr (msg);
    }
    else
    if (streq (command, "CONNECT")) {
        char *address = zmsg_popstr (msg);
        char *service = zmsg_popstr (msg);
        if (self->nbr_servers < SERVER_MAX) {
            self->server [self->nbr_servers++] = server_new (
                self->ctx, address, atoi (service), self->subtree);
            //  We broadcast updates to all known servers
            zsocket_connect (self->publisher, "%s:%d",
                address, atoi (service) + 2);
        }
        else
            zclock_log ("E: too many servers (max. %d)", SERVER_MAX);
        free (address);
        free (service);
    }
    else
    //  .split set and get commands
    //  When we set a property, we push the new key-value pair onto
    //  all our connected servers:
    if (streq (command, "SET")) {
        char *key = zmsg_popstr (msg);
        char *value = zmsg_popstr (msg);
        char *ttl = zmsg_popstr (msg);

        //  Send key-value pair on to server
        kvmsg_t *kvmsg = kvmsg_new (0);
        kvmsg_set_key  (kvmsg, key);
        kvmsg_set_uuid (kvmsg);
        kvmsg_fmt_body (kvmsg, "%s", value);
        kvmsg_set_prop (kvmsg, "ttl", ttl);
        kvmsg_send     (kvmsg, self->publisher);
        kvmsg_store    (&kvmsg, self->kvmap);
        free (key);
        free (value);
        free (ttl);
    }
    else
    if (streq (command, "GET")) {
        char *key = zmsg_popstr (msg);
        kvmsg_t *kvmsg = (kvmsg_t *) zhash_lookup (self->kvmap, key);
        byte *value = kvmsg? kvmsg_body (kvmsg): NULL;
        if (value)
            zmq_send (self->pipe, value, kvmsg_size (kvmsg), 0);
        else
            zstr_send (self->pipe, "");
        free (key);
    }
    free (command);
    zmsg_destroy (&msg);
    return 0;
}

//  .split backend agent
//  The asynchronous agent manages a server pool and handles the
//  request-reply dialog when the application asks for it:

static void
clone_agent (void *args, zctx_t *ctx, void *pipe)
{
    agent_t *self = agent_new (ctx, pipe);

    while (true) {
        zmq_pollitem_t poll_set [] = {
            { pipe, 0, ZMQ_POLLIN, 0 },
            { 0,    0, ZMQ_POLLIN, 0 }
        };
        int poll_timer = -1;
        int poll_size = 2;
        server_t *server = self->server [self->cur_server];
        switch (self->state) {
            case STATE_INITIAL:
                //  In this state we ask the server for a snapshot,
                //  if we have a server to talk to...
                if (self->nbr_servers > 0) {
                    zclock_log ("I: waiting for server at %s:%d...",
                        server->address, server->port);
                    if (server->requests < 2) {
                        zstr_sendm (server->snapshot, "ICANHAZ?");
                        zstr_send  (server->snapshot, self->subtree);
                        server->requests++;
                    }
                    server->expiry = zclock_time () + SERVER_TTL;
                    self->state = STATE_SYNCING;
                    poll_set [1].socket = server->snapshot;
                }
                else
                    poll_size = 1;
                break;
                
            case STATE_SYNCING:
                //  In this state we read from snapshot and we expect
                //  the server to respond, else we fail over.
                poll_set [1].socket = server->snapshot;
                break;
                
            case STATE_ACTIVE:
                //  In this state we read from subscriber and we expect
                //  the server to give HUGZ, else we fail over.
                poll_set [1].socket = server->subscriber;
                break;
        }
        if (server) {
            poll_timer = (server->expiry - zclock_time ())
                       * ZMQ_POLL_MSEC;
            if (poll_timer < 0)
                poll_timer = 0;
        }
        //  .split client poll loop
        //  We're ready to process incoming messages; if nothing at all
        //  comes from our server within the timeout, that means the
        //  server is dead:
        
        int rc = zmq_poll (poll_set, poll_size, poll_timer);
        if (rc == -1)
            break;              //  Context has been shut down

        if (poll_set [0].revents & ZMQ_POLLIN) {
            if (agent_control_message (self))
                break;          //  Interrupted
        }
        else
        if (poll_set [1].revents & ZMQ_POLLIN) {
            kvmsg_t *kvmsg = kvmsg_recv (poll_set [1].socket);
            if (!kvmsg)
                break;          //  Interrupted

            //  Anything from server resets its expiry time
            server->expiry = zclock_time () + SERVER_TTL;
            if (self->state == STATE_SYNCING) {
                //  Store in snapshot until we're finished
                server->requests = 0;
                if (streq (kvmsg_key (kvmsg), "KTHXBAI")) {
                    self->sequence = kvmsg_sequence (kvmsg);
                    self->state = STATE_ACTIVE;
                    zclock_log ("I: received from %s:%d snapshot=%d",
                        server->address, server->port,
                        (int) self->sequence);
                    kvmsg_destroy (&kvmsg);
                }
                else
                    kvmsg_store (&kvmsg, self->kvmap);
            }
            else
            if (self->state == STATE_ACTIVE) {
                //  Discard out-of-sequence updates, incl. HUGZ
                if (kvmsg_sequence (kvmsg) > self->sequence) {
                    self->sequence = kvmsg_sequence (kvmsg);
                    kvmsg_store (&kvmsg, self->kvmap);
                    zclock_log ("I: received from %s:%d update=%d",
                        server->address, server->port,
                        (int) self->sequence);
                }
                else
                    kvmsg_destroy (&kvmsg);
            }
        }
        else {
            //  Server has died, failover to next
            zclock_log ("I: server at %s:%d didn't give HUGZ",
                    server->address, server->port);
            self->cur_server = (self->cur_server + 1) % self->nbr_servers;
            self->state = STATE_INITIAL;
        }
    }
    agent_destroy (&self);
}
```