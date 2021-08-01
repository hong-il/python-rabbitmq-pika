<details>
<summary>목록:</summary>
  
* [Pika - Consumer편](document/consumer.md)
* [Pika - Broker편](document/broker.md)

</details>
  
**AMQP 0-9-1의 개념**
-------------
AMQP 0-9-1(Advanced Message Queuing Protocol)은 메시징 미들웨어 브로커와 통신할 수 있도록 하는 메시징 프로토콜입니다. 

**메시징 브로커의 개념**
-------------
메시징 브로커는 publisher(게시하는 응용 프로그램, 생산자라고도 함)로부터 메시지를 수신 하고 이를 consumer(이를 처리하는 응용 프로그램, 소비자라고도 함)에게 routing 합니다. 이들은 전부 네트워크 프로토콜이기 때문에 publisher, consumer 및 브로커는 각자 다른 시스템에 구현할 수 있습니다.

**AMQP 0-9-1 모델 요약**
-------------
AMQP 0-9-1 모델에서 메시지는 exchange에 publish됩니다. 그런 다음 exchange는 binding이라는 규칙을 사용하여 메시지 복사본을 queue에 배포합니다. 그런 다음 브로커는 queue에 존재하는 소비자에게 메시지를 전달하거나 소비자가 요청할 때 queue에서 메시지를 전달합니다.

네트워크는 신뢰할 수 없고 응용 프로그램은 메시지를 처리하지 못할 수 있으므로 AMQP 0-9-1 모델에는 메시지 응답(ack)이라는 개념이 있습니다 . 메시지가 consumer에게 전달되면 consumer는 자동으로 또는 응용 프로그램 개발자가 설정한 시점에 즉시 브로커에게 알립니다. 그리고 메시지 응답이 사용 중이라면 브로커는 해당 메시지(또는 메시지 그룹)에 대한 알림을 수신할 때만 queue에서 메시지를 완전히 제거합니다.

**Exchange 유형**
-------------
RabbitMQ에서 Exchange 유형은 총 4가지가 있습니다(Kafka엔 Exchange가 없음).

* Direct exchange

* Fanout exchange

* Topic exchange

* Headers exchange

현재 Repository는 Topic exchange 방식을 채택하고 있습니다.

Topic exchange는 메시지 routing key와 queue를 exchange에 동일한 방식으로 binding한 하나 이상의 queue로 메시지를 전송합니다. 즉, Topic exchange 유형은 다양한 publish/subscribe 패턴 변형을 가지는 메시지의 멀티캐스트 전송에 사용하는 것이 가장 적합합니다.

**Bindings**
-------------
Bindings는 exchange가 메시지를 queue로 전송하는 데 사용하는 규칙입니다. exchange가 메시지를 queue로 전송하도록 지시하려면 queue가 exchange에 binding되어 있어야 합니다. binding에는 일부 exchange 유형에서 사용되는 다양한 routing key 속성이 있을 수 있습니다. routing key의 목적은 exchange에 publish된 특정 메시지를 binding된 queue로 전송하도록 선택하는 것입니다.

**Consumer**
-------------
응용 프로그램에서 메시지를 소비하지 않고 queue에 적재만 하는 것은 아무런 의미가 없습니다. 따라서 AMQP 0-9-1 모델에서는 어플리케이션이 적재된 queue를 소비하기 위해 다음 두 가지 방식 중 한 가지를 채택하여 사용해야 합니다.

* Subscribe(Push API): 권장 옵션

* Polling(Pull API): 매우 비효율적이므로 가능한 한 피할 것

Subscribe(Push API)는 생성한 consumer를 특정한 queue에 연동 시켜 적재된 queue를 실시간으로 소비하게 합니다. consumer는 queue 당 두 개 이상을 등록할 수도 있고, 독점적으로 단 하나만 등록할 수도 있습니다(소비하는 동안 해당 queue에서 다른 모든 consumer 제외).

각 consumer(Subscription)는 문자열로 이루어진 Consumer tag를 갖고 있으며 그것을 사용하여 메시지 수신을 취소할 수도 있습니다.

**Message Acknowledgements(ack)**
-------------
AMQP 0-9-1 모델에서 consumer는 메시지 수신 또는 처리 여부를 확인하기 위한 내장 기능을 갖고 있습니다. 그것을 Acknowledgements(ack)라고 합니다. 응용 프로그램에서는 네트워크가 불안정하거나 어플리케이션에서 에러가 발생하는 경우가 종종 있기 때문에 AMQP 브로커가 메시지를 수신하지 못한 경우 Acknowledgements(ack)를 사용하여 메시지를 queue에 다시 배치 시켜서 다른 consumer에 바로 전송되도록 합니다.

**Connections**
-------------
AMQP 0-9-1 연동은 기본적으로 long-lived 방식입니다. AMQP 0-9-1은 신뢰성 높은 통신을 위해 TCP를 사용합니다. 연동을 위해서는 인증 과정이 필수이며 TLS를 통한 보안이 가능합니다. 응용 프로그램을 서버에 더 이상 연결할 필요가 없다면 기본 TCP 연결을 강제로 닫지 말고 AMQP 0-9-1 연결 부분을 정상적으로 닫아야 합니다.

**Channels**
-------------
클라이언트가 수행하는 모든 프로토콜 작업은 채널에서 수행됩니다. 특정 채널의 통신은 다른 채널의 통신과 완전히 분리되므로 모든 프로토콜 method 역시 고유한 채널 ID를 가지고 있으며, 브로커와 클라이언트는 이 채널 ID를 통해 해당 method가 어떤 채널에서 실행되었는지 알 수 있습니다. 채널 ID는 개발 단에서 강제로 입력해줄 수도 있지만 대부분의 경우 자동 발행을 권장합니다. 채널은 연결 종속적이기 때문에 연결이 끊어지면 채널도 자동으로 닫힙니다.

**Virtual Hosts**
-------------
웹 서버에서 사용하는 가상 호스트와 유사하며 단일 브로커가 여러 분리된 환경(user, exchange, queue 등)을 호스팅할 수 있도록 완전히 격리된 환경을 제공합니다.
