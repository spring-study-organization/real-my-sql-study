<h1 id="myisam-스토리지-엔진-아키텍처">MyISAM 스토리지 엔진 아키텍처</h1>
<p>MyISAM 스토리지 엔진의 성능에 미치는 키 캐시, OS의 캐시 &amp; 버퍼에 대해 보는 목차</p>
<p><img alt="" src="https://velog.velcdn.com/images/jojehuni_9759/post/226ac357-cfb6-4f89-8da9-86a2d1384d65/image.png" /></p>
<p>위 그림은 MyISAM 스토리지 엔진의 구조다.</p>
<h2 id="키-캐시">키 캐시</h2>
<p>MyISAM의 키 캐시 : InnoDB의 버퍼 풀과 비슷한 역할을 한다.</p>
<ul>
<li>키 캐시는 인덱스만을 대상으로 작동한다.</li>
<li>인덱스의 디스크 쓰기 작업에 대해서만 부분적으로 버퍼링 역할을 한다.</li>
</ul>
<p>키 캐시가 얼마나 효율저긍로 작동하는지 수식으로 알 수 있다.</p>
<pre><code>키 캐시 히트율(Hit rate) = 100 - (Key_reads / Key_read_requests * 100)</code></pre><p><code>Key_reads</code> : 인덱스를 디스크에서 읽어들인 횟수
<code>Key_read_requests</code> : 키 캐시로부터 인덱스를 읽은 횟수를 저장하는 상태 변수</p>
<p>위의 상태 값들을 알아보는 방법으로는 <code>SHOW GLOBAL STATUS</code> 명령을 포함하여 아래와 같이 입력하면 된다고 한다.</p>
<pre><code class="language-sql">mysql&gt; SHOW GLOBAL STATUS LIKE 'key%';</code></pre>
<p>매뉴얼에서는 키 캐시를 이용한 쿼리의 비율을 99% 이상으로 유지하라고 권장하며, 미만이라면 키 캐시를 조금 더 크게 설정하는 것이 좋다.</p>
<hr />
<h2 id="운영체제의-캐시-및-버퍼">운영체제의 캐시 및 버퍼</h2>
<p>MyISAM 테이블의 인덱스는 키 캐시를 이용해 디스크를 검색하지 않고도 충분히 빠르게 검색할 수 있다.</p>
<p>하지만, MyISAM은 디스크로의 I/O를 해결할 캐시나 버퍼링 기능을 가지고 있지 않기 때문에 빈번한 디스크 I/O 호출을 발생시킬 수 있다.</p>
<p>따라서 MySQL 의 최대 물리 메모리를 40% 이상 넘지 않도록 설정하고, 나머지 메모리 공간을 운영체제가 사용하여 운영체제의 캐시 공간을 마련하는 것이 좋다.</p>
<blockquote>
<p>MySQL이나 다른 애플리케이션에서 메모리를 모두 사용해버리면, 운영체제가 캐시 용도로 사용할 수 있는 메모리 공간이 없어져서 MyISAM 테이블에 대한 쿼리 처리가 느려지기 때문이다.</p>
</blockquote>
<hr />
<h2 id="데이터-파일과-프라이머리-키인덱스-구조">데이터 파일과 프라이머리 키(인덱스) 구조</h2>
<p>InnoDB 스토리지 엔진을 사용하는 테이블은 프라이머리 키에 의해서 클러스터링이 돼 저장되는 것을 이전 게시글에서 봤다.</p>
<p>MyISAM 테이블은 프라이머리 키에 의한 클러스터링 없이 데이터 파일이 힙(Heap) 공간처럼 활용된다.</p>
<p><strong>즉, MyISAM 테이블에 레코드는 프라이머리 키 값과 무관하게 INSERT되는 순서대로 데이터 파일에 저장된다.</strong></p>
<p>추가로 저장되는 레코드는 모두 ROWID라는 물리적 주솟값을 가지며 모든 인덱스들은 이 값을 포인터로 가진다.</p>
<blockquote>
<p>MyISAM 테이블에서 ROWID는 가변길이와 고정 길이 두 가지 방법으로 저장될 수 있다.</p>
</blockquote>
<ul>
<li><p>고정길이</p>
<ul>
<li><p>MAX_ROWS 옵션을 설정하지 않으면 <code>myisam_data_pointer_size</code> 시스템 변수에 저장된 바이트 수만큼의 공간을 사용</p>
</li>
<li><p><code>myisam_data_pointer_size</code> 시스템 변수의 기본값은 7이므로 ROWID는 2바이트부터 7바이트까지 가변적인 ROWID를 갖게 된다.</p>
</li>
<li><p>첫 번째 바이트는 ROWID의 길이를 저장하고 나머지 공간은 실제 ROWID를 저장</p>
</li>
<li><p>가변적인 ROWID를 가지면 데이터 파일에서 레코드 위치(offset)가 ROWID로 사용</p>
</li>
</ul>
</li>
<li><p>가변길이</p>
<ul>
<li><p>자주 사용되지는 않지만 테이블을 생성할 때 MAX_ROWS 옵션을 사용할 수 있는데, 이 옵션을 명시하면 최대로 가질 수 있는 레코드가 한정된 테이블을 생성</p>
</li>
<li><p>MAX_ROWS 옵션에 의해 레코드의 개수가 한정되면 ROWID 값으로 4바이트 정수를 사용</p>
</li>
<li><p>이 때 레코드가 INSERT 된 순번이 ROWID로 사용</p>
</li>
</ul>
</li>
</ul>