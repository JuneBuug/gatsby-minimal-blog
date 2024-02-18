---
title   : '트위터가 볼 수 있는 트윗수를 제한한다고? rate-limit' 
slug  : '/rate-limit-algorithm'
excerpt : 
date    : 2023-07-02 21:30:01 +0900
updated : 2023-07-02 21:30:28
banner: ./thumb.png
tags    : 
---

## 서론 

오늘 트위터를 들어가보니 꽤 난리가 나있었다. 일론 머스크가 트위터의 `verified account` (즉, 파란 체크 - 어느 시점부터 돈내고 받을 수 있게 바뀌었다.) 는 하루에 6000개 의 포스트를 볼 수 있도록 제한하고, 인증받지 않은 계정은 600개, 그리고 신규 가입 + 미인증 계정은 300 개만 보도록 한다고 한다.
 
![elon](./elon.png)

트위터의 포스트를 많이 읽는 편은 아니어서 모르겠지만 슥 다른 사람들의 이야기를 보니 실제로 API 제한으로 뜨는 모양이다. ㅠㅠ 'bytebytego' 의 운영자인 Alex Xu는 굉장한 속도로 지난 6월 해당 블로그의 주제였던 [rate limit](https://blog.bytebytego.com/p/rate-limiting-fundamentals) 를 홍보했고, 그것에 감동받아서 (ㅋㅋ) rate limit 알고리즘을 공부해보기로한다. 참고로 해당 주제는 동일인이 저자인 '가상 면접 사례로 배우는 대규모 시스템 설계 기초, 4장 '처리율 제한 장치의 설계' 와 내용이 거의 동일하다. 

![alex](./alex.png)

## rate limit 이 뭔데? 

처음 이름을 들으면 rate limit 이 생소하기 마련이다. rate limit 은 보통 '처리율 제한 장치' 정도로 번역되며, 클라이언트나 서비스가 보내는 요청의 처리율 (rate) 를 제한한다는 점에서 이런 이름이 붙었다. HTTP 에서는 특정 기간 내에 전송되는 클라이언트의 요청 횟수를 제한한다. API 요청 횟수가 제한 장치에 정의된 임계치를 넘어사면 추가로 받은 요청은 처리를 중단해버린다. 

- 사용자는 초당 2회 이상 새 글을 올릴 수 없다. 
- 같은 IP 주소로 하루에 10개 이상 계정을 생성할 수 없다. 
- 유저의 인기도를 하루에 한번밖에 올릴 수 없다.

### 장점은요? 

이런 제한 장치를 두면 좋은 점이 있다. 

**DoS 공격 방지**  | DoS(Denial Of Service) 공격 / DDoS(Distributed Denial Of Service) 에 의한 자원 고갈을 방지할 수 있다.  DoS 공격류는 특정 포인트를 집중 공격해서 해당 서비스를 사용 불가하게 만드는 것을 목적으로 하는데, 만약 제한 장치 (rate limit) 가 없다면 딱 공격하기 좋다. 대형 IT 기업이 공개한 많은 API 는 요청 포인트와 방법까지 모두 공개되어있으므로. 예를 들어 원래의 트위터는 3시간 동안 300개의 트윗만 올릴 수 있도록 제한하고 있다. DoS 공격을 방어하지 못하면, java 로 구성된 서버의 경우 예상보다 과도한 요청때문에 여력을 확보하려고 GC 를 하고, GC 동안 생기는 stop-the-world 로 계속 서비스 응답은 지연되는 악순환을확인할 수 있다. 🥲

**비용 절감** | 요청을 제한하면 상대적으로 서버를 많이 두지 않아도 되고, 우선순위가 높은 API 에 자원을 더 많이 할당할 수 있다. 또한 비용면에서, 내 API 가 또다른 외부 API 를 호출하는데 비용이 발생하고 있다고 생각해보자. 그러면 외부의 공격자가 과도하게 많이 호출하는 경우 API 호출 당 비용이 급증한다. 따라서 횟수를 제한하는 게 비용 절감과 직결된다. 


**서버 과부하 방지** | 봇에서 오는 트래픽이나 어뷰징 유저로 유발된 트래픽을 걸러내는데 rate-limit 를 활용할 수 있다. 


### 가깝고 친절한 예  : nginx 에서 막아주는걸 보자 

우리의 친구 [nginx 에서는 rate limit 기능을 지원](https://www.nginx.com/blog/rate-limiting-nginx/) 하고 있다. nginx에서는 이후에 설명할 `leaky bucket` 알고리즘을 사용해서 rate limit 을 지원하고, rate limit 할 만한 일이 있을 때 로그를 남길 수 있는 기능 또한 지원한다.

nginx에서 rate limit를 설정하기 위한 명령어는 `limit_req_zone` 그리고 `limit_req` 가 있다. 

```nginx.conf
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;
 
server {
    location /login/ {
        limit_req zone=mylimit;
        
        proxy_pass http://my_upstream;
    }
}
```


limit_req_zone 는 rate limit을 위한 파라미터를 정의하고, limit_req 는 rate limit 옵션 자체를 활성화한다. (위예시에서 line 5)

limit_req_zone 의 구성을 보자. 
- Key - 예시에서 `$binary_remote_addr` 로 표현된것. 처리율 제한이 적용될 성질을 의미한다. 예를 들어 위에서는 client 의 바이너리 IP 주소에 대해서 적용된다는 뜻이다. 이는 즉, IP 당 제한을 할거라는 의미로, IP 당 제한율이 적용된다는 뜻이다. 

- Zone - 각 IP 주소의 상태, 그리고 얼마나 자주 해당 url 로 접근했는지를 저장하는 공용 메모리 (shared memory) 존을 저장한다. 공용 메모리에 저장하므로, nginx worker 프로세스 끼리는 해당 값을 공유하고 있다. zone은 두 부분이 있는데, zone=`내가 정의하고 싶은 zone 이름` 을 정의하고, `:` 다음에는 해당 메모리의 크기를 정의한다.  만육천개의 IP 를 저장하는 데 1MB 정도 사용한다고하니, 위 정의에서는 십육만개정도의 IP 정보를 저장할 수 있다.  만약 저장공간이 꽉찼는데 새로운 정보를 저장해야하면, 가장 오래된 것부터 삭제한다.  


- Rate - 우리가 가장 관심있는 주제, 즉 처리율을 설정한다. 예시에서는 처리율이 초당 10 회를 넘지 못하게 되어있다. NGINX 는 ms 단위로 요청을 추적하므로, 이 비율은 100ms 당 1개의 요청과 일맥상통한다. `burst` 를 허용하는 기능도 활성화할 수 있는데, 이는 이전에 통과한 요청 이후로 100ms 가 안지났는데 도착한 요청이 있으면 해당 요청은 거부하는 것을 의미한다. 

이제 그럼 실제로 궁금한 건 알고리즘에 관한 이야기다. 위에서 논의한 leaky bucket 을 살펴보자.


## 다양한 rate limit 방식 

### leaky bucket 

leaky bucket은 한국식으로 옮기면 새는 바가지 (?) 정도 될지도 모르겠다. '누출 버킷 알고리즘' 으로도 번역된다. 
![](https://www.nginx.com/wp-content/uploads/2017/06/leaky-bucket-featured-image-640x385.jpg)

nginx는 위의 이미지로 leaky bucket을 설명하고 있다. 위에서 물을 부으면, 물이 떨어지기 마련인데, 양동이 덕분에 떨어지는 물의 양은 줄어든다. 만약 물이 부어지는 속도가 새는 속도를 넘어서면, 양동이는 넘치게된다. 이를 요청 처리 관점에서 보면 물은 클라이언트의 요청을 의미하고, 양동이는 FIFO 에 따라 기다리고 있는 요청을 의미한다. 새는 물은 서버에서 처리해서 응답을 받은 요청들을, 그리고 넘치는 물은 리젝되어버린 요청들을 의미한다. 

실제로 구현은 이렇다. 

- FIFO 큐로 구현한다. 요청이 도착하면 큐가 가득 차 있는지 본다. 
	- 빈 자리가 있으면 큐에 요청을 추가한다. 
	- 가득 차 있으면 새 요청은 버린다. 
	- 지정된 시간마다 큐에서 요청을 꺼내서 처리한다.


**장점** | 큐의 크기가 제한되어 메모리 사용량 측면에서 효율적이다. 그리고 고정된 처리율을 갖고 있어 안정된 출력이 필요한 경우 적합하다. 

**단점** | 단시간에 트래픽이 몰리는 경우 큐에는 오래된 요청이 쌓이고, 그 요청을 제때 처리하지 못하면 새로운 요청들은 그냥 버려지게 된다. 두 개 인자(버킷 크기와 처리율)를 올바르게 튜닝하기도 어렵다. 

nginx 에서 버킷 크기는 공용메모리의 크기가 될 것이고, 처리율은  `rate=10r/s` 부분이 될 것이다. 그런데 새로운 ip의 요청이 들어오면 오래된 걸 버린다고 했으니, 위 단점을 상쇄해보려는 시도처럼 보인다. 

### token bucket 

토큰 버킷은 간단하고 보편적인 방식의 처리율 제한 알고리즘이다. 아마존과 스트라이프가 API 요청을 통제하기 위해서 이를 사용한다고 한다. 

- 토큰 버킷은 지정된 용량을 갖는 컨테이너다. 이 버킷에는 사전 설정된 양의 토큰이 주기적으로 채워진다. 
	- 단, 토큰이 꽉차면 더이상 추가되지 않는다. 추가로 공급된 토큰은 버려진다.
	- 토큰 공급기(refiller) 가 따로 존재한다.

- 각 요청은 처리될 때마다 하나의 토큰을 사용한다. 
	- 요청이 도착하면 버킷에 충분한 토큰이 있는지 검사한다. 
		- 있으면, 토큰 하나를 꺼낸 후 요청을 시스템에 전달한다. 
		- 없는 경우, 해당 요청은 버려진다. 


이 토큰 버킷 알고리즘 역시 두개 인자를 갖는다. 
- 버킷 크기: 버킷에 담을 수 있는 토큰 갯수 
- 토큰 공급률 : 초당 몇초의 토큰이 공급되는가? 

버킷 자체가 몇개여야 하는지는 서비스에 따라 다르다. 
- 보통 API 엔드포인트마다 별도 버킷을 둔다. 하루에 한번 포스팅을 하고, 좋아요는 하루에 다섯번이라면, 두개의 버킷을 두어야한다. 
- IP 주소별로 해야한다면 IP 주소마다 버킷을 할당해야한다. 
- 시스템의 전체 처리율을 초당 만개로 하고 싶다면, 모든 요청이 하나의 버킷을 공유해야한다. 

**장점** | 
- 구현이 쉽다. 
- 메모리 사용 측면에서 효율적. 
- 짧은 트래픽도 처리 가능하다. 

**단점** |
- 이 알고리즘의 두 개 인자를 튜닝하는 것도 까다로운 일이다.


### 고정 윈도우 카운터 알고리즘

고정 윈도우 카운터 알고리즘  (fixed window counter) 은 다음과 같이 동작한다. 

- 타임라인을 고정된 간격의 윈도우로 나누고, 각 윈도우마다 카운터를 붙인다. 
- 요청이 접수될 때마다 이 카운터의 값은 1씩 증가한다. 
- 이 카운터의 값이 사전 설정된 임계치에 도달하면, 새로운 요청은 새로운 윈도우가 열릴 때 (시간이 지나서) 까지 버려진다. 

![fixed](./fixed.png)
하지만 고정 윈도우 카운터 알고리즘은  단점이 있다. 윈도우를 나누다보니 경계에서 순간적으로 트래픽이 집중되면 할당된 양보다 많은 요청이 처리되는 것처럼 될 수 있다. 1분 단위 윈도우에서 분당 3개 요청만 허용하는 시스템이라고 하자. 아래 와 같은 식으로 요청이 들어오는 경우,

![fixed-disadvantags](./fixed-dis.png)
00:30 초 부터 1m 30s 까지의 1분에는 3개가 아니라 6개가 처리된다. 즉 처리한도의 두배가 순간적으로 가능하다. 

**장점** | 
- 메모리 효율이 좋다. 
- 이해가기 쉽다. 
- 특정 트래픽 패턴을 처리하기 좋다. 

**단점** | 
- 윈도우 경계에서 트래픽이 몰리면, 윈도우의 특성상 처리한도보다 많은 값을 처리하게 된다. 


### 이동 윈도우 로깅 알고리즘 

위 고정 윈도우 알고리즘의 문제를 상쇄하기 위해서, 이동 윈도우 로깅 알고리즘(sliding window logging)도 있다. 동작원리는 다음과 같다. 

- 이 알고리즘은 요청의 타임스탬프를 추적한다. 타임 스탬프 데이터는 보통 redis 의 sorted set 같은 캐시에 보관한다. 
- 새 요청이 오면 만료된 타임스탬프는 제거한다. 만료된 타임스탬프는, 현재 윈도우의 시작 시점보다 이전인 값을 의미한다. 
- 새 요청의 타임스탬프를 로그에 추가한다. 
- 로그의 크기가 허용치보다 같거나 작으면 시스템에 전달한다.
	- 그렇지 않으면 처리를 거부한다. 

로그가 타임스탬프를 기록하는 것이므로, 어떻게 보면 '저장소' 라는 점에서 leaky bucket 처럼 보일 수도 있을 것 같다. 

분당 2회 처리하는 이동 윈도우 알고리즘을 보자. 
![moving](./moving.png)

**장점** | 
- 어느 순간의 윈도우를 관찰해도, 처리율 한도를 넘지 않는다. 

**단점** | 
- 메모리 소비가 많다. 거부된 요청의 타임스탬프도 보관하기 때문이다. 


## 마치며 

이동 윈도우 카운터 알고리즘과 처리율 제한 장치에 대한 고려사항은 여기에서 다루지 않았다. 처리율 제한 장치를 실제로 미들웨어로 개발하는 경우도 있겠지만, 여러 제약이 있는 경우 위에서 다뤄진 것처럼 웹서버의 기능을 사용하는 접근법이 있을 것 같다. 

## 참고 
- https://blog.bytebytego.com/p/rate-limiting-fundamentals

- 가상 면접 사례로 배우는 대규모 시스템 설계 기초, 4장 '처리율 제한 장치의 설계'

- https://www.nginx.com/blog/rate-limiting-nginx/