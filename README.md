# ico-server(address-server)
- ICO 진행을 위한 서버단 개발관련 발표 정리(address-server) 
- 사정상 코드는 아직 미공개합니다... 
  
## Address-Server 발표

## 1. 개요  
#### 1-1. 목적
- ICO 지원을 위한 주소생성 서버 구축.

#### 1-2. 구조 그림
- Address-Server 구조  
<img width="839" alt="1" src="https://user-images.githubusercontent.com/32521173/51990848-3cf44000-24ed-11e9-8a48-95629f73aae8.png">

  
#### 1-3. Address-Server 주요 역할
1. 주소 생성  
2. node로 들어온 코인 transaction을 읽어와서 db에 저장.

  
## 2. Process
- **주소생성** :   
      * ICO-Server : 
          * 1. 클라이언트단에서 사용자가 주소 생성을 요청한다.
          * 2. Controller에서 API명령을 처리한다.    
                  ==> 사용자의 요청(http://localhost:8080/api/addresses (post))  
          * 3. `walletService.createWallet(walletVO);` 을 수행하여, create-address 라는 Publish를 수행한다.    
       
      * Address-Server :         
          * 4. Subscribe을 하고 있던 Address-Server가 요청을 수신한다. (Subscribe)    
          * 5. rpc명령어를 통해 node로 부터 주소를 walletservice에 리턴한다.    
          * 6. 리턴받은 데이터를, 유저정보를 포함하여 주소값과 함께 DB에 Insert(wallet테이블)    
          * 7. Redis를 통해서 생성된 주소와 유저 정보를 넘긴다. (Publish)    
          * 실제 Address-Server의 역할은 여기까지!!!!!      
                
          * 8. (address server에서 response한 데이터) websocket이 client로 넘긴다.  
          * 9. client에서 해당 주소를 받아서 QR코드와 주소를 화면에 세팅한다.  
          
          
      
- **schedule 동작** :   
      * 1. `QuartzUtil`에서 createJobDetail등에 대한 component설정.      
      * 2. schedule jobs 에서 node들에 대한(btc, eth, etc,,) JobBean과 JobBeanTrigger 빈 생성.     
            - JobBeanTrigger 에서 cron Expression으로 블록 타겟 타임설정 **(BTC계열 60초, ETH계열 15초)**    
      * 3. `SchedulerHandler`에 등록한 Job들의 receiverJob들 로부터 jobBean과 jobBeanTrigger들을 빈 객체 주입하여,     
            스케줄러를 등록시키는 component 클래스. init method의 @EventListener(ApplicationReadyEvent.class)를 통해 스케줄 시작.    
      * 4. `TransactionHandler` - 스케쥴링을 수행한다. for문을 돌면서 프로세스를 비동기로 받아서 처리.       
                BTC계열의 트랜잭션 체크와 ETH계열의 트랜잭션 체크가 존재, confirmations 체크 및 업데이트 실행.  
  
  
  
## 3. 환경설정
1. Node server config :   
      * `RPCCommonCode.java`에 각 coin들에 대한 rpc 호출 명령어를 담은 enum class 생성.  
      * model 패키지에 노드별(BTC, ETC) 클래스를 생성하고, BlockChainNodeInterface를 구현함.  
      * `CommonUtil` 클래스에서 RemoteCallType 메서드는, node에서 호출된 RPC 명령에 따른 데이터 리턴.  
      
2. redis config :   
      * `redisConfig` 클래스에 설정. 채널 등록, messageListener 등의 config설정이 있음.  
      
3. schedule :   
      * `asyncconfig` - 비동기 쓰레드풀 디폴트 설정 클래스, return ThreadPoolTaskExecutor.    
                      - 비동기 처리를 위한 설정, 현재 소스에 설정되진 않았지만 나중에 TransactionHandler에서 @Synconfig로 활용할 수 있음.  
      * `autowiringspringbeanjobfactory` - Adds autowiring support to quartz jobs.  
      * `SchedulerConfig` - quatz를 사용하기 위한 config.  

    
## 4. Redis pub/sub 활용 
#### 4-1. redis란 무엇인가? 
- Redis는 Remote dictionary system의 약자로, 메모리 기반의 **key/value store** 라고 한다.
- NoSQL DBMS로 분류되며, memcached와 같은 In Memory 솔루션으로 분류되기도 한다.
- 성능은 memcached에 버금가면서 다양한 데이타 구조체를 지원함으로써 Message Queue, Shared memory, Remote Dictionary 용도로도 사용된다.


> NoSQL ?  
- NoSQL 데이터베이스는 확장 가능한 성능 및 스키마 없는 데이터 모델에 최적화된 비관계형 데이터베이스.                    
- 기존의 관계형 DBMS가 갖고있는 특성 뿐만 아니라 다른 특성들을 부가적으로 지원.
- 열 기반, 문서, 그래프, 인 메모리 키-값 스토어를 비롯한 다양한 데이터 모델을 사용.  
- NoSQL 데이터베이스는 일반적으로 클러스터 관리 및 조정을 위한 도구를 제공.  
- NoSQL 데이터베이스는 일반적으로 스키마를 강제 적용하지 않는다.                                        
- 이를 통해 NoSQL 데이터베이스는 단순 검색 및 추가작업에 있어서 매우 최적화된 키 값 저장 기법을 사용하여 응답속도나 처리효율 등에 있어서 매우 뛰어난 성능을 나타낸다.  
 - ex) MongoDB.


