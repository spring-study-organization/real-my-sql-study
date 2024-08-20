<h1 id="트랜잭션과-잠금">트랜잭션과 잠금</h1>
<p>💡 본 내용은 Real MySQL 8.0 1권을 읽으면서 정리한 내용입니다.</p>
<ul>
<li>본문에 포함된 사진의 출처는 Real MySQL 5장입니다.</li>
</ul>
<h2 id="트랜잭션">트랜잭션</h2>
<p>MySQL의 동시성에 영향을 미치는</p>
<ul>
<li>잠금</li>
<li>트랜잭션</li>
<li>트랜잭션의 격리 수준 (Isolation level)</li>
</ul>
<p>이것들을 알아보자.</p>
<blockquote>
<p><strong>트랜잭션</strong> : 논리적인 작업 SET을 완벽하게 처리하거나, 처리하지 못 했을 경우에는 원 상태로 복구해서 작업의 일부만 적용되는 현상을 발생하지 않게 해주는 기능</p>
</blockquote>
<hr />
<p>잠금과 동시성은 비슷해보인다. 하지만 둘은 다르다.</p>
<ul>
<li>잠금 : 동시성을 제어하기 위한 기능</li>
<li>트랜잭션 : 데이터의 정합성(어떤 데이터들이 값이 서로 일치하는 상태) 을 보장하기 위한 기능</li>
</ul>
<p>잠금에 대해 좀 더 자세하게 이야기 한다면</p>
<p>하나의 레코드를 여러 커넥션에서 동시에 변경하려고 할 때 <strong>잠금이 없다면</strong>, 하나의 데이터를 여러 커넥션에서 동시에 변경할 수 있게 된다..</p>
<p>-&gt; 즉, <strong>해당 레코드의 값은 예측할 수 없는 상태가 된다.</strong></p>
<blockquote>
<p><strong>잠금</strong> : 여러 커넥션에서 동시에 동일한 자원을 요청할 경우 순서대로 한 시점에는 하나의 커넥션만 변경할 수 있게 해주는 역할</p>
</blockquote>
<p>그리고 트랜잭션의 격리 수준에서 격리 수준이란?</p>
<blockquote>
<p><strong>격리 수준</strong> : 하나 or 여러 개의 트랜잭션 간의 작업 내용을 어떻게 공유하고 차단할 것인지 결정하는 레벨을 의미한다.</p>
</blockquote>
<hr />
<h3 id="트랜잭션의-관점에서-innodb와-myisam-테이블의-차이">트랜잭션의 관점에서 InnoDB와 MyISAM 테이블의 차이</h3>
<p>둘 다 3이라는 값을 테스트용 테이블에 INSERT 한 뒤, </p>
<pre><code class="language-sql">mysql&gt; INSERT INTO tab_myisam (fdpk) VALUES (3);

