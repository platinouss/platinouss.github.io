---
title:  "[smallTalk] 1. WebSocket과 STOMP 이해하기"
excerpt: "최소 기능을 제공하는 STOMP 기반 메시징 서비스를 만들어 보면서, Spring에서 WebSocket과 STOMP을 활용하는 방법을 익혀보자."

categories:
  - Backend-project
  - smallTalk
tags:
  - WebSocket
  - STOMP

toc: true
toc_sticky: true

date: 2023-05-20
last_modified_at: 2023-05-20
---

## 개요
블로그에 실시간으로 문의하고 답변 받을 수 있는 서비스를 추가하면 괜찮을 것 같다고 생각하여, 이번 기회에 해당 기능을 구현해 보고자 한다.

이전에는 Polling(또는 Long Polling) 기법을 사용하여 실시간에 가깝게 구현하였지만, HTML5부터는 WebSocket을 통해 실시간 통신이 가능하다고 한다.

따라서, 실시간 채팅 웹 서비스(이하 smallTalk)를 WebSocket 기반으로 구현하고, Messaging 프로토콜은 STOMP를 사용하려 한다. (이후에 Messaging 프로토콜을 MQTT로도 구현해 보면서 전체적으로 성능 개선할 예정이다.)

이번 포스팅에서는 **WebSocket**과 **STOMP**의 기본적인 구조를 살펴보고, 실시간 채팅 웹 서비스의 최소한의 기능을 직접 구현해 보면서 Spring에서는 어떻게 구현해야 하는지 감을 잡아보고자 한다.

