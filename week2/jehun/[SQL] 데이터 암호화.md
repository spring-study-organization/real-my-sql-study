<h1 id="데이터-암호화">데이터 암호화</h1>
<p>💡 본 내용은 Real MySQL 8.0 1권을 읽으면서 정리한 내용입니다.</p>
<ul>
<li>본문에 포함된 사진의 출처는 Real MySQL 7장입니다.</li>
</ul>
<p>응용 프로그램의 암호화는 주요 정보를 가진 칼럼 단위로 암호화를 수행하며, 데이터베이스 수준에서는 테이블 단위로 암호화를 적용한다.</p>
<h2 id="mysql-서버의-데이터-암호화">MySQL 서버의 데이터 암호화</h2>
<p>데이터베이스 서버와 디스크 사이의 데이터 읽고 쓰기 지점에서 암호화 또는 복호화를 수행한다.
그래서 디스크 입출력 이외의 부분에서는 암호화 처리가 필요 없다.</p>
<p>즉, <strong>InnoDB 스토리지 엔진의 I/O 레이어에서만 데이터 암호화 및 복호화 과정이 실행된다.</strong></p>
<p><img alt="" src="https://velog.velcdn.com/images/jojehuni_9759/post/fb91545d-9d3a-4664-bcbd-9687986715be/image.png" /></p>
<ul>
<li>암호화된 테이블도 그렇지 않은 테이블과 동일한 쿼리 처리 과정을 거친다.<ul>
<li>MySQL 내부와 사용자 입장에서는 아무런 차이가 없기 때문이다.</li>
</ul>
</li>
</ul>
<p>이러한 암호화 방식을 <strong>TDE(Transparent Data Encryption, 투명한 데이터 암호화)</strong>이라고 함</p>
<hr />
<h3 id="2단계-키-관리">2단계 키 관리</h3>
<p>MySQL 서버의 TDE에서 암호화 키는 키링(KeyRing) 플러그인에 의해 관리된다.</p>
<ul>
<li>keyring_file File-Based 플러그인</li>
<li>keyring_encrypted_file Keyring 플러그인</li>
<li>keyring_okv KMIP 플러그인</li>
<li>keyring_aws Amazon Web Services Keyring 플러그인</li>
</ul>
<p>위와 같은 다양한 플러그인이 제공되지만, 마스터 키를 사용하는 방법만 다를 뿐 MySQL 서버 내부적으로 작동하는 방식은 모두 동일하다.</p>
<p>키링 플러그인은 2단계(2-Tier) 키 관리 방식을 사용함</p>
<p><img alt="" src="https://velog.velcdn.com/images/jojehuni_9759/post/114974cd-e3a6-4447-982a-4f4e451a8867/image.png" /></p>
<ul>
<li><p>2단계 암호화 아키텍처 사진이다.</p>
</li>
<li><p>데이터 암호화는 <strong>마스터 키</strong>와 <strong>테이블스페이스 키</strong>를 가지고 있다.</p>
<ul>
<li>외부 키 관리 솔루션 또는 디스크 파일에서 마스터 키를 가져온다.</li>
<li>암호화된 테이블이 생성될 때마다 해당 테이블을 위한 테이블스페이스 키를 발급한다.</li>
<li>MySQL 서버는 <strong>마스터 키를 이용해 테이블스페이스 키를 암호화</strong>해서 각 테이블의 데이터 파일 헤더에 저장한다.</li>
</ul>
</li>
<li><p>테이블스페이스 키는 테이블이 삭제되지 않는 이상 절대 변경되지 않는다.</p>
<ul>
<li>MySQL 서버 외부로도 절대 노출되지 않는다.</li>
</ul>
</li>
<li><p>마스터 키는 외부로 노출될 수 있기 때문에 주기적으로 변경해야 한다.</p>
<ul>
<li><code>ALTER INSTANCE ROTATE INNODB MASTER KEY;</code></li>
<li><strong>암호화 키 변경으로 인한 과도한 시스템 부하를 피하기 위해 2단계 암호화 방식을 사용한다.</strong></li>
</ul>
</li>
</ul>
<hr />
<h3 id="암호화와-성능">암호화와 성능</h3>
<ul>
<li><p>MySQL 서버의 암호화는 TDE 방식 ⇒ 디스크로부터 한 번 읽은 데이터 페이지는 복호화되어 InnoDB 버퍼 풀에 적재된다.</p>
<ul>
<li><strong>데이터 페이지가 한 번 메모리에 적재되면 암호화되지 않은 테이블과 동일한 성능</strong>을 보인다.</li>
</ul>
</li>
<li><p>InnoDB <strong>버퍼 풀에 존재하지 않는 데이터를 읽을 경우</strong></p>
<ul>
<li><strong>복호화 과정</strong>을 거쳐야 한다. -&gt; 즉, <strong>쿼리 처리가 지연</strong>된다.</li>
</ul>
</li>
<li><p><strong>암호화된 테이블이 변경되는 경우</strong></p>
<ul>
<li>다시 디스크로 동기화될 때 암호화 되어야 한다. -&gt; 즉, <strong>디스크 저장 시 추가 시간이 걸림</strong></li>
</ul>
</li>
<li><p>SELECT, UPDATE, DELETE 명령을 실행하는 경우</p>
<ul>
<li>변경하고자 하는 레코드를 InnoDB 버퍼 풀로 읽어와야 한다.</li>
<li><strong>즉, 새롭게 디스크에서 읽어야 하는 데이터 페이지의 개수에 따라 복호화 지연이 발생한다.</strong></li>
</ul>
</li>
<li><p>암호화한다고 해서 InnoDB 버퍼 풀의 효율이 달라지거나 메모리 사용 효율이 떨어지는 현상은 발생하지 않는다.</p>
<ul>
<li>데이터 페이지는 암호화 키보다 훨씬 크기 때문</li>
</ul>
</li>
<li><p><strong>같은 테이블에 대해 암호화와 압축이 동시에 적용되면 압축을 먼저 실행하고 암호화를 적용함</strong></p>
<ul>
<li>암호화된 결과문은 압축률이 떨어짐</li>
<li>암호화가 먼저 실행된다면, InnoDB 버퍼 풀에 존재하는 데이터 페이지에 대해서도 매번 암복호화 작업을 수행해야 됨</li>
</ul>
</li>
<li><p>암호화된 테이블의 경우 읽기는 3<del>5배, 쓰기는 5</del>6배정도 느림</p>
</li>
</ul>
<hr />
<h3 id="암호화와-복제">암호화와 복제</h3>
<p>MySQL 서버의 복제에서 레플리카 서버는 소스 서버의 모든 사용자 데이터를 동기화하기 때문에 실제 데이터 파일도 동리할 것이라 생각할 수 있다.</p>
<p>하지만, TDE를 이용한 암호화 사용 시 -&gt; 마스터 키와 데이블스페이스 키는 그렇지 않다.</p>
<p>MySQL 서버에서 기본적으로 모든 노드는 각자의 마스터 키를 할당해야 한다.</p>
<p>DB 서버의 로컬 디렉터리에 마스터 키를 관리할 때는 두 서버가 다른 키를 가질 수밖에 없겠지만, 원격으로 키 관리 솔루션을 사용하는 경우에도 두 서버는 서로 다른 마스터 키를 갖도록 해야 한다.</p>
<p>=&gt; 애초에 마스터 키 자체가 레플리카로 복제되지 않아 테이블 스페이스 키도 복제되지 않는다.</p>
<blockquote>
<p>즉, 소스 서버와 레플리카 서버는 서로 각자의 마스터 키와 테이블스페이스 키를 관리한다.
그래서 실제 암호화된 데이터가 저장된 데이터 파일의 내용은 완전히 달라진다.</p>
</blockquote>
<p>복제 소스 서버의 마스터 키를 변경할 때는</p>
<pre><code class="language-sql">ALTER INSTANCE ROTATE INNODB MASTER KEY</code></pre>
<p>위의 명령을 실행하는데, 이 때 이 명령 자체는 레플리카 서버로 복제되지만, 실제로 전달되는 것은 아니다.</p>
<blockquote>
<p>마스터 키 로테이션을 실행하면 소스 서버와 레플리카 서버가 각각 서로 다른 마스터 키를 새로 발급받는다.</p>
</blockquote>
<p>MySQL 서버의 백업에서 TDE의 키링 파일을 백업하지 않는 경우가 있다. =&gt; 이 경우 키링 파일을 찾지 못하면 데이터 복구가 불가능해진다.</p>
<p>만약, 키링 파일과 데이터 백업을 별도로 백업한다면, 마스터 키 로테이션 명령으로 TDE의 마스터 키가 언제 변경됐는지 기억해둬야 한다.</p>
<blockquote>
<p><strong>물론 보안을 위해 별도로 보관하는 것을 권장하긴 한다. 그럼에도 복구를 감안하고 백업 방식을 선택해야 한다.</strong></p>
</blockquote>
<hr />
<h2 id="keyring_file-플러그인-설치">keyring_file 플러그인 설치</h2>
<p>TDE의 암호화 키 관리는 플러그인 방식으로 제공된다.</p>
<ul>
<li><p><code>keyring_file</code> 플러그인은 테이블스페이스 키를 암호화하기 위해 <strong>마스터 키를 디스크 파일로 관리함</strong></p>
<ul>
<li>마스터 키가 저장된 파일이 외부에 노출되면 데이터 암호화는 무용지물된다.. 조심.</li>
</ul>
</li>
<li><p>TDE 플러그인은 가장 빨리 초기화돼야 하므로 MySQL 서버의 설정 파일(my.cnf)에 다음을 추가하면 된다.</p>
<ul>
<li>라이브러리 명시<ul>
<li><code>early-plugin-load = keyring_file.so</code></li>
</ul>
</li>
<li>마스터 키를 저장할 파일 경로 설정<ul>
<li><code>keyring_file_data = /very/secure/directory/tde_master.key</code></li>
</ul>
</li>
</ul>
</li>
<li><p><code>SHOW PLUGINS</code> 명령으로 <code>keyring_file</code> 플러그인의 초기화 여부 확인 가능</p>
</li>
</ul>
<hr />
<h2 id="테이블-암호화">테이블 암호화</h2>
<p>키링 플러그인은 마스터 키를 생성하고 관리하는 부분까지만 담당한다.</p>
<p>따라서 어떤 키링 플러그인을 사용하든지 암호화된 테이블을 생성하고 활용하는 방법은 모두 동일하다.</p>
<h3 id="테이블-생성">테이블 생성</h3>
<p><code>ENCRYPTION='Y'</code> 옵션만 넣으면 됨</p>
<pre><code class="language-sql">mysql&gt; CREATE TABLE tab_encrypted (
        id INT,
        data VARCHAR(100),
        PRIMARY KEY(id)
    ) ENCRYPTION='Y';</code></pre>