mysql&gt; INSERT INTO tab_innodb (fdpk) VALUES (3);</code></pre>
<p><code>Auto-commit</code> 모드에서 1,2,3 이라는 값을 INSERT 하는 쿼리 문장을 실행한다고 했을 때</p>
<p>아래와 같은 결과를 보여준다.</p>
<ul>
<li><p>MyISAM</p>
<ul>
<li>1, 2, 3</li>
</ul>
</li>
<li><p>InnoDB</p>
<ul>
<li>3</li>
</ul>
</li>
</ul>
<p><strong>둘 다 Primary Key 중복 오류로 쿼리는 실패한다.</strong></p>
<p>근데 레코드를 조회해보면 오류 발생을 했음에도 MyISAM은 <strong>부분 업데이트</strong>를 하고,
InnoDB는 데이터의 정합성을 지키는 트랜잭션의 원칙을 지켰다.</p>
<p>MyISAM 같은 경우, 부분 업데이트 현상으로 인해 실패 쿼리의 경우 다시 레코드를 삭제한 뒤 재처리 작업이 필요하다.</p>
<p>1개뿐이라면 간단하지만, 2개 이상부터는 고민거리가 될 것이다..</p>
<hr />
<h3 id="주의사항">주의사항</h3>
<p>트랜잭션은 DBMS의 커넥션과 동일하게 꼭 필요한 최소의 코드에만 적용하는 것이 좋다.</p>
<p>-&gt; 프로그램 코드에서 트랜잭션의 범위를 최소화하라는 의미이다.</p>
<p>간단한 예시들을 보자. 
사용자가 게시판에 게시물을 작성한 후 저장 버튼을 눌렀을 때 서버에서 처리하는 내용.</p>
<blockquote>
</blockquote>
<ol>
<li>처리 시작 <ul>
<li>데이터베이스 커넥션 생성</li>
<li>트랜잭션 시작</li>
</ul>
</li>
<li>사용자의 로그인 여부 확인</li>
<li>사용자의 글쓰기 내용의 오류 여부 확인</li>
<li>첨부로 업로드된 파일 확인 및 저장</li>
<li>사용자의 입력 내용을 DBMS에 저장</li>
<li>첨부 파일 정보를 DBMS에 저장</li>
<li>저장된 내용 or 기타 내용을 DBMS에서 조회</li>
<li>게시물 등록에 대한 알림 메일 발송</li>
<li>알림 메일 발송 이력을 DBMS에 저장<ul>
<li>트랜잭션 종료</li>
<li>데이터베이스 커넥션 반납</li>
</ul>
</li>
<li>처리 완료</li>
</ol>
<p>많은 개발자가 1, 2번 사이에 커넥션을 생성하면서 트랜잭션을 시작하고, 
9, 10번 사이에 트랜잭션을 Commit하고 커넥션을 종료한다.</p>
<p>실제로 DBMS에 데이터를 저장하는 작업인 트랜잭션은 5번부터인데 말이다.
-&gt; 2,3,4번은 트랜잭션에 포함시킬 필요가 없다.</p>
<p>일반적으로 DB 커넥션은 개수가 제한적이고, 각 단위 프로그램이 커넥션을 소유하는 시간이 길어질 수록?
=&gt; 사용할 수 있는 커넥션의 개수는 줄어든다..</p>
<p><strong>그렇게 된다면 각 단위 프로그램에서 커넥션을 가져가려고 기다려야 할 수도 있다.</strong></p>
<hr />
<p>다음으로는 <strong>메일 전송과 같은 네트워크를 통해 원격 서버에 통신하는 작업</strong> (예시 중 8번) 들은 <strong>트랜잭션에서 제외하는 것이 좋다.</strong></p>
<p>프로그램이 실행하는 동안 <strong>메일 서버와 통신할 수 없는 상황이 발생할 시</strong>, 웹 서버뿐 아닌 <strong>DBMS 서버까지 위험해질 수도 있다.</strong></p>
<p>DBMS의 작업은 예시에서 4개가 있는데, 사용자의 입력 내용을 저장하는 5, 6번은 필수로 하나의 트랜잭션이어야 한다.</p>
<p>그리고 7번은 단순 조회이기 때문에 굳이 트랜잭션에 넣지 않아도 되며, 9번 또한 5,6번 작업과 같이 묶지 않아도 괜찮다.</p>
<hr />
<p>가능하면 아래와 같이 트랜잭션을 적용하는 것이 좋다.</p>
<blockquote>
</blockquote>
<ol>
<li>처리 시작 </li>
<li>사용자의 로그인 여부 확인</li>
<li>사용자의 글쓰기 내용의 오류 여부 확인</li>
<li>첨부로 업로드된 파일 확인 및 저장<ul>
<li><strong>데이터베이스 커넥션 생성</strong></li>
<li><strong>트랜잭션 시작</strong></li>
</ul>
</li>
<li>사용자의 입력 내용을 DBMS에 저장</li>
<li>첨부 파일 정보를 DBMS에 저장<ul>
<li><strong>트랜잭션 종료</strong></li>
</ul>
</li>
<li>저장된 내용 or 기타 내용을 DBMS에서 조회</li>
<li>게시물 등록에 대한 알림 메일 발송<ul>
<li><strong>트랜잭션 시작</strong></li>
</ul>
</li>
<li>알림 메일 발송 이력을 DBMS에 저장<ul>
<li><strong>트랜잭션 종료</strong></li>
<li><strong>데이터베이스 커넥션 반납</strong></li>
</ul>
</li>
<li>처리 완료</li>
</ol>
<hr />
<h2 id="mysql-엔진의-잠금">MySQL 엔진의 잠금</h2>
<p>MySQL 엔진에서의 잠금은</p>
<ul>
<li>스토리지 엔진 레벨</li>
<li>MYSQL 엔진 레벨</li>
</ul>
<p>2가지로 나눌 수 있다.</p>
<p>여기서 MySQL 엔진은 MySQL 서버에서 스토리지 엔진을 제외한 나머지 부분으로 이해하면 된다.</p>
<ul>
<li><p>MySQL 엔진 레벨의 잠금은 스토리지 엔진에 영향을 끼친다.</p>
</li>
<li><p>스토리지 엔진 레벨의 잠금은 스토리지 엔진 간 상호 영향을 미치지는 않는다.</p>
</li>
</ul>
<hr />
<p>MySQL 엔진에서는 여러 잠금 기능이 존재한다.</p>
<ul>
<li>테이블 데이터 동기화를 위한 Table Lock</li>
<li>테이블 구조를 잠그는 메타데이터 락 (Metadata Lock)</li>
<li>사용자의 필요에 맞게 사용할 수 있는 네임드 락 (Named Lock)</li>
</ul>
<p>이러한 잠금 기능들이 존재하는데 어떤 경우에 사용되는지 알아보자.</p>
<hr />
<h3 id="글로벌-락">글로벌 락</h3>
<p>MySQL에서 제공하는 잠금 중 가장 범위가 크며</p>
<pre><code class="language-SQL">FLUSH TABLES WITH READ LOCK</code></pre>
<p>위와 같은 명령으로 <strong>글로벌 락</strong>을 획득할 수 있다.</p>
<p>영향을 미치는 범위는 MySQL 서버 전체이다.</p>
<p>한 세션에서 글로벌 락을 획득하면 다른 세션에서 SELECT 를 제외한 DDL, DML 문장을 실행하는 경우에 글로벌 락이 해제될 때까지 해당 문장이 대기 상태로 남는다.</p>
<p>작업 대상 테이블이나 데이터베이스가 달라도 동일하게 영향을 미친다.</p>
<blockquote>
<p>즉, 여러 DB에 존재하는 MyISAM이나 MEMORY 테이블에 대해 <code>mysqldump</code> 로 일관된 백업을 받아야 할 때 사용해야 한다.</p>
</blockquote>
<p>하지만 MySQL 서버가 업그레이드 되면서 MyISAM이나 MEMORY 스토리지 엔진 보다는 InnoDB 스토리지 엔진의 사용이 일반화됐다.</p>
<p>InnoDB 스토리지 엔진은 트랜잭션을 지원하기 때문에 일관된 데이터 상태를 위해 모든 데이터 변경 작업을 멈출 필요 없다.</p>
<h3 id="백업-락">백업 락</h3>
<p>또, MySQL 8.0 부터는 InnoDB가 기본 스토리지 엔진으로 채택되면서 조금 더 가벼운 글로벌 락의 필요성이 생겼으며 그래서 백업 락이 도입 (ex. XtraBackup, Enterprise Backup) 됐다.</p>
<pre><code class="language-SQL">mysql&gt; LOCK INSTANCE FOR BACKUP;
-- // 백업 실행
mysql&gt; UNLOCK INSTANCE;</code></pre>
<p><strong>특정 세션에서 백업 락을 획득하면</strong> 아래와 같은 테이블의 스키마, 사용자의 인증 관련 정보는 변경 불가능</p>
<ul>
<li>데이터베이스 및 테이블 등 모든 객체 생성 및 변경, 삭제</li>
<li>REPAIR TABLE 과 OPTIMIZE TABLE 변경</li>
<li>사용자 관리 및 비밀번호 변경</li>
</ul>
<p><strong>하지만</strong>, 백업 락은 일반적인 테이블의 데이터 변경은 허용된다.</p>
<p>일반적인 MySQL 서버의 구성은 <strong>소스 서버</strong>와 <strong>레플리카 서버</strong>로 구성되는데, 주로 백업은 레플리카 서버에서 실행되기 때문이다.</p>
<p>그럼에도 <code>FLUSH TABLES WITH READ LOCK</code> 명령을 통해 글로벌 락을 획득하면 복제는 백업 시간만큼 지연된다..</p>
<p>즉, 레플리카 서버에서 백업을 실행하는 도중 소스 서버에 문제가 생기면 레플리카 서버의 데이터가 최신 상태가 될 때까지 서비스를 멈춰야 할 수도 있다.</p>
<p>그래서 MySQL 8.0 버전의 XtraBackup, Enterprise Backup 툴들은 복제가 진행되는 상태에서도 일관된 백업을 만들 수 있다.</p>
<p><strong>이 때도 백업 중 스키마 변경이 실행되는 백업은 실패한다.</strong></p>
<hr />
<h3 id="테이블-락">테이블 락</h3>
<blockquote>
<p><strong>테이블 락</strong> : 개별 테이블 단위로 설정되는 잠금</p>
</blockquote>
<p>명시적 or 묵시적으로 특정 테이블의 락을 획득할 수 있다.</p>
<ul>
<li><strong>명시적 방법</strong><pre><code class="language-sql">LOCK TABLES table_name [ READ | WRITE ]</code></pre>
위의 명령을 사용하면 된다. (InnoDB, MyISAM 둘다 동일)</li>
</ul>
<p>잠금 해제 명령어 : <code>UNLOCK TABLES</code></p>
<ul>
<li><strong>묵시적 방법</strong>
MyISAM 이나 MEMORY 테이블에 데이터를 변경하는 쿼리를 실행하면 발생</li>
</ul>
<p>즉, 쿼리가 실행되는 동안 자동으로 획득됐다가 완료된 후 자동 해체</p>
<blockquote>
<p>InnoDB 테이블의 경우 : 스토리지 엔진 차원에서 레코드 기반의 잠금을 제공해서 묵시적인 테이블 락이 설정되지 않는다.</p>
</blockquote>
<hr />
<h3 id="네임드-락">네임드 락</h3>
<blockquote>
<p><strong>네임드 락</strong> : 단순히 사용자가 지정한 문자열(String)에 대해 획득하고 반납하는 잠금</p>
</blockquote>
<p><code>GET_LOCK()</code> 함수를 이용해 임의의 문자열에 대해 잠금 설정이 가능하다.</p>
<p><strong>특징</strong></p>
<ul>
<li>대상이 테이블이나 레코드 또는 <code>AUTO_INCREMENT</code>와 같은 데이터베이스 객체가 아니다.</li>
<li>많은 레코드에 대해서 복잡한 요건으로 레코드를 변경하는 트랜잭션에 유용하게 사용 가능하다.</li>
</ul>
<p>2번째 특징에 대해 더 이야기하면, 배치 프로그램처럼 많은 레코드를 변경하는 쿼리는 데드락의 원인이 자주 되곤 한다.</p>
<p>데드락을 최소화하는 방법으로는 프로그램의 실행 시간을 분산 or 프로그램의 코드 수정이지만, 완전한 해결책 X + 간단한 방법 X</p>
<p>-&gt; 이러한 경우 동일 데이터를 변경하거나 참조하는 프로그램끼리 분류해 네임드 락을 걸고 쿼리를 실행하면 간단히 해결 가능하다.</p>
<p>8.0 버전부터는 중첩해서 사용도 가능하다.</p>
<hr />
<h3 id="메타데이터-락">메타데이터 락</h3>
<blockquote>
<p><strong>메타데이터 락</strong> : 데이터베이스 객체 (ex. 테이블, 뷰) 의 이름이나 구조를 변경하는 경우에 획득하는 락</p>
</blockquote>
<p>명시적으로 획득하는 락이 아닌, 자동으로 획득하는 락이다.</p>
<hr />
<h2 id="innodb-스토리지-엔진의-잠금">InnoDB 스토리지 엔진의 잠금</h2>
<p>MySQL에서 제공하는 잠금과는 별개로 스토리지 엔진 내부에서 레코드 기반의 잠금 방식을 탑재하고 있다.</p>
<p>그래서 MyISAM보다 훨씬 뛰어난 동시성 처리가 가능하지만, MySQL 명령을 이용해 사용되는 잠금에 대한 정보에 접근이 어렵다.</p>
<p>예전에는 <code>lock_monitor</code>와 <code>SHOW ENGINE INNODB STATUS</code> 의 방법이 전부였다.</p>
<p>최근에는 iNNOdb의 트랜잭션과 잠금, 잠금 대기 중인 트랜잭션의 목록을 조회할 수 있는 방법도 도입됐다.</p>
<ul>
<li>MySQL 서버의 <code>information-schema</code> DB에 존재하는 <code>INNODB_TRX</code>, <code>INNODB_LOCKS</code>, <code>INNODB_LOCK_WAITS</code> 라는 테이블을 조인해서 조회하면</li>
</ul>
<ol>
<li>어떤 트랜잭션이 어떤 잠금을 대기하는지 찾기</li>
<li>해당 잠금을 어느 트랜잭션이 가지고 있는지 찾기</li>
<li>장시간 잠금을 가지고 있는 클라이언트를 찾아서 종료시키기</li>
</ol>
<p>위와 같은 것들이 가능하다.</p>
<p>그리고 더 업그레이드 되면서는 <code>Performance Schema</code>를 이용해서 내부 잠금에 대한 모니터링 방법도 추가됐다.</p>
<hr />
<p>InnoDB 스토리지 엔진은 잠금 정보가 상당히 작은 공간으로 관리되기 때문에 레코드 락이 페이지 락으로, 또는 테이블 락으로 레벨업 되는 경우(락 에스컬레이션)는 없다.</p>
<p>일반 사용 DBMS와는 조금 다르게 InnoDB 스토리지 엔진에서는 레코드 락뿐 아니라 레코드와 레코드 사이의 간격을 잠그는 갭(GAP) 락이라는 것이 존재한다.</p>
<p>아래에 종류에 대한 내용이 있다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/jojehuni_9759/post/e966ef37-118c-4da0-af2f-4b039fa7668e/image.png" /></p>
<p>(점선 레코드는 실제 존재하지 않는 레코드를 가정한 것)</p>
<hr />
<h3 id="레코드-락">레코드 락</h3>
<blockquote>
<p><strong>레코드 락</strong> : 레코드 자체만을 잠그는 것. </p>
</blockquote>
<p>다른 상용 DBMS의 레코드 락과 동일한 역할을 한다.</p>
<p>하지만, 중요한 차이점이 있는데 <strong>InnoDB 스토리지 엔진은 레코드 자체가 아니라 인덱스의 레코드를 잠근다는 점</strong>이다.</p>
<p>인덱스가 하나도 없는 테이블이더라도 내부적으로 자동 생성된 클러스터 인덱스를 이용해 잠금을 설정한다.</p>
<p>대부분 보조 인덱스를 이용한 변경 작업은 넥스트 키 락(Next Key Lock) 또는 갭(Gap Lock)을 사용하지만, <strong>PK 또는 Unique Index에 의한 변경 작업에서는 gap에 대해서는 잠그지 않고</strong> 레코드 자체에 대해서만 락을 건다.</p>
<hr />
<h3 id="갭-락-gap-lock">갭 락 (Gap Lock)</h3>
<blockquote>
<p><strong>갭 락</strong> : 레코드 자체가 아니라 레코드와 <strong>바로 인접한 레코드 사이의 간격만을 잠그는 것을 의미</strong>한다.</p>
</blockquote>
<p><strong>레코드와 레코드 사이의 간격에 새로운 레코드가 생성(INSERT)되는 것을 제어하는 것</strong>이며, <strong>넥스트 키 락의 일부로 자주 사용</strong>된다.</p>
<hr />
<h3 id="넥스트-키-락-next-key-lock">넥스트 키 락 (Next Key Lock)</h3>
<blockquote>
<p><strong>넥스트 키 락</strong> : 레코드 락과 갭 락을 합쳐 놓은 형태의 잠금</p>
</blockquote>
<p><code>STATEMENT</code> 포맷의 바이너리 로그를 사용하는 MySQL(8.0 이전 버전) 서버에서는 <code>REPEATABLE READ</code> 격리 수준을 사용해야 한다. 그리고 <code>innodb_locks_unsafe_for_binlog</code> 시스템 변수가 비활성화되면(0으로 설정되면) 변경을 위해 검색하는 레코드에는 넥스트 키 락 방식으로 잠금이 걸린다.</p>
<h4 id="innodb의-갭-락이나-넥스트-키-락">InnoDB의 갭 락이나 넥스트 키 락</h4>
<p>바이너리 로그에 기록되는 쿼리가 레플리카 서버에서 실행될 때 소스 서버에서 만들어 낸 결과와 동일한 결과를 만들어내도록 보장하는 것이 주목적이다.</p>
<p>근데 둘 다 의외로 데드락이 발생하거나 다른 트랜잭션을 기다리게 하는 일이 자주 발생한다.</p>
<p>그래서 바이너리 로그 포맷을 ROW 형태로 바꿔서 넥스트 키 락, 갭락을 줄이는 것이 좋다.</p>
<blockquote>
<p><strong>참고</strong>
MySQL 8.0 버전이 되면서 ROW 포맷의 바이너리 로그에 대한 안정성이 올라갔다.
그래서 <code>STATEMENT</code> 포맷의 바이너리 로그가 가지는 단점을 많이 해결했고, 8.0에서는 ROW 포맷의 바이너리 로그가 기본값이 되었다.</p>
</blockquote>
<hr />
<h3 id="자동-증가-락">자동 증가 락</h3>
<p>MySQL에서는 자동 증가하는 숫자 값을 추출(채번)하기 위해 <code>AUTO_INCREMENT</code>라는 칼럼 속성을 제공한다.</p>
<p><code>AUTO_INCREMENT</code> 칼럼이 사용된 테이블에 동시에 여러 레코드가 INSERT되는 경우, 저장되는 각 레코드는 중복되지 않고 저장된 순서대로 증가하는 일련번호 값을 가져야 하기 때문에 내부적으로 <code>AUTO_INCREMENT</code> 락이라고 하는 테이블 수준의 잠금을 사용한다.</p>
<p><code>AUTO_INCREMENT</code> 락은 INSERT와 REPLACE 쿼리 문장과 같이 새로운 레코드를 저장하는 쿼리에서만 필요하며, UPDATE나 DELETE 등의 쿼리에서는 걸리지 않는다.</p>
<p><strong>INSERT나 REPLACE 문장에서 <code>AUTO_INCREMENT</code> 값을 가져오는 순간만 락이 걸렸다가 즉시 해제된다.</strong></p>
<p><code>AUTO_INCREMENT</code> 락은 테이블에 단 하나만 존재하기 때문에 두 개의 INSERT 쿼리가 동시에 실행되는 경우 하나의 쿼리가 <code>AUTO_INCREMENT</code> 락을 걸면 나머지 쿼리는 <code>AUTO_INCREMENT</code> 락을 기다려야 한다.</p>
<p><code>AUTO_INCREMENT</code> 컬럼에 명시적으로 값을 설정하더라도 자동 증가 락을 걸게 된다.</p>
<hr />
<h2 id="인덱스와-잠금">인덱스와 잠금</h2>
<p>InnoDB의 잠금은 레코드를 잠그는 것이 아니라 인덱스를 잠그는 방식으로 처리된다.</p>
<blockquote>
<p>즉, 변경해야 할 레코드를 찾기 위해 검색한 인덱스의 레코드를 모두 락을 걸어야 한다.</p>
</blockquote>
<p>아래 예제가 있다. (Real MySQL 8.0에 있는 예제)</p>
<pre><code class="language-sql">-- // 예제 데이터베이스의 employees 테이블에는 first_name 칼럼만
-- // 멤버로 담긴 ix_firstname이라는 인덱스가 준비돼 있다.
-- // KEY ix_firstname(first_name)
-- // employess 테이블에서 first_name='Georgi'인 사원은 전체 253명이 있으며,
-- // first_name='Georgi'이고 last_name='Klssen'인 사원은 딱 1명만 있는 것을 아래 쿼리로 확인할 수 있다.
mysql&gt; SELECT COUNT(*) FROM employess WHERE first_name='Georgi';
+--------+
 |      253  |