## WebSocket
* HTTP의 단방향 통신을 보완하기 위해서, HTML5 명세(RFC-6455)에 포함된 실시간 양방향(Full-duplex) 통신 프로토콜이다.
* HTTP 기본 포트인 80, 443 포트로 접속을 하므로 추가 방화벽을 열지 않고도 통신이 가능하며, HTTP 규격인 CORS나 인증 등의 과정이 동일하다.
* 접속(Handshaking)까지는 HTTP 프로토콜을 이용하고, 이후 통신은 WebSocket 자체 프로토콜로 WebSocket Frame을 전송한다.
* 연결을 그대로 유지하고, 클라이언트의 요청 없이도 데이터를 전송할 수 있는 프로토콜이다.
* 프로토콜 요청은 [ws://~]로 시작한다.

<center>
    <img src="/assets/images/smallTalk/1/1.png" width="90%" height="90%" alt="WebSocket Handshaking" title="WebSocket Handshaking"/>
</center>

## Message Broker

* Publisher(송신자)가 전달한 메시지를 적합한 Subscriber(수신자)에게 전달해 주어 메시지를 교환할 수 있게 한다. (Publish-Subscribe 패턴)
* 일반적으로 메시지가 적재되는 공간을 Message Queue(메시지 큐)라 하고, 메시지 그룹을 Topic(토픽)이라 부른다.
* 대표적으로 RabbitMQ, Apache Kafka, Redis 등이 있다.

<center>
    <img src="/assets/images/smallTalk/1/2.png" width="90%" height="90%" alt="Message Broker" title="Message Broker"/>
</center>

## STOMP(Simple Text Oriented Messaging Protocol)
* 텍스트 기반의 Messaging 프로토콜이다.
* 신뢰할 수 있는 양방향 스트리밍 네트워크 프로토콜(TCP, WebSocket 등) 위에서 통신하는 STOMP Frame 기반 프로토콜이다.
* Message Broker를 사용하여, Publish-Subscribe 구조로 Messaging 한다.

```
// STOMP Frame 구조
COMMAND             // SEND, SUBSCRIBE, UNSUBSCRIBE, BEGIN, COMMIT, ABORT, ACK, NACK, DISCONNECT
header1:value1
header2:value2

Body^@
```

> STOMP Frame의 자세한 구조는 [**stomp.github.io - STOMP Frames**](https://stomp.github.io/stomp-specification-1.2.html#STOMP_Frames)에서 확인할 수 있습니다.


이렇게 Spring에서 STOMP와 Message Broker를 사용한 Messaging Flow는 다음과 같다.
<center>
    <img src="/assets/images/smallTalk/1/3.png" width="100%" height="100%" alt="STOMP Message Flow with Message Broker" title="STOMP Message Flow with Message Broker"/>
</center>


## Messaging 서비스 최소 구현
이번에는 [**spring.io guide**](https://spring.io/guides/gs/messaging-stomp-websocket/)에 소개된 예제 코드를 분석해 보며, Spring에서 WebSocket과 STOMP를 어떤 식으로 활용해야 하는지 감을 익혀보자.

### GreetingController
먼저 살펴볼 부분은 Controller 부분이다.

```java
@Controller
public class GreetingController {

	@MessageMapping("/hello")
	@SendTo("/topic/greetings")
	public Greeting greeting(HelloMessage message) throws Exception {
//		Thread.sleep(1000); // simulated delay
		return new Greeting("Hello, " + HtmlUtils.htmlEscape(message.getName()) + "!");
	}

}
```

* **@MessageMapping**  
  * 메시지가 특정 경로로 전송되면 해당 메서드가 호출되는 어노테이션이다.  
  * 위의 예시에서는, 메시지가 /hello 대상으로 전달되면 greeting() 메서드가 호출된다.

* **@SendTo**  
  * 특정 Topic을 subscribe한 모든 사람에게 반환 값을 전달한다.  
  * 위의 예시에서는 /topic/greetings을 subscribe 한 모든 사람에게 Greeting 객체를 전달한다.

### WebSocketConfig
다음으로 살펴볼 부분은 WebSocketConfig 부분이다.

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

	@Override
	public void configureMessageBroker(MessageBrokerRegistry config) {
		config.enableSimpleBroker("/topic", "/queue");
		config.setApplicationDestinationPrefixes("/app");
	}

	@Override
	public void registerStompEndpoints(StompEndpointRegistry registry) {
		registry.addEndpoint("/gs-guide-websocket").withSockJS();
	}

}
```

* **@EnableWebSocketMessageBroker**  
  * Message Broker가 지원하는 WebSocket 메시지 처리를 활성화하는 어노테이션이다.

* **void configureMessageBroker(MessageBrokerRegistry config)**  
  * Message Broker를 구성하는 메서드이다.  
  * enableSimpleBroker()를 호출하여, 메모리 기반 Message Broker가 /topic과 /queue 접두사가 붙은 메시지를 처리하고, 클라이언트에게 다시 전달할 수 있도록 한다.
  * setApplicationDestinationPrefixes()를 호출하여, @MessageMapping 어노테이션에서 지정한 경로에 접두사 /app을 붙인다. 따라서 클라이언트에서 /app/hello 경로로 메시지를 보내면 GreetingController.greeting() 메서드가 처리한다.  

> 정리하면, enableSimpleBroker() 메서드에 파라미터로 넘겨주는 접두사(예시에서는 /topic 또는 /queue)가 붙은 요청은 Message Broker가 처리하고, setApplicationDestinationPrefixes() 메서드에 파라미터로 넘겨주는 접두사(예시에서는 /app)가 붙은 요청은 Controller에서 @MessageMapping 어노테이션이 붙은 메서드가 직접 처리한다고 생각하시면 됩니다.

* **void registerStompEndpoints(StompEndpointRegistry registry)**  
  * /gs-guide-websocket이라는 endpoint를 등록하여 클라이언트 단에서 WebSocket을 사용할 수 없는 경우, 대체할 수 있는 전송 방식을 사용하도록 SockJS fallback 옵션을 활성화하는 메서드이다.  
  * SockJS 클라이언트는 /gs-guide-websocket에 연결하고, 사용 가능한 최상의 전송 방식(websocket, xhr-streaming, xhr-polling 등)을 선택하여 사용한다.

이렇게 구현된 STOMP 기반 Messaging 서비스의 flow는 다음 그림과 같다.

<center>
    <img src="/assets/images/smallTalk/1/4.png" width="100%" height="100%" alt="STOMP Message Flow" title="STOMP Message Flow"/>
</center>

### app.js

마지막으로는 클라이언트 단에서 subscribe와 메시지 전송 등을 요청하는 부분이다.  

이전에 WebSocketConfig에서 설정한 접두사를 어떤 식으로 붙여 서버에 요청하는지 확인해 보자.

<center>
    <img src="/assets/images/smallTalk/1/5.png" width="100%" height="100%" alt="real-time chatting" title="real-time chatting"/>
</center>

```js
// ...

function connect() {
    var socket = new SockJS('/gs-guide-websocket');
    stompClient = Stomp.over(socket);
    stompClient.connect({}, function (frame) {
        setConnected(true);
        console.log('Connected: ' + frame);
        stompClient.subscribe('/topic/greetings', function (greeting) {
            showGreeting(JSON.parse(greeting.body).content);
        });
    });
}

function disconnect() {
    if (stompClient !== null) {
        stompClient.disconnect();
    }
    setConnected(false);
    console.log("Disconnected");
}

function sendName() {
    stompClient.send("/app/hello", {}, JSON.stringify({'name': $("#name").val()}));
}

function showGreeting(message) {
    $("#greetings").append("<tr><td>" + message + "</td></tr>");
}

// ...
```

* **connect()**  
  1. SockJS를 사용하여, SockJS 서버가 연결을 대기하는 /gs-guide-websocket에 연결을 요청한다.
  2. 연결이 성공하면, 클라이언트는 /topic/greetings 경로로 subscribe를 요청한다. (subscribe는 Message Broker에서 처리하는 것이므로 enableSimpleBroker() 메서드의 파라미터인 /topic 접두사를 붙인 경로로 지정한다.)

* **sendName()**  
  * /app/hello 경로로 메시지를 전송한다. 이때 경로에 setApplicationDestinationPrefixes() 메서드의 파라미터인 /app 접두사가 붙었으므로 GreetingController.greeting() 메서드가 메시지를 수신한다.

## 마무리
다음 포스팅에서는 spring docs를 통해, spring에서의 WebSocket, STOMP와 관련된 메서드들을 더 자세히 살펴보고자 한다.

## 참고
* [**stackoverflow - what is sec-websocket-key for?**](https://stackoverflow.com/questions/18265128/what-is-sec-websocket-key-for)
* [**dev.to - message brokers, a brief walk-through**](https://dev.to/salemzii/message-brokers-a-brief-walk-through-5h7f)
* [**stomp.github.io - STOMP Frames**](https://stomp.github.io/stomp-specification-1.2.html#STOMP_Frames)
* [**docs.spring.io - STOMP Message Flow**](https://docs.spring.io/spring-framework/reference/web/websocket/stomp/message-flow.html)
* [**spring.io guide - Messaging-STOMP-WebSocket**](https://spring.io/guides/gs/messaging-stomp-websocket/)