<p>모든 테이블에 암호화를 적용하려면 <code>default_table_encryptioni</code> 시스템 변수를 <code>ON</code>으로 설정하면 됨</p>
<hr />
<h3 id="응용-프로그램-암호화와의-비교">응용 프로그램 암호화와의 비교</h3>
<p>응용 프로그램에서 직접 암호화해서 MySQL 서버에 저장하는 경우도 있다.</p>
<p>이 경우에는 아래와 같이 될 수 있다.</p>
<ol>
<li>칼럼 값의 암호화 여부를 서버가 인지하지 못한다.</li>
<li>암호화된 칼럼은 인덱스의 기능을 100% 활용하지 못한다.</li>
<li>암호회된 칼럼을 기준으로 정렬하는 쿼리를 사용하지 못한다.</li>
</ol>
<p>3 - 1.MySQL 서버는 이미 암호화된 값을 기준으로 정렬한다. 그래서 암호화 되기 전의 값을 기준으로 정렬할 수 없다.</p>
<blockquote>
<p>즉, 고민할 필요 없이 MySQL 서버의 암호화 기능을 선택할것을 권장한다.</p>
</blockquote>
<hr />
<h3 id="테이블스페이스-이동">테이블스페이스 이동</h3>
<ul>
<li>테이블스페이스 이동 기능이 훨씬 효율적이고 빠른 경우<ul>
<li>테이블을 다른 서버로 복사해야 할 때</li>
<li>특정 테이블의 데이터 파일만 백업했다가 복구해야 할 때</li>
</ul>
</li>
</ul>
<p><code>FLUSTH TABLES</code> 명령으로 테이블스페이스 <code>Export</code> 가능</p>
<pre><code class="language-sql">mysql&gt; FLUSH TABLES source_table FOR EXPORT;</code></pre>
<h4 id="암호화되지-않은-테이블의-테이블스페이스-복사-과정">암호화되지 않은 테이블의 테이블스페이스 복사 과정</h4>
<ol>
<li><code>FLUSTH TABLES</code> 명령 실행</li>
<li>source_table의 저장되지 않은 변경 사항을 모두 디스크로 기록한다.</li>
<li>더 이상 source_table에 접근할 수 없게 잠금을 건다.</li>
<li>source_table의 구조를 <code>source_table.cfg</code> 파일로 기록</li>
<li>데이터 파일과 <code>source_table.cfg</code> 파일을 목적지 서버로 복사</li>
</ol>
<h4 id="암호화된-테이블의-테이블스페이스-복사-과정">암호화된 테이블의 테이블스페이스 복사 과정</h4>
<ol>
<li><code>FLUSTH TABLES</code> 명령 실행</li>
<li>임시로 사용할 마스터 키 발급해서 <code>source_table.cfp</code> 파일로 기록한다.</li>
<li>기존 마스터 키를 이용해서 암호화된 테이블의 테이블스페이스 키를 복호화한다.</li>
<li>임시로 발급된 마스터 키를 이용해서 테이블스페이스 키를 다시 암호화해서 데이터 파일의 헤더 부분에 저장한다.</li>
<li>데이터 파일과 <code>source_table.cfp</code> 파일을 목적지 서버로 복사한다.</li>
</ol>
<hr />
<h2 id="언두-로그-및-리두-로그-암호화">언두 로그 및 리두 로그 암호화</h2>
<p>MySQL 8.0.16 버전부터 <code>innodb_undo_log_encrypt</code>, <code>innodb_redo_log_encrypt</code> 시스템 변수를 이용해 InnoDB 스토리지 엔진의 <strong>리두 로그와 언두 로그를 암호화된 상태로 저장할 수 있다.</strong></p>
<p>테이블의 암호화는 일단 테이블 하나에 암호화가 적용되면 해당 테이블의 모든 데이터가 암호화돼야 한다.</p>
<p>하지만 리두 로그나 언두 로그는 그렇게 적용할 수 없다.</p>
<blockquote>
<p>즉, 실행 중인 MySQL 서버에서 언두 로그나 리두 로그를 활성화한다고 해도, 모든 리두 로그나 언두 로그의 데이터를 해당 시점에 한 번에 암호화해서 다시 저장할 수 없다.</p>
</blockquote>
<p>그래서 아래와 같은 경우들이 있다.</p>
<ol>
<li><p>리두 로그나 언두 로그를 평문으로 저장하다가 암호화가 활성화되는 경우</p>
<ul>
<li>그때부터 생성되는 리두 로그나 언두 로그만 암호화해서 저장</li>
<li>비활성화 되는 경우도 마찬가지</li>
</ul>
</li>
<li><p>리두 로그와 언두 로그 데이터 모두 각각의 테이블스페이스 키로 암호화 됨</p>
<ul>
<li>테이블스페이스 키는 다시 마스터 키로 암호화됨</li>
</ul>
</li>
</ol>
<blockquote>
<p>즉, 리두 로그와 언두 로그를 위한 각각의 프라이빗 키가 발급되고, 해당 프라이빗 키는 마스터 키로 암호화되어 리두 로그 파일과 언두 로그 파일의 헤더에 저장됨</p>
</blockquote>
<p><strong>InnoDB 리두 로그의 암호화 여부 확인 방법</strong></p>
<pre><code class="language-sql">mysql&gt; SHOW GLOBAL VARIABLES LIKE 'innodb_redo_log_encrypt';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| innodb_redo_log_encrypt | OFF   |</code></pre>
<hr />
<h2 id="바이너리-로그-암호화">바이너리 로그 암호화</h2>
<h3 id="바이너리-로그-암호화-키-관리">바이너리 로그 암호화 키 관리</h3>
<p>바이너리 로그 파일의 암호화는 상황에 따라 중요도가 높아질 수 있다.
바이너리 로그와 릴레이 로그 파일 데이터의 암호화를 위해 <strong>2단계 암호화 키 관리 방식을 사용</strong>한다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/jojehuni_9759/post/8646ea15-3750-4724-aae8-40e290212a45/image.png" /></p>
<p><strong>파일 키로 로그 파일의 데이터를 암호화</strong>해서 디스크에 ㅓ장한다.</p>
<ul>
<li><strong>파일 키는 바이너리 로그 암호화 키로 암호화</strong>해서 각 바이너리 로그와 릴레이 로그 파일의 헤더에 저장한다.<ul>
<li>즉, <strong>바이너리 로그 암호화 키는 마스터 키와 동일한 역할</strong></li>
</ul>
</li>
</ul>
<hr />
<h3 id="바이너리-로그-암호화-키-변경">바이너리 로그 암호화 키 변경</h3>
<p><code>ALTER INSTANCE ROTATE BINLOG MASTER KEY</code> 명령으로 바이너리 암호화 키를 변경할 수 있다.</p>
<p>MySQL 서버의 바이너리 로그 파일이 암호화돼 있는지 여부 확인 방법</p>
<pre><code class="language-sql">mysql&gt; SHOW BINARY LOGS;</code></pre>
<hr />
<h3 id="mysqlbinlog-도구-활용">mysqlbinlog 도구 활용</h3>
<p>바이너리 로그 파일이 한 번 암호화되면 바이너리 로그 암호화 키가 없으면 복호화할 수 없다.</p>
<p>근데, 암호화 키는 MySQL 서버만 갖고 있어서 복호화가 불가능하다..</p>
<p><code>mysqlbinlog</code>를 활용해서 MySQL 서버에 접속해서 바이너리 로그를 가져오는 방법밖에 없다.</p>
<pre><code class="language-sql">linux&gt; mysqlbinlog --read-from-remote-server -uroot -p -vvv mysql-bin.000011</code></pre>
<p>mysql-bin.000011 로그 파일을 가지고 있다는 가정하에 mysql-bin.000011 로그 파일의 내용을 확인하고자 한다면, 위와 같이 하는 수밖에 없다.</p>
<p>mysqlbinlog 도구가 mysql-bin.000011 파일을 읽는 것은 아니기 때문에 <code>--read-from</code> 파라미터가 필요하다.</p>