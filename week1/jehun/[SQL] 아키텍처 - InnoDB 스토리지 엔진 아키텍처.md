<h1 id="innodb-스토리지-엔진-아키텍처">InnoDB 스토리지 엔진 아키텍처</h1>
<p> <img alt="" src="https://velog.velcdn.com/images/jojehuni_9759/post/3a415824-7d64-4671-a8e2-6e7ad1b74419/image.png" /></p>
<p>MySQL의 스토리지 엔진 중 가장 많이 사용되는 InnoDB 스토리지 엔진을 살펴보자.</p>
<ul>
<li>레코드 기반의 잠금 제공 -&gt; 높은 동시성 처리 가능</li>
<li>안정적이며 성능이 뛰어나다.</li>
</ul>
<p>주요 특징들을 보자</p>
<h2 id="프라이머리-키에-의한-클러스터링">프라이머리 키에 의한 클러스터링</h2>
<ul>
<li><p>InnoDB의 모든 테이블은 프라이머리 키를 기준으로 클러스터링 돼 저장된다. </p>
<ul>
<li><strong>프라이머리 키 값의 순서대로 디스크의 저장된다.</strong></li>
</ul>
</li>
<li><p>모든 세컨더리 인덱스는 레코드 주소 대신 프라이머리 키의 값을 논리적인 구조로 사용한다.</p>
</li>
<li><p>클러스터링 인덱스여서 Range 스캔이 상당이 빠르게 처리된다.</p>
</li>
<li><p>쿼리의 실행 계획에서 다른 보조 인덱스보다 비중이 높게 설정돼, 프라이머리 키가 인덱스로 선택될 확률이높다.</p>
</li>
</ul>
<blockquote>
<p><strong>MyISAM 스토리지 엔진</strong>
MyISAM 스토리지 엔진에서는 클러스터링 키 지원 X
즉, 프라이머리 키와 세컨더리 인덱스는 구조적인 차이가 없다. (유니크 제약을 가진 세컨더리 인덱스와 같다.)</p>
<p>모든 인덱스는 물리적인 레코드의 주소 값을 가진다.</p>
<p>해당 내용은 이후에 더 자세하게 다룬다.</p>
</blockquote>
<hr />
<h2 id="외래키-지원">외래키 지원</h2>
<ul>
<li>InnoDB 스토리지 엔진 Level 에서만 지원하는 기능이다.</li>
<li>외래키는 실제 서비스 운영 환경에 사용하지 않는 경우가 있다. <ul>
<li>그 이유는 부모와 자식 테이블에 인덱스 생성이 필요하고, 변경 시 잠금이 여러 테이블로 전파되어 데드락이 발생할 확률이 높아지기 때문이다.</li>
</ul>
</li>
<li><code>foreign_key_checks</code> 시스템 변수를 OFF로 설정하여 외래키 관계 체크 작업을 일시적으로 멈출 수 있다.</li>
</ul>
<blockquote>
<p><code>foreign_key_checks</code> 시스템 변수를 OFF한 경우 데이터 일관성에 문제가 생길 수 있으므로 <strong>반드시 부모와 자식 테이블 간 일관성을 맞춘 후 다시 활성화해야 한다.</strong></p>
</blockquote>
<pre><code class="language-sql">mysql&gt; SET foreign_key_checks=OFF;

-- // 작업 실행 후

