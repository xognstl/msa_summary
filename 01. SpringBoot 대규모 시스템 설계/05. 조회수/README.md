## 조회수

### 1. 조회수 설계 & Redis 개발 환경 세팅
___
#### 조회수 요구사항
- 조회수
- 조회수 어뷰징 방지 정책 
  - 사용자는 게시글 1개당 10분에 1번 조회 집계

<br>

#### 조회수 설계
- 조회수는 게시글이 조회된 **횟수만** 저장하면 된다.
    - 좋아요 수, 게시글 수, 댓글 수처럼 다른 데이터 개수로 파생X
    - 사용자는 전체 내역을 확인하지 못하므로, 조회수만 있으면 된다.
    - 즉, 전체 조회수를 단일 레코드에 비정규화 상태로 바로 저장해도 충분

```text
- 게시글/댓글/좋아요는 아래와 같은 처리 과정으로 수행했다.
    1. start transaction
    2. 게시글/댓글/좋아요 데이터 삽입
    3. 게시글/댓글/좋아요 수 데이터 갱신
    4. commit

- 게시글/댓글/좋아요는 데이터 일관성이 중요(불일치 발생시 사용자가 인지 가능)
트래픽은 비교적 적다.
- 조회수는 비교적 덜 중요하다. (단순히 조회된 횟수만 보여주면 되고, 불일치 인지 어렵다)
트래픽은 비교적 많다.

- 게시글/댓글/좋아요 수는 트랜잭션을 지원하는 RDBMS(MySQL)를 사용했다.
트랜잭션을 사용하여 데이터의 일관성을 지키고, 안전한 저장소(디스크)에 영구적으로 저장한다.
=> 디스크 접근 비용과 트랜잭션 관리 비용 발생

- 조회수는 데이터 일관성이 덜 중요하고, 다른 데이터에 의해 파생되는 데이터가 아니므로, 트랜잭션이나 안전한 저장소가 반드시 필요하진 않다.
트래픽이 많다. => 디스크는 접근 비용이 비싸고 디스크보다 빠른 저장소 사용 고려

디스크는 속도는 느리고 용량이 크다. 가격은 용량 대비 저렴, 비휘발성
메모리는 속도는 빠르고 용량이 작다. 가격은 용량 대비 비쌈, 휘발성
```

#### Redis
- In-memory Database
- 고성능
- NoSQL Database : 관계형 데이터베이스와 달리 정해진 스키마가 없고, 유연한 데이터 모델 사용
- 키- 값 저장소
- String, List, Set, Sorted Set, Hash 등 다양한 자료 구조 지원
- TTL(Time To Live) 지원 : 일정 시간이 지나면 데이터 자동 삭제
- Single Thread 
  - 단일 스레드에서 순차 처리 => 동시성 문제 해결 
- 데이터 백업 지원
  - 메모리는 휘발성 저장소지만, 데이터를 안전한 디스크에 저장하는 방법 제공(AOF, RDB)
- Redis Cluster
  - 확장성, 부하분산, 고가용성을 위한 분산 시스템 구성 방법 제공

<br>

- 고성능 작업 : 메모리는 상대적으로 빠르기 때문에 Redis 자체를 DB로 이용 가능, 동시성 문제에 유리
- 캐시 : 더 느린 저장소에서 더 빠른 저장소에 데이터를 저장해두고 접근하는 기술
  - 매번 디스크에서 데이터 접근하면 느리므로, redis에 데이터를 일정 시간 캐시해둘 수 있다.
- pub/sub : 메시지 발행 및 구독을할 수 있기때문에 실시간 통신에도 활용 가능

<br>

#### Distributed In-memory Database - Redis Cluster
- Redis에서도 Distributed를 구성하기 위한 기능을 제공
  - 여러 개의 Redis 서버가 클러스터를 이루면서 분산 시스템을 이룬다.
  - 확장성, 부하 분산, 고가용성, 안정성 등을 지원하기 위한 Redis Cluster
- Redis를 수평적으로 확장할 수 있게 해주는 기능
- 샤딩을 지원한다
  - 확장성을 위해 논리적 샤드를 지원한다. (Logical Shard) : 16384개의 slot
  -  shard 선택 방식 (Physical Shard)
    - key의 hash 값으로 slot(Logical Shard)을 구하고, slot으로 shard(Physical Shard) 선택
    - 1. slot = hash_function(key) 2. shard = select_shard(slot)
