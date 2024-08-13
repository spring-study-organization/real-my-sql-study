<h1 id="mysql-엔진-아키텍처">MySQL 엔진 아키텍처</h1>
<h2 id="전체-구조">전체 구조</h2>
<p><img alt="" src="https://velog.velcdn.com/images/jojehuni_9759/post/5459afea-e65b-49ec-9300-2172a1982c5f/image.png" /></p>
<p><strong>MySQL 서버 : MySQL 엔진 + 스토리지 엔진</strong> 이라고 생각하고 보자.</p>
<hr />
<h3 id="mysql-엔진">MySQL 엔진</h3>
<ul>
<li>커넥션 핸들러 + SQL Parser 및 전처리기, 쿼리 연산 최적화 (옵티마이저)</li>
</ul>
<blockquote>
<p>커넥션 핸들러 : 클라이언트로부터의 접속 및 쿼리 요청을 처리
MySqL은 대부분의 프로그래밍 언어로부터의 접근은 모두 지원한다.</p>
</blockquote>
<p>즉, DBMS의 두뇌에 해당하는 처리를 수행하는 것이 MySQL 엔진이다.</p>
<hr />
<h3 id="스토리지-엔진-storage-engine">스토리지 엔진 (Storage Engine)</h3>
<p>MySqL에서 처리를 수행한 뒤, 실제 데이터를 디스크 스토리지에 저장하거나 읽어오는 부분이다.</p>
<p>MySQL 엔진은 하나지만, 스토리지 엔진은 여러 개 동시에 사용 가능하다.
그리고 각각의 버퍼, 캐시가 있어서 각각 쿼리 최적화, 데이터 최적화에 사용한다.</p>
<h4 id="핸들러">핸들러</h4>
<p>MySQL 엔진의 쿼리 실행기에서 데이터를 읽거나 쓸 때 각 스토리지 엔진에도 읽기 or 쓰기 요청을 한다.</p>
<blockquote>
<p>이 때 이러한 요청을 <strong>핸들러 요청</strong>이라고 한다.
그리고 그 때 사용하는 API를 <strong>핸들러 API</strong>라고 한다.</p>
</blockquote>
<hr />
<h2 id="mysql-스레딩-구조">MySQL 스레딩 구조</h2>
<p><img alt="" src="https://velog.velcdn.com/images/jojehuni_9759/post/95a06208-647c-467a-9748-a8966863a633/image.png" /></p>
<blockquote>
<p>MySQL 서버는 프로세스 기반 X -&gt; <strong>스레드 기반으로 작동된다.</strong></p>
<ol>
<li>포그라운드 스레드 / 2. 백그라운드 스레드 <strong>2가지로 구분</strong>된다.</li>
</ol>
</blockquote>
<h3 id="포그라운드-스레드">포그라운드 스레드</h3>
<ol>
<li>포그라운드 스레드 = 사용자 스레드 : 각 클라이언트가 요청하는 쿼리 문장 처리</li>
</ol>
<ul>
<li><p>서버에 접속된 클라이언트 수만큼 존재한다.</p>
</li>
<li><p>사용하지 않으면 스레드 캐시로 돌아간다. (커넥션이 종료되면 돌아간다.)</p>
<ul>
<li>이미 일정 개수 이상의 대기 중인 스레드가 있으면 다시 넣지는 않고 종료한다.</li>
</ul>
</li>
<li><p>스레드 캐시에 유지할 수 있는 개수가 한정적 (<code>thread_cache_size</code>로 설정 가능)</p>
</li>
<li><p>버퍼나 캐시로부터 데이터를 가져오고, 없는 경우에는 직접 디스크의 데이터 or 인덱스 파일로부터 읽어와서 처리한다.</p>
</li>
</ul>
<blockquote>
<p>참고</p>
<ul>
<li><code>MyISAM</code> 테이블은 디스크 쓰기 작업까지 포그라운드가 담당</li>
<li><code>InnoDB</code> 테이블은 버퍼나 캐시까지만 포그라운드가 담당, 디스크로 기록하는 작업은 백그라운드가 담당</li>
</ul>
</blockquote>
<hr />
<h3 id="백그라운드-스레드">백그라운드 스레드</h3>
<ol start="2">
<li>백그라운드 스레드 : 주로 InnoDB에서 사용되는 스레드</li>
</ol>
<ul>
<li><p>Insert Buffer 병합하는 스레드</p>
</li>
<li><p>로그를 디스크로 기록 스레드 - <code>로그 스레드 (Log Thread)</code></p>
</li>
<li><p>InnoDB 버퍼풀의 데이터를 디스크에 기록 - <code>쓰기 스레드 (Write Thread)</code></p>
</li>
<li><p>데이터를 버퍼로 읽어오는 스레드</p>
</li>
<li><p>잠금이나 데드락을 모니터링하는 스레드</p>
</li>
</ul>
<p>로그 스레드와 쓰기 스레드가 가장 중요한 역할을 한다.</p>
<blockquote>
<p><strong>데이터 읽기 작업은 절대 지연될 수 없다.</strong>
(사용자가 <code>SELECT</code> 쿼리를 실행했는데 요청된 <code>SELECT</code> 는 10분 뒤에 결과를 돌려주겠다라고 응답을 보내는 <code>DBMS</code>는 없다.)</p>
</blockquote>
<p>그래서 대부분의 <code>DBMS</code>와 <code>InnoDB</code>는 그래서 쓰기 작업을 버퍼링해 일괄 처리해 데이터가 변경될 디스크의 데이터 파일로 저장되는 것까지 기다리지 않아도 된다.</p>
<p>하지만, <code>MyISAM</code>은 포그라운드 스레드가 읽기, 쓰기 둘 다 하기 때문에 쓰기 버퍼링 기능 사용할 수 없음</p>
<hr />
<h2 id="메모리-할당-및-사용-구조">메모리 할당 및 사용 구조</h2>
<p><img alt="" src="https://velog.velcdn.com/images/jojehuni_9759/post/9ab3a005-4bd5-4eb2-b68b-539daf120dc8/image.png" /></p>
<p>크게 글로벌 메모리 영역, 로컬 메모리 영역으로 구분할 수 있다.</p>
<h3 id="글로벌-메모리-영역">글로벌 메모리 영역</h3>
<ol>
<li>글로벌 메모리 영역</li>
</ol>
<ul>
<li>모든 메모리 공간은 MySQL 서버가 시작되면서 OS로부터 할당된다.<ul>
<li>OS 종류마다 달라도 요청된 메모리 공간을 100% 할당 가능하며, 필요할 때 조금씩 할당해주는 경우도 있다.</li>
<li>하지만, 정확한 메모리의 양을 측정하는 것은 어렵다.</li>
</ul>
</li>
<li>클라이언트 스레드 수와 무관하게 하나의 메모리 공간만 할당된다. (<strong>모든 스레드에 의해 공유된다.</strong>)<ul>
<li>단, 필요에 따라 2개 이상의 메모리 공간 할당도 가능.</li>
</ul>
</li>
</ul>
<p><strong>대표적인 글로벌 메모리 영역</strong></p>
<ul>
<li>테이블 캐시 / InnoDB 버퍼 풀 / InnoDB 어댑티드 해시 인덱스 / InnoDB 리두 로그 버퍼</li>
</ul>
<hr />
<h3 id="로컬-메모리-영역">로컬 메모리 영역</h3>
<ol start="2">
<li>로컬 메모리 영역 (= 클라이언트 메모리 영역 = 세션 메모리 영역)</li>
</ol>
<ul>
<li><p>MySQL 서버에 접속하면 서버에서는 클라이언트 커넥션으로부터의 요청을 처리하기 위해 스레드를 하나씩 할당한다. -&gt; 그것이 <strong>클라이언트 스레드</strong></p>
</li>
<li><p>MySQL  서버상 존재하는 클라이언트 스레드가 쿼리를 처리하는데 사용하는 독립적인 메모리 영역</p>
<ul>
<li><strong>절대 공유되어 사용되지 않음</strong></li>
</ul>
</li>
<li><p>조인버퍼나 소트 버퍼의 경우 필요할 때만 공간이 할당됨</p>
</li>
<li><p><strong>커넥션 버퍼</strong> / <strong>소트 버퍼</strong> / 조인 버퍼 / 바이너리 로그 캐시 / 네트워크 버퍼
커넥션 버퍼나 결과 버퍼는 커넥션이 열려있는 동안 계속 할당된 상태로 공간이 남아있음</p>
</li>
</ul>
<hr />
<h2 id="플러그인-스토리지-엔진-모델">플러그인 스토리지 엔진 모델</h2>
<p><img alt="" src="https://velog.velcdn.com/images/jojehuni_9759/post/3cd797ad-4793-4d52-bfdd-aee7cd8cb0eb/image.png" /></p>
<p>특이한 구조 중에 플러그인 모델이 있다.</p>
<p>수많은 사용자의 요구 조건을 만족시키기 위해 기본적으로 제공되는 스토리지 엔진 외에 부가적인 기능을 더 제공하는 스토리지 엔진을 개발하여 플러그인 형태로 제공할 수 있다.</p>
<p>인증이나 전문 검색 파서 또는 쿼리 재작성과 같은 플러그인이 있으며, 비밀번호 검증과 커넥션 제어 등에 관련된 다양한 플러그인이 제공된다.</p>
<p>MySQL 서버의 기능을 커스텀하게 확장하거나 새로운 기능을 플러그인을 이용해 구현할 수도 있다.</p>
<hr />
<h2 id="쿼리-실행-구조">쿼리 실행 구조</h2>
<p><img alt="" src="https://velog.velcdn.com/images/jojehuni_9759/post/f92b3fe2-1c7b-4060-835f-48043de419e9/image.png" /></p>
<ol>
<li>쿼리 파서 : 쿼리 문장을 토큰으로 쪼개는 것 (문법 확인 과정) -&gt; <strong>파스 트리를 만든다.</strong></li>
<li>전처리기 : 만든 파스 트리 속 문제점 확인 -&gt; <strong>객체의 존재 여부, 접근 권한 등 확인</strong></li>
<li>옵티마이저 : 비용 아끼기 위한 <strong>더 나은 실행 계획 도출</strong></li>
<li>쿼리 실행기 : 핸들러에게 요청 후 받은 결과들을 또 다른 핸들러에 요청하며 연결하는 역할</li>
<li>핸들러 : 데이터를 저장, 읽기 하는 역할</li>
</ol>
<hr />
<blockquote>
<p><strong>복제</strong> 라는 개념도 있다. 이건 매우 중요한 역할을 하기 때문에 다음에 자세하게 다뤄보겠다.</p>
</blockquote>
<hr />
<blockquote>
<p><strong>쿼리 캐시</strong>
쿼리 캐시는 빠른 응답을 필요로 하는 웹 기반 응용 프로그램에서 매우 중요한 역할을 담당했지만, 테이블의 데이터가 변경되면 캐시에 저장된 결과에서 해당 테이블과 연관된 모든 것들을 삭제해야 하므로 동시 처리 성능 저하를 유발하는 문제점이 있었다.</p>
</blockquote>
<p><strong>결국 MySQL 8.0에서는 쿼리 캐시에 대한 기능이 완전히 제거되었고 관련된 시스템 변수도 모두 제거됐다.</strong></p>
<blockquote>
<hr />
<p><strong>스레드 풀</strong>
내부적으로 사용자의 요청을 처리하는 스레드 개수를 줄여 MySQL 서버의 CPU가 제한된 개수의 스레드 처리에만 집중할 수 있도록 서버 자원 소모를 줄이는 역할 </p>
<p><strong>그러나</strong> 스레드 그룹 개수와 CPU 코어 개수를 맞추는 것이 CPU 프로세서 친화도를 높이는 데 좋다</p>
<hr />
<p><strong>트랜잭션 지원 메타데이터</strong>
MySQL 서버 5.7 버전까지 테이블 구조를 파일 기반으로 관리했다.
하지만 이러한 파일 기반의 메타데이터는 생성 및 변경 작업에 트랜잭션을 지원하지 않아서 생성이나 변경 도중 MySQL 서버가 비정상 종료가 되면 일관성이 보장되지 않아 테이블이 깨지는 현상이 발생했다.</p>
<p>따라서 MySQL 8.0부터는 테이블 구조 정보나 스토어드 프로그램의 코드 관련 정보 등의 정보를 모두 InnoDB의 테이블에 저장되도록 변경되었다.</p>
</blockquote>