+--------+

mysql&gt; SELECT COUNT(*) FROM employess WHERE first_name='Georgi' AND last_name='Klssen';
+--------+
 |           1  |
+--------+

-- // employees 테이블에서 first_name='Georgi'이고, last_name=&quot;Klassen'인 사원의
-- // 입사 일자를 오늘로 변경하는 쿼리를 실행해보자.
mysql&gt; UPDATE employess SET hire_date=NOW() WHERE first_name='Georgi' AND last_name='Klassen';</code></pre>
<p>이러면 UPDATE 문장이 실행됐을 때 인덱스로 사용하는건 first_name='Georgi'이고, last_name column이 인덱스에 없기 때문에 first_name=’Georgi’인 레코드 253건의 레코드가 모두 잠긴다.</p>
<p>위 예제만 돼도 약 250건 가량밖에 안 되지만 더욱 많아진 상황에서 UPDATE 문장을 위해 적절히 인덱스가 준비돼 있지 않다면, 각 클라이언트 간의 동시성이 상당히 떨어져서 한 세션에서 UPDATE 작업을 하는 중에는 다른 클라이언트는 그 테이블을 업데이트하지 못하고 기다려야 하는 상황이 발생할 것이다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/jojehuni_9759/post/883b3ce1-5686-4203-8990-f97a875b35f2/image.png" /></p>
<p>위 사진은 UPDATE 문장이 어떻게 변경할 레코드를 찾고 수행되는지를 보여준다.</p>
<p><strong>만약 위 테이블에서 인덱스가 하나도 없다면 테이블을 풀 스캔하면서 UPDATE 작업을 하는데, 이 과정에서 테이블의 모든 레코드를 잠그게 된다.</strong></p>
<p>이것이 MySQL의 방식이며, MySQL의 InnoDB에서 인덱스 설계가 중요한 이유이다.</p>
<hr />
<h2 id="레코드-수준의-잠금-확인-및-해제">레코드 수준의 잠금 확인 및 해제</h2>
<p>InnoDB 스토리지 엔진을 사용하는 테이블의 레코드 수준의 잠금은 테이블 수준 잠금보다 조금 더 복잡하고 문제의 원인을 발견하고 해결하기도 어렵다.</p>
<p>MySQL 5.1 버전부터는 레코드 잠금과 잠금 대기에 대한 조회가 가능하다.</p>
<p>그래서 쿼리 하나로 잠금과 잠금 대기를 바로 확인할 수 있다.</p>
<pre><code class="language-sql">// 명령이 실행된 상태의 프로세스 목록을 조회
SHOW PROCESSLIST;</code></pre>
<hr />
<pre><code class="language-sql">// performance_schema의 data_locks 테이블과 data_lock_waits 테이블을 조인해 
// 잠금 대기 순서 조회