- 서버가 추가되면, 자동으로 데이터가 분산된다. 
- 데이터 복제 기능을 제공한다. (고가용성)
```text
Redis Cluster는 샤딩을 지원하고, 16,384개의 논리적 샤드(=Slot=슬롯)로 분리된다.
이러한 슬롯은 각 물리적 샤드에 균등하게 분산될 수 있다.
이를 통해 확장성과 부하 분산의 이점을 가진다.

Redis Cluster는 데이터 복제도 지원한다.
이를 통해 장애 시에도 유연하게 대처할 수 있는 고가용성을 제공한다.
```

<br>

#### 조회수 설계
- 조회수의 데이터 일관성이 비교적 덜 중요하다고는 하나, 완전히 유실 되어서는 안되는 것이다. => 어느 정도 데이터의 영속성은 필요한 것이다.
- Redis에는 디스크로의 데이터 백업을 위해, AOF와 RDB 기능을 제공한다.
    - AOF(Append Only File) : 수행된 명령어를 로그 파일에 기록하고, 데이터 복구를 위해 로그를 재실행
    - RDB(Snapshot) : 저장된 데이터를 주기적으로 파일에 저장

<br>

- 애플리케이션에서 Redis에 저장된 데이터를 MySQL에 직접 백업도 할것
- 약간의 데이터 유실은 허용한다는 관점이기 때문에, 실시간으로 모든 데이터를 백업할 필요는 없다.
- 시간 단위 백업
  - N분 단위로 Redis의 데이터를 MySQL로 백업
  - 배치 또는 스케줄링 시스템 구축 필요
  - 백업 전 장애 시에 유실될 수 있다.
- 개수 단위 백업
  - N개 단위로 Redis의 데이터를 MySQL로 백업
  - 조회 시점에 간단히 처리 가능
  - 백업 전 장애 시에 유실될 수 있음   
  - 개수 단위 안채워지면 유실될 수 있음

#### Docker를 이용한 Redis 환경 세팅
- $ docker run --name board-redis -d -p 6379:6379 redis:7.4
- 조회수 백업용도 테이블
```sql
create table article_view_count (
    article_id bigint not null primary key,
    view_count bigint not null
);
```

<br>

