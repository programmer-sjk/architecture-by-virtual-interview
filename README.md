# 가상 면접 사례로 배우는 대규모 시스템 설계 기초

## 1장. 사용자 수에 따른 규모 확장성

### 어떤 데이터베이스를 사용할 것인가?
- 전통적인 RDBMS나 비 관계형 데이터베이스를 선택할 수 있다.
- 아래의 경우 비 관계형 데이터베이스가 바람직한 선택일 수 있다.
  - 아주 낮은 응답 지연시간이 요구됨
  - 다루는 데이터가 비정형이라 관계형 데이터가 아님
  - 아주 많은 양의 데이터를 저장할 필요가 있음

### 수직적 규모 확장 vs 수평적 규모 확장
- 더 좋은 하드웨어 (Cpu, RAM)를 사용하는 스케일 업 방식과 서버를 늘리는 스케일 아웃 방식이 있다.
- 서버로 들어오는 트래픽 양이 적을 때는 수직적 확장이 좋은 방법이며 단순하다는 장점이 있다.
- 하지만 자동복구(failover)나 다중화 방안은 제시되지 않는다. 서버에 장애가 발생하면 웹 사이트는 중단된다.
- 따라서 대규모 애플리케이션을 지원하는 데는 수평적 규모 확장이 적절하다.
- 앞의 설계에서 사용자는 웹 서버에 바로 연결된다. 웹 서버가 다운되면 사용자는 웹 사이트에 접속할 수 없다.
또한 너무 많은 사용자가 접속할 겨우 응답속도가 느려지거나 서버 접속이 불가능 할 수 있다. 이때에는 로드 밸런서를 도입하는게 좋다.

### 로드 밸런서
- 로드 밸런서는 웹 서버들에게 트래픽을 적절히 분산하는 역할을 하며 사용자는 로드밸런서의 공개 IP주소로 접속한다.
- 로드 밸런서에 두 대의 서버가 연결되어 있다고 가정할 떄, 한 대의 서버가 죽어도 다른 서버가 동작하므로 가용성이 향상된다.
- 웹 서버로 유입되는 트래픽이 증가한다면 서버를 더 추가하면 된다. 로드밸런서가 자동적으로 트래픽을 분산할 것이다.

### 데이터베이스 다중화
- master <-> slave로 다중화 한다. 쓰기 작업은 master DB에만 수행하며 slave DB는 읽기 작업을 수행한다.
- 보통 읽기 작업이 쓰기 작어보다 많으므로 한 대의 master DB에 여러 slave DB를 두는게 일반적이다.
- 더 나은 성능: 쓰기 작업은 주 서버로 전달되는 반면 읽기 작업은 부 서버로 분산되기 때문에 성능이 좋아진다.
- 가용성: 자연재해나 DB 서버 중 하나가 장애가 나더라도 다중화 되어 있다면 계속 서비스 할 수 있게 된다.

#### 문제상황 
- Slave DB가 한 대 뿐인데 장애가 발생한다면 읽기 연산은 master DB로 전달될 것이다. 동시에 새로운 Slave DB가 구동되어 장애서버를
대체할 것이다. Slave DB가 여러대라면 읽기 연산은 분산될 것이며 새로운 DB가 장애 서버를 대체할 것이다.
- Master DB가 다운되면 한 대의 Slave DB만 있는 경우, 그 DB가 새로운 master DB가 되고 새로운 Slave DB가 추가될 것이다.

### 캐시
- 캐시는 자주 사용되는 데이터를 메모리에 두고 요청이 빨리 처리되도록 돕는 저장소이다.
- 캐쉬 사용시 주의할 점
  - 캐시는 어떤 상황에 바람직할까? -> 데이터 갱신은 자주 발생하지 않지만 빈번하게 참조가 발생한다면 고려해볼만 하다.
  - 어떤 데이터를 두어야 할까? -> 캐시 서버가 재 시작되면 캐시 내의 모든 데이터가 사라지므로 영속적으로 보관할 데이터는 DB에 두어야 한다.
  - 어떻게 만료되는가? -> 만료기간이 너무 짧다면 DB를 많이 읽게 될 것이고 너무 길다면 원본가 차이가 날 가능성이 높아진다.
  - 장애는 어떻게 대처할 것인가? -> 캐시 서버를 한대만 둘 경우 장애 지점이 될 수 있으므로 여러 지역에 캐시 서버를 분산해야 한다.
  - 캐시 메모리는 얼마나 잡을것인가? -> 캐시 메모리가 너무 작으면 데이터가 캐시에서 자주 밀려나버려 캐시의 성능이 떨어진다. 이를 막을 한 가지
  방법은 메모리를 과할당 하는 것이다. 이렇게 하면 캐시에 보관할 데이터가 갑자기 늘어났을 때 생길 문제를 방지할 수 있다.
  - 데이터 방출 정책은 무엇인가? -> 캐시가 꽉 찬 상태에서 추가로 데이터를 넣어야 하는 경우, 기존 데이터를 내보내야 한다. 이를 방출 정책이라
  하는데, 가장 널리 쓰이는 것은 LRU(마지막으로 사용된 시점이 오래된 데이터를 내보내는 정책)

### 컨텐츠 전송 네트워크(CDN)
- CDN은 정적 컨텐츠를 전송하는데 쓰이는 분산된 네트워크이다. 이미지, 비디오, CSS, JavaScript 파일 등을 캐시할 수 있다.
- 사용자가 웹 사이트를 방문하면 그 사용자에게 가장 가까운 CDN 서버가 정적 컨텐츠를 전달하게 된다.
- 예를 들어 사용자 A가 image를 요청 시, CDN 서버는 그 이미지가 없으면 서버에게 요청해 이미지를 저장 후 사용자에게 전달하게 된다.
이후 사용자 B가 image 요청 시, CDN 서버가 만료되지 않은 이미지를 응답하게 된다.
- CDN 사용시 고려해야 할 사항
  - 비용: CDN 데이터 전송 양에 따라 요금을 내게 되므로 자주 사용하지 않은 컨텐츠는 CDN에서 뺴자
  - 적절한 만료기한: 만료 기한이 너무 길면 컨텐츠의 신선도가 떨어질 것이고, 너무 짧으면 원본 서버에 너무 자주 접속하게 된다.
  - 장애 대처: CDN 자체가 죽었을 때를 고려해야 한다. 가령 CDN이 일정 기간 응답하지 않으면 원본 서버로부터 직접 컨텐츠를 가져오도록 구성한다.