SELECT
    r.trx_id waiting_trx_id,
    r.trx_mysql_thread_id waiting_thread, 
    r.trx_query waiting_query,
    b.trx_id blocking_trx_id, 
    b.trx_mysql_thread_id blocking_thread, 
    b.trx_query blocking_query
FROM performance_schema.data_lock_waits w 
    INNER OOIN information_schema.innodb_trx b
        ON b.trx_id = w.blocking_engine_transaction_id 
    INNER JOIN information_schema.innodb_trx r
        ON r.trx_id = w.requesting_engine_transaction_id;</code></pre>
<hr />
<p>만약 특정 스레드가 어떤 잠금을 가지고 있는지 더 상세히 확인하고 싶다면 performance_schema의 data_locks 테이블이 가진 컬럼을 모두 살펴보면 된다.</p>
<pre><code class="language-sql">SELECT * FROM performance_schema.data_locks\G</code></pre>
<p>스레드를 강제 종료하기 위해서는 <code>KILL {특정 스레드 번호}</code>를 실행하면 된다.</p>
<pre><code class="language-sql">// 예시
KILL 17</code></pre>
<hr />
<h2 id="mysql의-격리-수준">MySQL의 격리 수준</h2>
<blockquote>
<p><strong>격리 수준</strong> : 여러 트랜잭션이 동시에 처리될 때 특정 트랜잭션이 다른 트랜잭션에서 변경하거나 조회하는 데이터를 볼 수 있게 허용할지를 결정하는 것이다.</p>
</blockquote>
<p>격리 수준은 크게 <code>READ UNCOMMITED</code>, <code>READ COMMITED</code>, <code>REPEATABLE READ</code>, <code>SERIALIZABLE</code> 4가지로 나뉜다.</p>
<p><code>READ UNCOMMITED</code> 중에는 <code>DIRTY READ</code> 라고 하는 격리 수준도 있는데, 일반적인 데이터베이스에서는 거의 사용하지 않는다.</p>
<ul>
<li><code>SERIALIZABLE</code> 또한 <strong>동시성이 중요한 데이터베이스에서는 거의 사용되지 않는다.</strong></li>
</ul>
<p>4개의 격리 수준에서 순서대로 뒤로 갈수록 각 트랜잭션 간의 데이터 격리(고립)정도가 높아지며, 동시 처리 성능도 떨어지게 된다. (그래도 <code>SERIALIZABLE</code> 만 아니면 크게 차이는 나지 않는다.)</p>
<table>
<thead>
<tr>
<th></th>
<th>DIRTY READ</th>
<th>NON-REPEATED READ</th>
<th>PHANTOM READ</th>
<th>비고</th>
</tr>
</thead>
<tbody><tr>
<td>READ UNCOMMITTED</td>
<td>발생</td>
<td>발생</td>
<td>발생</td>
<td>거의 사용하지 않음</td>
</tr>
<tr>
<td>READ COMMTTIED</td>
<td>없음</td>
<td>발생</td>
<td>발생</td>
<td>오라클,PostgreSQL(기본)</td>
</tr>
<tr>
<td>REPEATABLE READ</td>
<td>없음</td>
<td>없음</td>
<td>발생</td>
<td>(innoDB는 없음)    innoDB(기본)</td>
</tr>
<tr>
<td>SERIALIZABLE</td>
<td>없음</td>
<td>없음</td>
<td>없음</td>
<td>거의 사용하지 않음</td>
</tr>
</tbody></table>
<p><code>SQL-92</code> 또는 <code>SQL-99</code> 표준에 따르면</p>
<p><code>REPEATABLE READ</code> 격리 수준에서는 <code>PHANTOM READ</code>가 발생할 수 있지만, <strong>InnoDB에서는 독특한 특성 때문에 <code>REPEATABLE READ</code> 격리 수준에서도 <code>PHANTOM READ</code>가 발생하지 않는다.</strong></p>
<p>일반적인 온라인 서비스 용도의 데이터베이스는 <code>READ COMMITED</code>와 <code>REPEATABLE READ</code> 중 하나를 사용한다.</p>
<p>오라클 같은 DBMS에서는 주로 READ COMMITED 수준을 많이 사용하며, MySQL에서는 REPEATABLE READ를 주로 사용한다.</p>
<hr />
<h3 id="read-uncommited">READ UNCOMMITED</h3>
<blockquote>
<p>어떤 트랜잭션의 작업이 완료되지 않았는데도, 다른 트랜잭션에서 볼 수 있는 데이터 불일치 현상</p>
</blockquote>
<p>READ UNCOMMITED 격리 수준에서는 아래 이미지와 같이 각 트랜잭션의 변경 내용이 COMMIT이나 ROLLBACK 여부와 상관없이 다른 트랜잭션에서 보인다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/jojehuni_9759/post/b05dd678-1842-49b2-906a-7d34e2b6a820/image.png" /></p>
<p>사진을 보자.</p>
<ul>
<li><p><strong>사용자 A</strong>는 emp_no가 500000 이고 first_name이 “Lara”인 새로운 사원을 INSERT한다.</p>
</li>
<li><p><strong>사용자 B</strong>가 emp_no=500000 인 사원을 검색하는데, 사용자 A가 INSERT한 사원의 정보를 커밋되지않은 상태에서도 조회할 수 있다.</p>
</li>
</ul>
<p>여기서 문제는 사용자 A가 처리 도중 알 수 없는 문제가 발생해 <strong>INSERT된 내용을 롤백한다고 하더라도 사용자 B는 “Lara”가 정상적인 사원이라고 생각하고 계속 처리할 것이라는 점</strong>이다.</p>
<p><strong>어떤 트랜잭션에서 처리한 작업이 완료되지 않았는데도 다른 트랜잭션에서 볼 수 있는 현상을 더티 리드(DIRTY READ)</strong>라 하고, <strong>더티 리드가 허용되는 격리 수준이 READ UNCOMMITED</strong>이다.</p>
<p>더티 리드를 유발하는 READ UNCOMMITED는 RDBMS 표준에서는 트랜잭션의 격리 수준으로 인정하지 않을 정도로 정합성에 문제가 많은 격리 수준이므로, MySQL을 사용한다면 최소한 READ COMMITED 이상의 격리 수준을 사용할 것을 권장한다.</p>
<hr />
<h3 id="read-commited">READ COMMITED</h3>
<p><strong>READ COMMITED</strong>는 오라클 DBMS에서 기본으로 사용되는 격리 수준이며, 온라인 서비스에서 가장 많이 선택되는 격리 수준이다.</p>
<p>이 레벨에서는 위에서 언급한 <strong>더티 리드같은 현상은 발생하지 않는다.</strong></p>
<p><strong>어떤 트랜잭션에서 데이터를 변경했더라도 COMMIT이 완료된 데이터만 다른 트랜잭션에서 조회할 수 있기 때문이다.</strong></p>
<h4 id="read-commited-동작-방식">READ COMMITED 동작 방식</h4>
<p><img alt="" src="https://velog.velcdn.com/images/jojehuni_9759/post/92993cbc-9146-48b6-9599-1daa603894ed/image.png" /></p>
<p>READ-COMMITTED 격리 수준에서부터 언두 로그가 등장한다.
데이터 변경 발생 시 언두 로그 영역에 이전 레코드 정보가 백업된다.</p>
<p>사용자 A가 데이터를 변경한 후, 사용자 B가 해당 컬럼을 조회하면 <strong>언두 로그에 백업된 레코드를 조회하게 된다.</strong></p>
<blockquote>
<p>따라서 <strong>READ-COMMITTED 격리 수준에서는 어떤 트랜잭션에서 변경한 내용이 커밋되기 전까지는 다른 트랜잭션에서 그러한 변경 내역을 조회할 수 없다.</strong></p>
</blockquote>
<hr />
<h3 id="read-committed-격리-수준의-문제점">READ-COMMITTED 격리 수준의 문제점</h3>
<p><code>READ-COMMITTED</code> 격리 수준에서도 <code>NON-REPEATABLE READ</code> 라는 부정합의 문제가 존재한다. </p>
<p>레코드가 반복되어 읽어지는 상황에서 발생하는 데이터 부정합 문제이다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/jojehuni_9759/post/6732cb2b-fd68-4596-9952-559c57c69faf/image.png" /></p>
<p>중간에 다른 트랜잭션에서 커밋한 데이터 때문에 한 트랜잭션 안에서 같은 SELECT문의 실행 결과가 달라지는 문제가 발생</p>
<p>이는 별다른 문제가 없어 보이지만, 사실 사용자 B가 하나의 트랜잭션 내에서 똑같은 SELECT 쿼리를 실행했을 때는 항상 같은 결과를 가져와야 한다는 <strong>REPEATABLE READ 정합성</strong>에 어긋나는 것이다.</p>
<ul>
<li>일반적으로는 크게 문제가 되지 않을 수 있지만, 금전적인 내용을 다루는 경우 주의해야 한다.<ul>
<li>오늘 입금된 금액의 총합을 조회하는 상황에서 NON-REPEATABLE READ가 발생할 경우, 총합을 계산하는 SELECT 쿼리가 실행될 때 마다 다른 결과를 가져올 수 있기 때문이다.</li>
</ul>
</li>
</ul>
<hr />
<h3 id="repeatable-read">REPEATABLE READ</h3>
<p>InnoDB 스토리지 엔진에서 기본으로 사용되는 격리 수준으로, 바이너리 로그를 가진 MySQL 서버에서는 최소 REPEATABLE READ 격리 수준 이상을 사용해야 한다.</p>
<p>이 격리 수준에서는 <strong>READ COMMITED 격리 수준에서 발생하는 NON-REPEATABLE READ 부정합이 발생하지 않는다.</strong></p>
<p><strong>InnoDB 스토리지 엔진은 트랜잭션이 <code>ROLLBACK</code> 될 가능성에 대비해 변경 전 레코드를 언두(Undo) 공간에 백업해두고 실제 레코드 값을 변경</strong>하는데, 이러한 변경 방식을 <strong>MVCC</strong>라고 한다. (이전 게시글에서도 다룬 것이다.)</p>
<p><code>REPEATABLE READ</code>는 이 MVCC를 위해 <strong>언두 영역에 백업된 이전 데이터를 이용해 동일 트랜잭션 내에서는 동일한 결과를 보여줄 수 있게 보장한다.</strong></p>
<p>사실 READ COMMITED도 MVCC를 이용해 COMMIT되기 전의 데이터를 보여준다.</p>
<ul>
<li>이 둘의 차이<ul>
<li>언두 영역에 백업된 레코드의 여러 버전 가운데 몇 번째 이전 버전까지 찾아 들어가야 하느냐에 있다.</li>
</ul>
</li>
</ul>
<p>모든 InnoDB의 트랜잭션은 <strong>고유한 트랜잭션 번호(순사적으로 증가하는 값)</strong>을 가진다.</p>
<p>언두 영역에 백업된 모든 레코드에는 변경을 발생시킨 트랜잭션의 번호가 포함돼 있다.</p>
<p>언두 영역의 백업된 데이터는 InnoDB 스토리지 엔진이 불필요하다고 판단되는 시점에 <strong>주기적으로 삭제</strong>한다.</p>
<p>REPEATABLE READ 격리 수준에서는 MVCC를 보장하기 위해 실행 중인 트랜잭션 가운데 가장 오래된 트랜잭션 번호보다 트랜잭션 번호가 앞선 언두 영역의 데이터는 삭제할 수 없다.</p>
<p>가장 오래된 트랜잭션 번호 이전의 트랜잭션에 의해 변경된 모든 언두 데이터가 필요한 것은 아니다.</p>
<p>더 정확하게는 특정 트랜잭션 번호의 구간 내에서 백업된 언두 데이터가 보존돼야 한다.</p>
<hr />
<h4 id="repeatable-read-격리-수준-작동-방식">REPEATABLE READ 격리 수준 작동 방식</h4>
<p><img alt="" src="https://velog.velcdn.com/images/jojehuni_9759/post/acb8e8a5-f4b5-418e-800c-9387ba51556d/image.png" /></p>
<p>위 사진의 예제를 보자.</p>
<ul>
<li>사용자 A의 트랜잭션 번호는 12이며, 사용자 B의 트랜잭션 번호는 10이다.</li>
<li>사용자 A가 first_name을 “Toto”로 변경하고 커밋을 수행한다.</li>
<li>사용자 B가 emp_no=500000인 사원을 A 트랜잭션의 변경 전후 각각 한 번씩 SELECT했는데 결과는 항상 “Lara”라는 값을 가져온다.</li>
</ul>
<p>사용자 B가 BEGIN 명령으로 트랜잭션을 시작하면서 10번이라는 트랜잭션 번호를 부여받았는데, </p>
<p>그때부터 사용자 B는 10번 트랜잭션 안에서 실행되는 모든 SELECT 쿼리는 트랜잭션 번호가 10번(자신의 트랜잭션 번호)보다 작은 트랜잭션 번호에서 변경한 것만 보게 된다.</p>
<p>또한, REPEATABLE READ 격리 수준에서도 아래 사진과 같이 부정합이 발생할 수 있다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/jojehuni_9759/post/df7e6f1d-76e8-48f6-ad71-cced08c02c3d/image.png" /></p>
<p>사용자 B는 BEGIN 명령으로 트랜잭션을 시작한 후 SELECT을 수행하기 때문에 두 번의 SELECT 쿼리 결과는 똑같아야 한다.</p>
<p>-&gt; 근데 다름.. 사용자 B가 실행하는 두 번의 <code>SELECT ... FOR UPDATE</code> 쿼리 결과는 서로 다르다.</p>
<p><strong>다른 트랜잭션에서 수행한 변경 작업에 의해 레코드가 보였다 안 보였다 하는 현상</strong>을 <strong>팬텀 리드(PHANTOM READ)</strong>라고 한다.</p>
<p><code>SELECT ... FOR UPDATE</code> 쿼리는 SELECT하는 레코드에 쓰기 잠금을 걸어야 하는데, 언두 레코드에는 잠금을 걸 수 없다.</p>
<p><code>SELECT ... FOR UPDATE</code>나 <code>SELECT ... LOCK IN SHARE MODE</code>로 조회되는 레코드는 언두 영역의 변경 전 데이터를 가져오는 것이 아니라 현재 레코드의 값을 가져오게 되는 것이다.</p>
<hr />
<h3 id="selializable">SELIALIZABLE</h3>
<p><strong>가장 단순한 격리 수준</strong>이면서 동시에 <strong>가장 엄격한 격리 수준</strong>으로, 그만큼 <strong>동시 처리 성능도 다른 트랜잭션 격리 수준보다 떨어진다.</strong></p>
<p>InnoDB 테이블에서 기본적으로 순수한 SELECT(<code>INSERT ... SELECT ...</code> 또는 <code>CREATE TABLE ... AS SELECT ...</code>가 아닌) 작업은 아무런 레코드 잠금도 설정하지 않고 실행된다.</p>
<p>하지만 트랜잭션의 격리 수준이 SERIALIZABLE로 설정되면 읽기 작업도 공유 잠금(읽기 잠금)을 획득해야만 하며, 동시에 다른 트랜잭션은 그러한 레코드를 변경하지 못하게 된다.</p>
<p>즉, 한 트랜잭션에서 읽고 쓰는 레코드를 다른 트랜잭션에서는 절대 접근할 수 없다는 것이다.</p>
<p>SERIALIZABLE 격리 수준에서는 일반적인 DBMS에서 일어나는 “PHANTOM READ”라는 문제가 발생하지 않지 않는다.</p>
<p>그래도 InnoDB 스토리지 엔진에서는 갭 락과 넥스트 키 락 덕분에 REPEATABLE READ 격리 수준에서도 이미 “PHANTOM READ가 발생하지 않기 때문에” 굳이 SERIALIZABLE을 사용할 필요성은 없다.</p>
<p>다음은 6장 데이터 압축에 대해 다뤄보겠다.</p>