### 2. 조회수 구현
___
- dependency, application.yml 파일 추가 기존과 같은 설정 + redis 설정
- Repository 
```java
@Repository
@RequiredArgsConstructor
public class ArticleViewCountRepository {
    private final StringRedisTemplate redisTemplate;

    // view::aritcle::{article_id}::view_count
    private static final String KEY_FORMAT = "view::aritcle::%s::view_count";

    // 조회수를 읽는 메소드
    public Long read(Long articleId) {
        String result = redisTemplate.opsForValue().get(generateKey(articleId));// 키 생성해서 조회
        return result == null ? 0L : Long.valueOf(result);
    }

    public Long increase(Long articleId) {
        return redisTemplate.opsForValue().increment(generateKey(articleId));
    }

    private String generateKey(Long articleId) {    // 키 생성
        return KEY_FORMAT.formatted(articleId);
    }
}
```
- Service
```java
@Service
@RequiredArgsConstructor
public class ArticleViewService {
    private final ArticleViewCountRepository articleViewCountRepository;

    public Long increase(Long articleId, Long userId) {
        return articleViewCountRepository.increase(articleId);
    }

    public Long count(Long articleId) {
        return articleViewCountRepository.read(articleId);
    }
}
```
- backup 용 entity
```java
@Table(name = "article_view_count")
@Entity
@Getter
@ToString
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class ArticleViewCount {
    @Id
    private Long articleId; // shard key
    private Long viewCount;

    public static ArticleViewCount init(Long articleId, Long viewCount) {
        ArticleViewCount articleViewCount = new ArticleViewCount();
        articleViewCount.articleId = articleId;
        articleViewCount.viewCount = viewCount;
        return articleViewCount;
    }
}
```
- backup 용 repository
```java
@Repository
public interface ArticleViewCountBackUpRepository extends JpaRepository<ArticleViewCount, Long> {

    @Query(
            value = "update article_view_count set view_count = :viewCount " +
                    "where article_id = :articleId and view_count < :viewCount",
            nativeQuery = true
    )
    @Modifying
    int updateViewCount(
            @Param("articleId") Long articleId,
            @Param("viewCount") Long viewCount
    );
}
```
- test
```java
@SpringBootTest
class ArticleViewCountBackUpRepositoryTest {
    @Autowired
    ArticleViewCountBackUpRepository articleViewCountBackUpRepository;
    @PersistenceContext
    EntityManager entityManager;

    @Test
    @Transactional
    void updateViewCountTest() {
        // given
        articleViewCountBackUpRepository.save(
                ArticleViewCount.init(1L, 0L)
        ); // 데이터 생성
        entityManager.flush();
        entityManager.clear();

        int result1 = articleViewCountBackUpRepository.updateViewCount(1L, 100L);
        int result2 = articleViewCountBackUpRepository.updateViewCount(1L, 300L);
        int result3 = articleViewCountBackUpRepository.updateViewCount(1L, 200L);

        assertThat(result1).isEqualTo(1);
        assertThat(result2).isEqualTo(1);
        assertThat(result3).isEqualTo(0);

        ArticleViewCount articleViewCount = articleViewCountBackUpRepository.findById(1L).get();
        assertThat(articleViewCount.getViewCount()).isEqualTo(300L);
    }
}
```
- backup processor
```java
@Component
@RequiredArgsConstructor
public class ArticleViewCountBackUpProcessor {
    private final ArticleViewCountBackUpRepository articleViewCountBackUpRepository;

    @Transactional
    public void backup(Long articleId, Long viewCount){
        int result = articleViewCountBackUpRepository.updateViewCount(articleId, viewCount);
        if(result == 0){    // 삽입된 레코드가 없을때
            articleViewCountBackUpRepository.findById(articleId)
                    .ifPresentOrElse(ignored -> { },
                            () -> articleViewCountBackUpRepository.save(ArticleViewCount.init(articleId, viewCount)));
        }
    }
}
```
- service mysql 백업 로직 추가
```java
@Service
@RequiredArgsConstructor
public class ArticleViewService {
    private final ArticleViewCountRepository articleViewCountRepository;
    private final ArticleViewCountBackUpProcessor articleViewCountBackUpProcessor;
    private static final int BACKUP_BATCH_SIZE = 100;

    public Long increase(Long articleId, Long userId) {
        Long count = articleViewCountRepository.increase(articleId);
        if(count % BACKUP_BATCH_SIZE == 0) {
            articleViewCountBackUpProcessor.backup(articleId, count);
        }
        return count;
    }

    public Long count(Long articleId) {
        return articleViewCountRepository.read(articleId);
    }
}
```
- controller
```java
@RestController
@RequiredArgsConstructor
public class ArticleViewController {
    private final ArticleViewService articleViewService;

    @PostMapping("/v1/article-views/articles/{articleId}/users/{userId}")
    public Long increase(
            @PathVariable("articleId") Long articleId,
            @PathVariable("userId") Long userId
    ) {
        return articleViewService.increase(articleId, userId);
    }

    @GetMapping("/v1/article-views/articles/{articleId}/count")
    public Long count(@PathVariable("articleId") Long articleId) {
        return articleViewService.count(articleId);
    }
}
```
- test
```java
public class ViewApiTest {
    RestClient restClient = RestClient.create("http://localhost:9003");

    @Test
    void viewTest() throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(100);
        CountDownLatch latch = new CountDownLatch(10000);

        for (int i = 0; i < 10000; i++) {
            executorService.submit(() -> {
                restClient.post()
                        .uri("/v1/article-views/articles/{articleId}/users/{userId}", 3L, 1L)
                        .retrieve();
                latch.countDown();
            });
        }

        latch.await();

        Long count = restClient.get()
                .uri("/v1/article-views/articles/{articleId}/count", 3L)
                .retrieve()
                .body(Long.class);

        System.out.println("count = " + count);
    }
}
```

<br>

### 3. 조회수 어뷰징 방지 정책 설계
___
- 조회 식별 : 로그인 사용자 -> 사용자, 비로그인 사용자 -> ip, User-Agent, 브라우저 쿠키, 토큰 등
- 사용자가 10분 내 게시글을 조회 했던사실을 인지하는 방법
  - 스프링부트는 stateless(무상태) 애플리케이션
  - 상태 저장소로 DB 사용