> memcached ?  
 분산 메모리 캐싱 시스템 , 오픈소스, 데이터베이스 부하를 줄여 동적 웹 어플리케이션의 속도 개선을 위해 사용하기도 한다.  
              - DB나 API 호출 또는 페이지 렌더링 등으로부터 받아오는 결과 데이터를 작은 단위의 key-value 형태로 메모리에 저장하는 방식.  
              - memcached는 필요량보다 많은 메모리를 가졌을 때, 시스템으로부터 메모리를 사용하고 필요로하는 메모리가 부족한 경우, 이를 더 쉽게 가져다 쓰도록 해준다.  


                   
#### 4-2. Why Redis?
## Redis Pub/Sub Model.  
<img width="864" alt="2" src="https://user-images.githubusercontent.com/32521173/51990856-3ebe0380-24ed-11e9-8e47-149b8ce9f62c.png">
  

  * 1:N 형태의 Publish/Subscribe 메세징 기능 지원.  
  * Pub/Sub System을 통해 현재 채널에 가입한 subscriber들 모두에게 특정 이벤트를 전달해야 하기 때문에 사용.
  * 현재 소스에 적용된, pub채널은 웹소켓에서 생성된 주소를 프론트 엔드로 넘길 수 있도록 하기 위한 채널이다.  
  * sub채널은 API Server가 Redis로 해당 채널로 주소를 생성하도록 하기 위한 채널이다.  
  
  
## Redis의 Message Queue 기능
- Cluster 환경을 고려한 Redis Pub/Sub 구조를 위해 도입.
- redis의 Pub/Sub System과 Message Queue는 다르다. 
- Pub/Sub은 현재 채널에 가입한 Subscribe들 모두에게 특정 이벤트를 **전달하는 것.**
- Message Queue는 보통 일반적으로 작업을 **큐잉** 하고 있다가, 요청하는 곳에 데이터를 전달하기 위해서 **보관하는 시스템.**
- 현재 서버는 클러스터로 구성이 되어 있고 이를 큐를 통해 처리하기 위해 도입하게 되었다.

## Redis Message Queue Concept
- Message Queue Concept 이미지
- 하나 이상의 consumer가 job queue(작업 대기열)에서 queue 메세지를 소비하게 된다.
<img width="852" alt="2019-01-31 12 16 37" src="https://user-images.githubusercontent.com/32521173/51991079-b2601080-24ed-11e9-999d-027ac13f7d14.png">
  
  


## Situation
- 클러스터 서버 구성일 경우, Redis를 통해 주소생성 요청을 publish하게 되면서, 해당 채널로 subscribe중인 cluster된 두개의 Address-Server에 publish가 넘어가게 된다. 
- publish 메세지가 넘어가게 되면, 2개의 address-Server는 각각 blockchain 노드에 주소 생성을 요청을 한다.
- 하나만 생성하면 되는데 2개의 주소를 생성하게 되고 이를 해결해야 한다.

## Solution
- redis로 publish할때는 단순한 메세지만 보내고,
- publish하기 전에, `redis에 Queue 메세지로 생성 정보를 담는다.`
- 2개의 서버중 하나가 Queue에 쌓인 메세지를 pop한다.
  -> pop을 하게 되면 queue에 쌓인 메세지는 사라진다.
