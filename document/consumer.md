Pika는 Python 3.4+ 전용의 RabbitMQ(AMQP 0-9-1) 클라이언트 라이브러리입니다.

AMQP는 클라이언트가 서버에 요청을 보낼 수 있고 서버 또한 클라이언트에 요청을 보낼 수 있는 양방향 RPC 프로토콜이기 때문에, 각 연결 어댑터에서 Pika를 활용한다면 IO 반복문을 더욱 수월하게 구현할 수 있습니다.

# 1. Installing Pika

Pika는 PyPI를 통해 다운로드할 수 있으며 pip를 사용하여 설치할 수 있습니다.
```
pip install pika
```
# 2. Pika adapters
```
Pika에서 제공하는 커넥션 어댑터 목록은 다음과 같습니다. 여기서는 Deadlock 방지를 위해 BlockingConnection을 다룹니다.

pika.adapters.asyncio_connection.AsyncioConnection - Python 3 전용의 비동기 연결 어댑터

pika.BlockingConnection - 예상 응답이 반환 될 때까지 IO를 차단하는 동기 연결 어댑터

pika.SelectConnection - Third-party 종속성이 없는 비동기 연결 어댑터

pika.adapters.gevent_connection.GeventConnection - Gevent 전용 비동기 연결 어댑터

pika.adapters.tornado_connection.TornadoConnection - Tornado 전용 비동기 연결 어댑터

pika.adapters.twisted_connection.TwistedProtocolConnection - Twisted 전용 비동기 연결 어댑터
```
# 3. Pika Connection

RabbitMQ에서 지원하는 모든 프로토콜은 TCP 기반이며 효율성을 위해 long-lived connection을 맺습니다.

Client가 RabbitMQ에 연결하고 성공적으로 인증한 뒤에야 Message를 Publish하거나 Consume 할 수 있습니다.

따라서 위 정보들을 바탕으로 Pika를 통해 RabbitMQ와의 Connection부터 맺어보겠습니다.
```
import pika

# {} 안에는 실제 RabbitMQ 서버 정보를 입력해야 합니다.
rabbitmq_url = "amqp://{id}:{password}@{ip}:{port}/{vhost}"
connection = pika.BlockingConnection(
            parameters=pika.URLParameters(f"{rabbitmq_url}")
        )
```
# 4. Pika Channel

Client가 수행하는 모든 프로토콜 작업은 Channel이 수행합니다. 일반적으로 Multiple threads/process 당 하나의 Channel 위에서 동작합니다.

다음은 Channel에서 Consumer를 작동 시켜보겠습니다.
```
try:
    channel = connection.channel()
    # Ack를 받지 못한 Message가 한 개라도 있으면 이 Channel에 더 이상 Message를 할당하지 않음
    channel.basic_qos(prefetch_count=1)
    # queue : Message를 할당 받을 대상 queue
    # on_message_callback : Message(json 타입)를 return 받을 method 이름, 아래에선 body로 return 받음
    #                       {method명}(self, channel, method, properties, body) 형태로 작성
    # auto_ack : True일 시, Message를 받으면 자동으로 전송 성공을 return, 이 경우 속도는 높아지지만 안정성이 떨어짐
    channel.basic_consume(queue={대상 queue}, on_message_callback={method명}, auto_ack=False)
    # 해당 Channel이 지금부터 Message를 할당 받습니다.
    channel.start_consuming()
except KeyboardInterrupt:
    # 해당 Channel이 지금부터 Message를 할당받지 않습니다.
    channel.stop_consuming()
except Exception:
    channel.stop_consuming()
    # Python 프로그램 종료 함수를 실행합니다.
    sys.exit(1)
```
각 Connection은 한정적인 Channel resource를 가지고 있으므로 사용하지 않는 Channel에 대해선 꼭 Consuming을 중지해주시기 바랍니다.

# 5. Pika Acknowledgement

Channel 설정에서 auto_ack를 False로 지정해줬기 때문에 수동으로 on_message_callback method에서 Ack를 return 해줘야만 합니다. Client는 이 Ack를 통해 Message의 처리 성공여부를 판별할 수 있습니다. 작성 위치는 
```
channel.basic_consume(queue={대상 queue}, on_message_callback={method명}, auto_ack=False)에서 지정했던 on_message_callback의 method 내부에 입력하시면 되겠습니다.

def {method명}(self, channel, method, properties, body):
    # deliver_tag : int/long로 이루어진 서버가 할당한 전송 태그(위 param에 포함된 method)
    # multiple : True로 설정하면 하나의 Method가 여러 Message를 처리, False로 설정하면 단일 Message를 처리
    # 에러가 발생한 Message를 다시 Queue에 Publish하지 않고 다음 Message를 처리하려면 아래의 Ack 방식을 채택
    channel.basic_ack(delivery_tag=method.delivery_tag, multiple=False)
```
이 밖에도 에러 Message로 하여금 다시 Queue에 Publish하는 basic_nack 함수(requeue 값을 True로 설정)와 에러가 발생할 시 Consuming을 중지 시켜버리는 basic_cancle 함수(Consumer tag 지정) 등이 있으나 여기에선 사용하지 않겠습니다.

위에 작성된 소스코드 만으로도 충분히 AMQP 0-9-1 모델의 Consumer를 작동 시킬 수 있습니다.