```text
MySQL 사용
1. 조회수 증가 요청이 오면, 마지막 조회 시점을 조회
2. 10분 이내의 조회 내역이 있는지 확인
3. 조회 내역에 따라서,
    1. 조회 내역이 없다면, 조회수를 증가 그리고 현재 시간으로 마지막 조회 시점을 업데이트
    2. 조회 내역이 있다면, 조회수를 증가X
=> 
- 트래픽이 많을 수 있다. => 성능을 위해 Redis 를 사용했는데 MySQL을 다시 사용하는 ..
- 동시성 문제 => MySQL은 조회수 동시 요청이 들어오면 락을 점유 => Redis 는 Single Thread , 하나의 명령어는 원자적으로 처리
- 자동 삭제 => MySQL은 게시글이 삭제되거나 갱신될 일이 없다면, 직접 삭제를 위한 배치등의 시스템을 구축해야한다.
 => Redis 는 TTL 을 지원(시간이 지나면 자동삭제)


- Redis 활용
1. 조회수 증가 요청이 오면, Redis에 TTL=10분 으로 데이터 저장     
    - 게시글 조회는 사용자 단위로 식별되므로, key=(articleId+userId)
    - 이미 저장된 데이터가 있으면 저장에 실패하는 명령어를 사용 => setIfAbsent (데이터가 없을 때에만 저장)
2. 데이터 저장 성공 여부에 따라,
    - 성공 했으면, 조회 내역이 없었음을 의미한다. 조회수를 증가한다.
    - 실패 했으면, 조회 내역이 있었음을 의미한다. 조회수를 증가하지 않는다    

- 이러한 과정은 사용자의 게시글 조회수 증가에 대해서 Lock을 획득한다고 볼 수 있다.
시스템은 확장성이 고려된 분산 시스템이다. 
이렇게 분산 시스템에서 락을 획득하는 것을, 분산 락이라고 한다. => Distributed Lock
- 조회수 서비스의 여러 서버 애플리케이션들은 사용자의 게시글 조회수 증가에 대해서 10분 간 분산 락을 획득
분산 락이 점유되면, 다른 요청은 락이 10분 후에 해제되기 까지, 락을 추가로 점유할 수 없다.

*** 동일 게시글에 동일한 사용자가 2개의 조회수 증가 요청을 동시에 호출
- 요청 1은 조회수 증가를 처리 하기 위해 분산 락을 요청 => key = articleId + userId, TTL = 10분
- 요청 1은 분산 락이 이미 점유된 게 없었기 때문에, 분산 락 획득에 성공한다. => setIfAbsent = True 
- 이어서 동일 게시글에 동일 사용자가 분산 락을 다시 요청
- 하지만 요청 1에서 이미 분산 락을 10분 간 점유 했으므로, 요청 2는 분산 락 획득에 실패 => setIfAbsent = False
- 요청 1에서는 조회 수 증가를 처리하고, 요청 2는 그냥 종료 된다. 
=> 분산 락이 점유되고 있는 10분 동안, 동일 게시글에 동일 사용자에 대한 모든 조회수 증가 요청은 무시된다.
Redis는 Single Thread로 동작하고, setIfAbsent는 원자적으로 처리되기 때문에, 동시성 고려는 필요 없었다.
```

<br>

### 4. 조회수 어뷰징 방지 정책 구현
___
- repository
```java
@Repository
@RequiredArgsConstructor
public class ArticleViewDistributedLockRepository {
    private final StringRedisTemplate redisTemplate;

    // view::article::{article_id}::user::{user_id}::lock
    private static final String KEY_FORMAT = "view::article::%s::user::%s::lock";

    // lock 획득 메소드
    public boolean lock(Long articleId, Long userId, Duration ttl) {
        String key = generateKey(articleId, userId);
        return redisTemplate.opsForValue().setIfAbsent(key, "", ttl);   // 이미 저장된 데이터가 있으면 false
    }

    public String generateKey(Long articleId, Long userId) {
        return KEY_FORMAT.formatted(articleId, userId);
    }
}
```
- Service 에 조회수 증가 메서드에 어뷰징 방지 조건 추가
```java
@Service
@RequiredArgsConstructor
public class ArticleViewService {
    private final ArticleViewCountRepository articleViewCountRepository;
    private final ArticleViewCountBackUpProcessor articleViewCountBackUpProcessor;
    private final ArticleViewDistributedLockRepository articleViewDistributedLockRepository;

    private static final int BACKUP_BATCH_SIZE = 100;
    private static final Duration TTL = Duration.ofMinutes(10);

    public Long increase(Long articleId, Long userId) {
        /* 어뷰징 방지 S, 실패시 증가 X, 현재 조회수 반환 */
        if (!articleViewDistributedLockRepository.lock(articleId, userId, TTL)) {
            return articleViewCountRepository.read(articleId);
        }
        /* 어뷰징 방지 E */

        Long count = articleViewCountRepository.increase(articleId);
        if(count % BACKUP_BATCH_SIZE == 0) {
            articleViewCountBackUpProcessor.backup(articleId, count);
        }
        return count;
    }

    public Long count(Long articleId) {
        return articleViewCountRepository.read(articleId);
    }
}
```
- 기존 Multi Thread로 1000번 조회하는 테스트 실행시(같은 ID 라는 조건) => 조회수 1이 증가됨을 확인할 수 있다.