mysql&gt; SET foreign_key_checks=ON;</code></pre>
<hr />
<h2 id="mvccmulti-version-concurrency-control">MVCC(Multi Version Concurrency Control)</h2>
<p>레코드 Level의 트랜잭션을 지원하는 DBMS가 제공하는 기능</p>
<p>MVCC의 가장 큰 목적은 <strong>잠금을 사용하지 않는 일관된 읽기를 제공</strong>하는데 있다. InnoDB 스토리지 엔진은 Undo log를 이용해 MVCC 기능을 구현했다.</p>
<ul>
<li><strong>Multi Version</strong>의 뜻? : <strong>레코드에 대해 여러 개의 버전이 동시에 관리된다는 의미</strong><ul>
<li>격리 수준(Isolation Level)에 따라 처리되는 방식이 다르다.</li>
</ul>
</li>
</ul>
<p><code>READ_UNCOMMITTED</code>의 경우 InnoDB 버퍼 풀이나 데이터 파일로부터 데이터를 읽어 반환한다. 
즉, 커밋됐든 아니든 변경된 상태의 데이터를 반환하게 된다.</p>
<p>그 이상의 격리 수준인 <code>READ_COMMITTED</code>, <code>REPEATABLE_READ</code>은 InooDB 버퍼 풀이나 데이터 파일 대신 변경되기 전 내용을 보관하고 있는 Undo 영역의 데이터를 반환한다.</p>
<p>위와 같은 과정을 MVCC라고 표현한다.</p>
<hr />
<h2 id="잠금-없는-일관된-읽기">잠금 없는 일관된 읽기</h2>
<p>InnoDB는 MVCC 기술을 활용해서 잠금을 걸지 않고 읽기 작업을 수행하는데, <strong>SERIALIZABLE이 아닌 다른 격리 수준에서는 다른 트랜잭션이 가지고 있는 잠금을 기다리지 않고 읽기 작업이 가능</strong>하다.</p>
<blockquote>
<p>SERIALIZABLE 이 아닌 것들 : READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ</p>
</blockquote>
<p><img alt="" src="https://velog.velcdn.com/images/jojehuni_9759/post/e2d8d67e-e907-4700-90b6-c0d71016b849/image.png" /></p>
<hr />
<h2 id="자동-데드락-감지">자동 데드락 감지</h2>
<ul>
<li><p>InnoDB는 스토리지 엔진은 잠금 대기 목록을 그래프 형태로 관리한다.</p>
<ul>
<li>내부적으로 잠금이 교착 상태에 빠지지 않았는지 체크하기 위해서</li>
</ul>
</li>
<li><p>데드락 감지 스레드가 주기적으로 잠금 대기 그래프를 검사해 교착 상태에 빠진 트랜잭션을 강제 종료한다.</p>
<ul>
<li><strong>언두 로그의 양이 적은 트랜잭션이 먼저 종료된다.</strong></li>
</ul>
</li>
<li><p>데드락 감지 스레드가 잠금 목록을 체크할 때 잠금 목록에 새로운 잠금이 걸리므로 동시 처리 스레드가 매우 많은 경우 -&gt; 많은 CPU 자원 소모가 생길 수 있다. </p>
<ul>
<li>따라서 서비스에 성능에 악영향을 줄 수 있는 상황이 생긴다.</li>
</ul>
</li>
</ul>
<blockquote>
<p>위와 같은 문제를 해결하는 방법
<code>innodb_deadlock_detect</code> 시스템 변수를 OFF 하여 데드락 감지 스레드를 작동하지 않게 한 뒤, <code>innodb_lock_wait_timeout</code> 시스템 변수를 활성화하여 데드락 상황에서 일정 시간 동안 잠금을 획득하지 못했을 경우 요청이 실패하고 에러 메시지를 반환하게 하는 방식으로 문제를 우회할 수 있다.</p>
</blockquote>
<hr />
<h2 id="자동화된-장애-복구">자동화된 장애 복구</h2>
<p>InnoDB 에는 손실이나 장애로부터 데이터를 보호하기 위해 여러 메커니즘이 탑재돼 있다. 
MySQL 서버가 시작될 때 완료되지 못한 트랜잭션이나 디스크에 일부만 기록된 데이터 페이지 등에 대한 일련의 복구 작업이 자동으로 된다.</p>
<p>그럼에도 <strong>디스크나 서버 하드웨어 이슈로 자동 복구를 못 하는 경우, 자동 복구를 멈추고 MySQL 서버가 종료돼 버린다.</strong></p>
<p>이 때, <code>innodb_force_recovery</code> 시스템 변수를 설정해서 MySQL 서버를 시작해야 한다.
단계 별로 알아보자.</p>
<blockquote>
<p>1 ~ 6의 값을 가지며, 0이 아닌 복구 모드에서는 SELECT 이외의 쿼리는 수행할 수 없다.</p>
</blockquote>
<ul>
<li><p><code>innodb_force_recovery</code> - 1 (SRV_FORCE_IGNORE_CORRUPT)</p>
<ul>
<li>InnoDB의 테이블스페이스의 데이터나 인덱스 페이지에서 손상된 부분이 발견돼도 무시하고 MySQL 서버를 시작한다.</li>
</ul>
</li>
<li><p><code>innodb_force_recovery</code> - 2 (SRV_FORCE_NO_BACKGROUND)
-백그라운드 스레드 가운데 메인 스레드를 시작하지 않고 MySQL 서버를 시작한다. InnoDB의 메인 스레드가 Undo 데이터를 삭제하는 과정에서 장애가 발생한다면 이 모드로 복구하면 된다.</p>
</li>
<li><p><code>innodb_force_recovery</code> - 3 (SRV_FORCE_NO_TRX_UNDO)</p>
<ul>
<li>커밋되지 않고 종료된 트랜잭션은 계속 그 상태로 남아 있게 MySQL 서버를 시작하는 모드</li>
</ul>
</li>
<li><p><code>innodb_force_recovery</code> - 4 (SRV_FORCE_NO_IBUF_MERGE)</p>
<ul>
<li>InnoDB 스토리지 엔진이 Insert Buffer의 내용을 무시하고 강제로 MySQL이 시작되게 한다.</li>
<li>Insert Buffer는 실제 데이터와 관련된 부분이 아니라 Index에 관련된 부분이므로 테이블을 덤프 한 후 다시 데이터베이스를 구축하면 데이터의 손실 없이 복구할 수 있다.</li>
</ul>
</li>
<li><p><code>innodb_force_recovery</code> - 5 (SRV_FORCE_NO_UNDO_LOG_SCAN)</p>
<ul>
<li>InnoDB 엔진이 UDNO 로그를 모두 무시하고 MySQL을 시작할 수 있다. </li>
<li>하지만 이 모드로 복구되면 MySQL 서버가 종료되던 시점에 커밋되지 않았던 작업도 모두 커밋된 것처럼 처리되므로 실제로는 잘못된 데이터가 데이터베이스에 남는 것이라고 볼 수 있다.</li>
</ul>
</li>
<li><p><code>innodb_force_recovery</code> - 6 (SRV_FORCE_NO_LOG_REDO)</p>
<ul>
<li>InnoDB 엔진이 리두 로그를 모두 무시한 채로 MySQL 서버가 시작된다. 커밋됐다 하더라도 REDO 로그에만 기록되고 데이터 파일에 기록되지 않은 데이터는 모두 무시된다. </li>
<li>즉, 마지막 체크포인트 시점의 데이터만 남게 된다.</li>
</ul>
</li>
</ul>
<p>위 6단계를 진행했어도 시작하지 않는다면 -&gt; 백업을 이용해 다시 구축하는 방법밖에 없다.</p>
<blockquote>
<p>백업이 있으면 마지막 백업으로 DB를 새로 구축하고, Binary log을 사용해 최대한 장애 시점까지의 데이터 복구도 가능하다.</p>
</blockquote>
<p><strong>마지막 풀 백업 시점과 Binary log가 있다면</strong>, InnoDB의 복구를 이용하는 것보다 <strong>마지막 풀 백업 시점과 Binary log를 사용해서 복구하는 것이 데이터 손실이 더 적다.</strong></p>
<p>단, 백업은 있어도 Binary log가 없다면 마지막 백업 시점까지만 복구 가능하다.</p>
<hr />
<h2 id="innodb-버퍼-풀">InnoDB 버퍼 풀</h2>
<p><strong>InnoDB 스토리지 엔진에서 가장 핵심적인 부분</strong></p>
<ul>
<li><strong>디스크의 데이터 파일이나 인덱스 정보를 메모리에 캐시해 두는 공간</strong>이다.</li>
<li>쓰기 작업을 지연시켜 일괄 작업으로 처리할 수 있게 해주는 <strong>버퍼 역할도 같이 한다.</strong></li>
</ul>
<blockquote>
<p><strong>버퍼 역할?</strong></p>
<p>Insert, Update, Delete처럼 데이터를 변경하는 쿼리는 데이터 파일의 이곳저곳에 위치한 레코드를 변경하기 때문에 랜덤한 디스크 작업을 발생시킨다.</p>
<p>하지만 버퍼 풀이 이러한 변경된 데이터를 모아서 처리하면 랜덤한 디스크 작업의 횟수를 줄일 수 있다.</p>
</blockquote>
<h3 id="버퍼-풀의-구조">버퍼 풀의 구조</h3>
<p>InnoDB 스토리지 엔진은 버퍼 풀이라는 거대한 메모리 공간을 페이지 크기(innodb_page_size 시스템 변수에 설정된)의 조각으로 쪼개 스토리지 엔진이 필요로 할 때 해당 데이터 페이지를 읽어 각 조각에 저장되는 구조이다.</p>
<p>InnoDB 스토리지 엔진은 버퍼 풀의 페이지 조각을 관리하기 위해 LRU리스트, 플러시 리스트, 프리 리스트라는 3개의 자료 구조를 관리한다.</p>
<ul>
<li><p>LRU(Least Recently Used) 리스트</p>
<ul>
<li>디스크로부터 한 번 읽어온 페이지를 최대한 오랫동안 InnoDB 버퍼 풀 메모리에 유지해서 디스크 읽기를 최소화</li>
</ul>
</li>
<li><p>플러시 리스트</p>
<ul>
<li><strong>디스크로 동기화되지 않은 데이터 페이지(더티 페이지)</strong> 의 변경 시점 기준의 페이지 목록을 관리</li>
</ul>
</li>
<li><p>프리 리스트</p>
<ul>
<li>데이터가 채워지지 않은 비어있는 페이지 목록으로, 새롭게 디스크의 데이터 페이지를 읽어와야 하는 경우 사용된다.</li>
</ul>
</li>
</ul>
<hr />
<h3 id="버퍼-풀과-리두-로그">버퍼 풀과 리두 로그</h3>
<p>버퍼 풀과 Redo log는 매우 밀접한 관계를 맺고 있다.</p>
<p>버퍼 풀은 서버의 메모리가 허용하는 만큼 크게 설정하면 할 수록 쿼리의 성능이 빨라진다.</p>
<p>하지만 데이터베이스 성능 향상을 위해 데이터 캐시 기능과 쓰기 버퍼링 기능을 지원하는데, 버퍼 풀의 메모리 공간을 늘리는 것은 데이터 캐시 기능만 향상시키는 것이다.</p>
<p>쓰기 버퍼링 기능까지 향상시키려면 리두 로그와의 관계를 먼저 이해해야 한다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/jojehuni_9759/post/0ffd58e7-0ca1-4e02-a3ef-fc059c53c1bf/image.png" /></p>
<h4 id="innodb-버퍼-풀과-리두-로그의-관계-설명의-흐름">InnoDB 버퍼 풀과 리두 로그의 관계 설명의 흐름</h4>
<ol>
<li><p>InnoDB 버퍼 풀은 디스크에서 읽은 상태 그대로인 클린 페이지와 여러 명령(INSERT, UPDATE, DELETE) 으로 인해 변경된 더티 페이지(변경된 데이터)를 가진다.</p>
</li>
<li><p>더티 페이지는 언젠가 디스크에 기록되어야 한다.</p>
</li>
<li><p>한정된 메모리 공간인 버퍼 풀에 더티 페이지가 계속 머무를 수 없으므로 리두 로그 파일과의 순환 고리를 이용해 데이터 변경을 기록한다.</p>
</li>
<li><p>리두 로그 파일에 데이터 변경이 기록될 때마다 LSN이라는 시퀀스 번호를 증가시킨다.</p>
</li>
<li><p>InnoDB 스토리지 엔진은 주기적으로 체크포인트 이벤트를 발생시켜 그 시점의 리두 로그 파일의 LSN을 시작점으로 삼는다.(마지막 LSN은 계속 증가하며 데이터 변경을 기록)</p>
</li>
<li><p>체크포인트가 발생하면 LSN 시작점부터 그보다 작은 LSN을 가진 리두 로그 엔트리와 관련된 버퍼 풀의 더티 페이지를 디스크로 동기화한다.</p>
</li>
</ol>
<p>이렇듯, 버퍼 풀의 더티 페이지 비율과 리두 로그 파일의 전체 크기는 쓰기 버퍼링 기능 성능에 관계되어 있다. </p>
<blockquote>
<p>따라서 우리가 데이터베이스 성능을 최적화하고 싶다면 InnoDB 버퍼 풀과 리두 로그 파일의 전체 크기를 적절히 선택해 최적 값을 찾아야 한다.</p>
</blockquote>
<ul>
<li>113 쪽 : 버퍼 풀 플러시</li>
</ul>
<hr />
<h3 id="버퍼-풀-상태-백업-및-복구">버퍼 풀 상태 백업 및 복구</h3>
<p>버퍼 풀과 쿼리의 성능과 매우 밀접하게 연결돼 있다.</p>
<p>버퍼 풀에 쿼리들이 사용한 데이터가 이미 준비돼 있으므로 디스크에서 데이터를 읽지 않아도 되기 때문에 쿼리 성능이 상당히 올라가게 된다.</p>
<p><strong>하지만</strong> 쿼리 요청이 매우 빈번한 서버를 재시작하면 성능이 저하되는 것을 볼 수 있는데, 이는 버퍼 풀에 데이터가 올라가 있지 않아서 모든 데이터를 디스크로부터 읽어와야 하기 때문에 발생하는 성능 저하이다.</p>
<blockquote>
<p>MySQL 5.5 버전에서는 서버 시작 전 주요 테이블과 인덱스에 대해 풀 스캔을 실행하고 서비스를 오픈했었다.</p>
<p>하지만 MySQL 5.6 버전부터는 버퍼 풀 덤프 및 적재 기능이 도입되어 <code>innodb_buffer_pool_dump_now</code>시스템 변수를 이용해 현재 버퍼 풀의 상태를 백업할 수 있게 됐다.</p>
</blockquote>
<hr />
<h2 id="double-write-buffer">Double Write Buffer</h2>
<p>Redo log는 Redo log 공간의 낭비를 막기 위해 페이지의 변경된 내용만 기록한다. </p>
<p>하드웨어의 오작동이나 시스템 비정상 종료 등의 문제로 인해 InnoDB 스토리지 엔진에서 더티 페이지를 디스크 파일로 플러시할 때 일부만 기록되는 문제(파셜 페이지 또는 톤 페이지)가 발생하면 그 페이지에 대한 내용을 복구하지 못할 수 있다.</p>
<blockquote>
<p>이러한 문제를 해결할 방법? -&gt; <strong>Double-Write</strong></p>
</blockquote>
<p><img alt="" src="https://velog.velcdn.com/images/jojehuni_9759/post/5494678f-bd62-490c-93a7-b20d4b468690/image.png" /></p>
<p>위 그림과 같이 시스템 테이블스페이스에 DoubleWrite 버퍼 공간에 기록된 변경 내용은 실제 데이터 파일에 'A' ~ 'E' 더티 페이지가 정상적으로 기록되면 더이상 필요 없어진다.</p>
<p>DoubleWrite 버퍼의 내용은 데이터 파일의 쓰기가 중간에 실패할 때만 원래의 목적으로 사용되는데</p>
<p>전체적인 흐름은 이런 식이다.</p>
<ol>
<li><p>실제 데이터 파일에 변경 내용을 기록하기 전에 더티 페이지를 묶어 시스템 테이블스페이스의 DoubleWrite 버퍼에 기록한다.</p>
</li>
<li><p>InnoDB 스토리지 엔진은 각 더티 페이지를 파일의 적당한 위치에 랜덤 쓰기를 실행한다.</p>
</li>
<li><p>데이터 파일의 페이지들과 DoubleWirte 버퍼의 내용을 비교한다.</p>
</li>
<li><p>데이터 파일 페이지와 DoubleWirte 버퍼의 내용이 다르다면 DoubleWirte 버퍼의 내용을 데이터 파일의 페이지로 복사한다.</p>
</li>
</ol>
<p><strong>DoubleWirte 버퍼는 데이터의 안정성을 위해 자주 사용된다.</strong></p>
<p>다만, SSD처럼 랜덤 IO 저장 시스템에서는 상당히 부담스러울 수 있으며 HDD처럼 자기 원판 순차 디스크 쓰기 저장 시스템에서 사용하기 덜 부담스럽다. </p>
<p>그럼에도 데이터의 무결성이 매우 중요하다면 DoubleWrite를 활성화하는 것이 좋다.</p>
<hr />
<h2 id="undo-log">Undo Log</h2>
<p>InnoDB 스토리지 엔진은 트랜잭션과 격리 수준을 보장하기 위해 DML (INSERT, UPDATE, DELETE)로 변경되기 이전 버전의 데이터를 별도로 백업한다.</p>
<p>그런 백업된 데이터를 <code>Undo Log</code> 라고 한다.</p>
<ul>
<li><strong>트랜잭션 보장</strong><ul>
<li>트랜잭션이 롤백되면 Undo Log에 백업해둔 이전 버전의 데이터를 이용해 복구한다.</li>
</ul>
</li>
<li><strong>격리 수준 보장</strong><ul>
<li>데이터를 변경하는 도중 다른 커넥션에서 데이터를 조회하면 격리 수준에 맞게(READ_COMMITTED, REPEATABLE READ) Undo Log에 백업해둔 데이터를 읽어서 반환한다.</li>
</ul>
</li>
</ul>
<p>Undo Log의 데이터는 크게 트랜잭션의 롤백 대비용 데이터와 격리 수준을 유지하면서 높은 동시성을 제공하기 위한 용도로 사용된다.</p>
<hr />
<h3 id="undo-log-레코드-모니터링">Undo Log 레코드 모니터링</h3>
<p>서비스 중인 MySQL 서버에서 활성 상태의 트랜잭션이 장시간 유지되는 것은 성능상 좋지 않다.</p>
<p>트랜잭션을 시작한 상태에서 완료하지 않고 하루 동안 방치했다고 하자.
시작 시점부터 생성된 Undo Log는 계속 스토리지 엔진에 보존되고, 결국 하루 양의 데이터 변경을 모두 저장해 디스크의 Undo Log 저장 공간은 계속 증가한다.</p>
<p>빈번하게 변경된 레코드를 조회하는 쿼리가 실행되면 스토리지 엔진은 Undo Log의 이력을 필요한 만큼 스캔해야만 레코드를 찾을 수 있다.</p>
<blockquote>
<p>따라서 쿼리의 성능이 전반적으로 떨어진다..</p>
</blockquote>
<p>MySQL 5.5 버전까지는 한 번 증가한 언두 로그 공간은 다시 줄어들지 않는 문제도 있었지만, 이 문제는 MySQL 5.7과 MySQL 8.0으로 업그레이드되면서 완전히 해결됐다고 한다.</p>
<p>하지만 장시간 트랜잭션이 유지되는 것은 성능상 좋지 않은 건 마찬가지이므로 MySQL 서버의 Undo Log 레코드가 얼마나 되는지는 항상 모니터링하는 것이 좋다.</p>
<p>Undo Log 건수를 확인하는 명령들을 보자.</p>
<pre><code class="language-sql">- // MySQL 서버의 모든 버전에서 사용 가능한 명령
SHOW ENGINE INNODB STATUS \G