- 1개의 서버가 메세지를 받아 처리할 때, 다른 서버는 queue에 쌓인 메세지가 없기 때문에 비지니스 로직이 중복으로 돌지 않는다.
- 주소서버에 적용된 구조  
<img width="820" alt="3" src="https://user-images.githubusercontent.com/32521173/51990861-3fef3080-24ed-11e9-90e9-697a1373481e.png">  
   
   

## Code
1. ICO-server

```
// rightpush를 수행하여 Queue 메세지로 생성정보를 담은 후, publis를 한다. 
listQueueOperations.rightPush("dc:create:address", CommonUtil.convertJsonStringFromObject(walletVO));
redisMessagePublisher.publish(CommonUtil.convertJsonStringFromObject(walletVO));
```

2. Address-Server

```
// leftpop을 수행하여 Queue에 쌓인 메세지를 pop시키고, (queue에 쌓인 메세지는 사라진다.)
String requestCreateWalletInfo = listQueueOperations.leftPop(REDIS_QUEUE_KEY);

// 메세지를 받아 처리가게 된 서버는 주소를 생성하는 비즈니스 로직을 생성하고, 
// 다른 서버는 queue에 쌓인 메세지가 없기 때문에 비즈니스 로직이 중복으로 돌지 않게 된다.
if(requestCreateWalletInfo != null ) {
  // 주소 생성 및 디비 저장하는 로직    
} else {
			LOG.info("다른 노드 서버에서 이미 pop을 시켰다.");
}

  
```


## 5. Quartz schedule 적용     
## why Quartz?
- Quartz 는 Database 기반으로 Scheduler Key 와 Trigger Key 를 통해 Schedule 을 Clustering 한다.  
- Database 를 기반으로 모든 Schedule에 대한 제어가 가능하다.
- Schedule 과 Trigger 를 가진 Batch Application 을 Grouping 할 수 있는 구조를 제공.
- 여러 Application을 가지고 있는 Server 들을 제어할 수 있으며, 그 Application 들을 Clustering 기능을 통해 적절히 분배하여 효율적으로 동작시킬 수 있다.
- 서버 장애 상황에도 대응할 수 있다. 
- ex) **clustering없는** 시스템에서 서버에 장애가 발생했을 경우,    
<img width="758" alt="4" src="https://user-images.githubusercontent.com/32521173/51990863-4087c700-24ed-11e9-9e59-62c0747d6f04.png">
    
- ex) **clustering을 적용하여**  시스템에서 서버에 장애가 발생했을 경우
- Clustering을 통해, 같은 Group군 안에 있는 정상 서버의 application들이 나머지 job들을 적절히 분배하여 실행할 수 있으므로, 효율적인 batch수행이 가능하다.  
<img width="838" alt="5" src="https://user-images.githubusercontent.com/32521173/51990866-41b8f400-24ed-11e9-9a66-f5a5a533794b.png">


  
            
## 6. 테스트
  * etc가 가장 빠르기 때문에, redis desktop manager로 주소를 생성한뒤, 받은 주소값으로 보낸 코인이 db에 들어가는걸 테스트 한다.
  * test로 만든 ico-server의 REST API 호출(post)을 통해 요청을 전달 후, db에 wallet정보가 저장되는것을 볼 수 있다.


#### 출처
- [Building Scalable, Distributed Job Queues with Redis and Ruby](https://www.slideshare.net/francescolaurita/italians-coderjune13redisqueue)
- [Quartz + Spring Batch 조합하기](https://blog.kingbbode.com/posts/spring-batch-quartz)
- [Redis Pub/Sub 시스템은 일반적인 Message Queue와 다르다.](https://charsyam.wordpress.com/2013/03/12/%EC%9E%85-%EA%B0%9C%EB%B0%9C-redis-pubsub-%EC%8B%9C%EC%8A%A4%ED%85%9C%EC%9D%80-%EC%9D%BC%EB%B0%98%EC%A0%81%EC%9D%B8-message-queue%EC%99%80-%EB%8B%A4%EB%A5%B4%EB%8B%A4/)
- http://bcho.tistory.com/654 [조대협의 블로그]
- http://jwprogramming.tistory.com/70 [개발자를 꿈꾸는 프로그래머]