- // MySQL 8.0 버전에서 사용 가능한 명령
SELECT count
FROM information_schema.innodb_metrics
WHERE SUBSYSTEM='transaction' AND NAME='trx_rseg_history_len';</code></pre>
<hr />
<h2 id="언두-테이블스페이스-관리">언두 테이블스페이스 관리</h2>
<blockquote>
<p><strong>언두 테이블스페이스</strong> : 언두 로그가 저장되는 공간</p>
</blockquote>
<ul>
<li>MySQL 5.6 버전 이전 -&gt; Undo Log가 모두 테이블스페이스에 저장됐다.</li>
</ul>
<p>=&gt; 시스템 테이블스페이스의 Undo Log는 MySQL 서버가 초기화될 때 생성되기 때문에 확장의 한계가 있다.</p>
<ul>
<li><p>MySQL 5.6 버전 : <code>innodb_undo_tablesapces</code> 시스템 변수를 도입</p>
<ul>
<li><code>innodb_undo_tablesapces</code> 시스템 변수 값을 0으로 설정한 경우 언두 로그를 시스템 테이블스페이스에 저장</li>
<li><code>innodb_undo_tablesapces</code> 시스템 변수 값을 2보다 큰 값으로 설정한 경우 별도의 언두 로그 파일에 저장</li>
</ul>
</li>
<li><p>MySQL 8.0</p>
<ul>
<li><code>innodb_undo_tablesapces</code> 시스템 변수 Deprecated</li>
<li>Undo Log를 항상 외부의 별도 로그 파일에 기록</li>
</ul>
</li>
</ul>
<p>아래 그림은 Undo 테이블스페이스가 어떤 형태로 구성되는지 보여준다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/jojehuni_9759/post/e8a1f22a-6111-4a9c-971e-736d1bbb326c/image.png" /></p>
<p>하나의 언두 테이블스페이스는 1개 이상 128개 이하의 롤백 세그먼트를 가지며, 롤백 세그먼트는 1개 이상의 언두 슬롯을 가진다.</p>
<p>하나의 트랜잭션이 필요로 하는 언두 슬롯의 개수는 DML의 특성에 따라 최대 4개까지 사용하게 된다. </p>
<blockquote>
<p>즉, 다음과 같은 수식으로 최대 동시 처리 가능 트랜잭션 개수를 예측해 볼 수 있다.
<code>최대 동시 트랜잭션 수 = (InnoDB 페이지 크기) / 16 * (롤백 세그먼트 개수) ******* (언두 테이블스페이스 개수)</code></p>
</blockquote>
<p>언두 로그 공간이 남는 것은 문제 X
하지만, 언두 로그 슬롯이 부족한 경우 트랜잭션을 시작할 수 없는 심각한 문제가 발생한다.</p>
<p>MySQL 8.0 이전에는 한 번 생성된 언두 로그는 변경이 허용되지 않고 정적으로 사용되었고, 
MySQL 8.0 버전부터는 <code>CREATE UNDO TABLESPACE</code>나 <code>DROP TABLESPACE</code> 같은 명령으로 언두 테이블스페이스를 동적으로 추가하고 삭제할 수 있게 개선되었다.</p>
<hr />
<h2 id="체인지-버퍼">체인지 버퍼</h2>
<blockquote>
<p><strong>체인지 버퍼</strong> : 변경해야 할 인덱스 페이지를 디스크로부터 읽어와야 하는 경우 자원 소모를 줄이기 위해 사용되는 임시 메모리 공간</p>
</blockquote>
<p>레코드가 INSERT 혹은 UPDATE 될 때 데이터 파일을 변경하는 작업뿐 아니라 해당 테이블에 포함된 인덱스도 업데이트하는 작업이 필요하다. </p>
<p><strong>그런데</strong> 테이블에 인덱스가 많다면, 이 작업은 디스크를 랜덤하게 읽는 작업이 필요하다. </p>
<p>InnoDB는 변경해야 할 인덱스 페이지가 버퍼 풀에 있으면 바로 업데이트를 수행하지만, 그렇지 않은 경우 디스크로부터 읽어와야 하기 때문이다.</p>
<p>따라서 이러한 작업은 상당히 많은 자원을 소모하게 된다.</p>
<blockquote>
<p>위와 같은 자원 소모를 막기 위해 InnoDB는 즉시 업데이트를 실행하지 않고 인덱스 레코드를 체인지 버퍼에 저장해두었다가 사용자에게 결과를 반환하는 형태로 성능을 향상시킨다. </p>
<p>이후 체인지 버퍼에 임시로 저장된 인덱스 레코드 조각은 버퍼 머지 스레드라는 백그라운드 스레드에 의해 병합된다.</p>
</blockquote>
<ul>
<li><p>MySQL 5.5 이전 버전</p>
<ul>
<li>INSERT 작업에 대해서만 이러한 버퍼링 기능(인서트 버퍼)을 사용할 수 있었다.</li>
</ul>
</li>
<li><p>MySQL 5.5 버전부터 개선돼 MySQL 8.0 버전부터</p>
<ul>
<li>INSERT, UPDATE, DELETE로 인해 키를 추가하거나 삭제하는 작업에 대해서도 버퍼링이 될 수 있게 개선되었다.</li>
</ul>
</li>
</ul>
<hr />
<h2 id="redo-log-및-log-buffer">Redo Log 및 Log Buffer</h2>
<h3 id="redo-log">Redo Log</h3>
<p>트랜잭선의 4가지 요소인 ACID(원자성, 일관성, 격리성, 지속성) 중 D에 해당하는 영속성과 가장 밀접하게 연관 돼 있는 Redo Log에 대해서 알아보자.</p>
<p>하드웨어나 소프트웨어 등 여러 가지 문제점으로 인해 <strong>MySQL 서버가 비정상적으로 종료되었을 때 데이터 파일에 기록되지 못한 데이터를 잃지 않게 해주는 안전장치 역할</strong>을 한다.</p>
<p>모든 DBMS에서 데이터 파일은 쓰기보다 읽기 성능을 고려한 자료구조가 있다.</p>
<p>쓰기는 디스크의 랜덤 액세스가 필요하다.. -&gt; 쓰기에 더 큰 비용이 필요함.</p>
<p>이로 인한 성능 저하를 막기 위해 데이터베이스 서버는 쓰기 비용이 낮은 자료 구조를 가진 Redo Log를 가지고 있다.</p>
<p>그래서 비정상 종료가 발생하면 Redo Log의 내용을 이용해 데이터 파일을 다시 서버가 종료되기 직전의 상태로 복구한다. (캬)</p>
<blockquote>
<p>참고) 데이터 베이스 서버는 ACID를 포함한 성능도 중요하기 때문에, 데이터 파일 뿐 아니라 Redo Log를 버퍼링 할 수 있는 버퍼 풀이나 Log Buffer와 같은 자료구조도 갖고 있다.</p>
</blockquote>
<p>MySQL 서버가 비정상 종료되는 경우 다음과 같은 일관되지 않은 데이터를 가질 수 있다.</p>
<ol>
<li>커밋됐지만 데이터 파일에 기록되지 않은 데이터</li>
<li><strong>롤백됐지만 데이터 파일에 이미 기록된 데이터</strong></li>
</ol>
<p>1번의 경우 Redo Log에 저장된 데이터를 다시 복사하기만 하면 된다.</p>
<p>2번의 경우 <strong>Redo Log로 해결할 수 없어 Undo Log의 내용을 가져와 데이터 파일에 복사해야 한다.</strong>
<strong>트랜잭션의 상태(커밋, 롤백, 실행 중)를 확인하기 위해 2번의 경우에도 Redo Log는 필요하다.</strong></p>
<hr />
<h3 id="로그-버퍼-log-buffer">로그 버퍼 (Log Buffer)</h3>
<p>변경 작업이 매우 많은 DBMS 서버의 경우 리두 로그의 기록 작업이 성능 저하로 이어질 수 있다. 
이러한 부분을 보완하기 위해 ACID 속성을 보장하는 수준에서 Redo Log 버퍼링을 수행하는데, <strong>이때 사용되는 공간이 로그 버퍼다.</strong></p>
<hr />
<h3 id="리두-로그-아카이빙">리두 로그 아카이빙</h3>
<p>MySQL 서버에 유입되는 데이터 변경이 너무 많으면 Redo Log가 빠르게 증가하고, 새로 추가되는 Redo Log 내용을 복사하기 전에 덮어 쓰일 수 있다. </p>
<p>이렇게 되면 데이터 백업 파일은 일관된 상태를 유지하지 못하고 데이터 백업에 실패하게 된다.</p>
<p>이러한 문제를 해결하기 위해 MySQL 8.0 버전부터는 <strong>Redo Log archiving 기능을 지원하여 데이터 변경이 많아서 Redo Log가 덮어 쓰인다고 해도 백업이 실패하지 않도록 하는 기능이 추가</strong>되었다.</p>
<p>백업 툴이 Redo Log Archiving을 사용하려면 먼저 MySQL 서버에서 아카이빙된 Redo Log가 저장될 디렉터리를 <code>innodb_redo_log_archive_dirs</code> 시스템 변수에 설정해야 한다.</p>
<p>그리고 이 디렉터리는 OS의 MySQL서버를 실행하는 유저만 접근 가능해야 한다.</p>
<hr />
<h3 id="리두-로그-활성화-및-비활성화">리두 로그 활성화 및 비활성화</h3>
<ul>
<li>MySQL 8.0 버전<ul>
<li>리두 로그를 수동으로 활성화하거나 비활성화할 수 있게 됐다. </li>
<li>데이터를 복구하거나 대용량 데이터를 한 번에 적재하는 경우 Redo Log를 비활성화해 데이터 적재 시간을 줄일 수 있다.</li>
</ul>
</li>
</ul>
<pre><code class="language-sql">// 리두 로그 비활성화
ALTER INSTANCE DISABLE INNODB REDO_LOG;

... 대량 데이터 적재 ...

// 리두 로그 활성화
ALTER INSTANCE ENABLE INNODB REDO_LOG;</code></pre>
<hr />
<h2 id="어댑티브-해시-인덱스">어댑티브 해시 인덱스</h2>
<p>일반적인 '인덱스' : 테이블에 사용자가 생성해둔 B-Tree 인덱스</p>
<p>하지만 이번의 <strong>어댑티브 해시 인덱스</strong>는 사용자가 수동으로 생성한 인덱스가 아닌 <strong>InnoDB 스토리지 엔진에서 사용자가 자주 요청하는 데이터에 대해 자동으로 생성하는 인덱스</strong>이다.</p>
<p>B-Tree의 검색 시간을 줄이기 위해 도입된 기능이고, 자주 읽히는 데이터 페이지의 키 값을 이용해 해시 인덱스를 만들고 필요할 때마다 어댑티브 해시 인덱스를 검색해 레코드가 저장된 데이터 페이지를 즉시 찾아갈 수 있게 해 준다.</p>
<p>해시 인덱스는 <strong>인덱스 키 값</strong>과 해당 <strong>인덱스의 키 값이 저장된 데이터 페이지 주소</strong>의 쌍으로 관리된다. </p>
<p>이때 인덱스 키 값은 B-Tree 인덱스의 고유 ID와 B-Tree 인덱스의 실제 키 값의 조합으로 생성된다.</p>
<p><strong>데이터 페이지 주소</strong>는 실제 키 값이 저장된 데이터 페이지의 메모리 주소를 가지는데, 이는 InnoDB 버퍼 풀에 로딩된 페이지 주소를 의미한다.</p>
<h3 id="성능-변화">성능 변화</h3>
<p>어댑티브 해시 인덱스를 사용한다고 해서 무조건 성능이 향상되는 것? -&gt; X</p>
<ul>
<li><p><strong>성능 향상에 크게 도움이 되지 않는 경우</strong></p>
<ul>
<li>디스크 읽기가 많은 경우</li>
<li>특정 패턴의 쿼리가 많은 경우(LIKE 패턴 검색이나 조인)</li>
<li>매우 큰 데이터를 가진 테이블의 레코드를 폭넓게 읽는 경우</li>
</ul>
</li>
<li><p><strong>성능 향상에 도움이 되는 경우</strong></p>
<ul>
<li>디스크의 데이터가 InnoDB 버퍼 풀 크기와 비슷한 경우(디스크 읽기가 많지 않은 경우)</li>
<li>동등 조건 검색(동등 비교와 IN 연산자)이 많은 경우</li>
<li>쿼리가 데이터 중에서 일부 데이터에만 집중되는 경우</li>
</ul>
</li>
</ul>
<p>어댑티브 해시 인덱스 또한 데이터 페이지의 인덱스 키가 해시 인덱스로 만들어져야 하고 불필요한 경우 제거돼야 한다.</p>
<p>어댑티브 해시 인덱스가 활성화되면 InnoDB 스토리지 엔진은 그 키 값이 해시 인덱스에 있든 없든 검색해봐야 한다. </p>
<p>따라서 효율이 없는 해시 인덱스를 InnoDB는 계속 사용하게 된다.</p>
<p>즉, 서비스 패턴을 파악해서 도움이 되는지 불필요한 오버헤드를 발생시키고 있는지 판단해 사용해야 한다.</p>