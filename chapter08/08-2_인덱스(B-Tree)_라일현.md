### 8.3.4 B-Tree 인덱스를 통한 데이터 읽기
- 인덱스 레인지 스캔

#### 8.3.4.1 인덱스 레인지 스캔
![](https://i.esdrop.com/d/f/5BqG1Oh0zF/TPJ8HIyhWR.png "인덱스를 이용한 레인지 스캔")
1. 인덱스 탐색(Index seek) : 인덱스에서 조건을 만족하는 값이 저장된 위치를 찾는다
2. 인덱스 스캔(Index scan) : 필요한 만큼 인덱스를 차례대로 쭉 읽는다
3. 인덱스 키와 레코드 주소를 이용해 레코드가 저장된 페이지를 가져오고 최종 레코드를 읽어옴 (경우에 따라 생략 가능)

```
mysql> SELECT * FROM employees WHERE first_name BETWEEN 'Ebbe' AND 'Gad';
```
- 가장 대표적이고 검색해야 할 인덱스의 범위가 결정됐을 때 사용하는 방식
- 시작해야 할 위치를 찾으면 그때부터는 리프 노드의 레코드만 순서대로 읽으면 된다
- 리프노드의 끝까지 읽으면 리프 노드 간의 링크를 이용해 다음 리프 노드를 찾아서 다시 스캔

![](https://i.esdrop.com/d/f/5BqG1Oh0zF/qFWAivw5Oh.png "인덱스 레인지 스캔을 통한 레코드 읽기")

- B-tree 인덱스에서 루트와 브랜치 노드를 이용해 스캔 시작 위치를 검색, 그 지점부터 필요한 방향으로 인덱스를 읽어 나감
- 리프 노드에 저장된 레코드 주소로 데이터 파일의 레코드를 읽어올 때 레코드 한 건 단위로 랜덤 I/O가 한 번씩 일어남
- 인덱스를 통해 읽어야 할 데이터 레코드가 20~25%를 넘으면 인덱스를 통한 읽기보다 테이블의 데이터를 직접 읽는 것이 더 효율적인 처리 방식
- 커버링 인덱스 : 인덱스만으로 조회

#### 8.3.4.2 인덱스 풀 스캔
![](https://i.esdrop.com/d/f/5BqG1Oh0zF/m95iWYrCxJ.png "인덱스 풀 스캔")
- 인덱스의 처음부터 끝까지 모두 읽는 방식
- 쿼리의 조건절에 사용된 칼럼이 인덱스의 첫 번째 칼럼이 아닌 경우 인덱스 풀 스캔 방식이 사용됨
	(A,B,C) 순서로 만들어져있지만 B 칼럼이나 C 칼럼으로 검색
- 먼저 인덱스 리프 노드의 제일 앞 또는 제일 뒤로 이동 > 인덱스 리프 노드를 연결하는 링크드 리스트를 따라 처음부터 끝까지 스캔
- 테이블 풀 스캔보다 효율적

#### 8.3.4.3 루스 인덱스 스캔
![](https://i.esdrop.com/d/f/5BqG1Oh0zF/1MeROc8Ute.png "루스 인덱스 스캔")
- 오라클의 인덱스 스킵 스캔과 비슷하다
- 중간에 불필요한 인덱스 키 값은 무시하고 다음으로 넘어감
- 일반적으로 GROUP BY 또는 집합 함수 가운데 MAX(), MIN() 함수 대해 최적화하는 경우에 사용
```
// dept_emp, emp_no 두 개의 칼럼으로 인덱스 생성되어 있음
// (dept_no, emp_no) 조합으로 정렬
mysql> SELECT dept_no, MIN(emp_no)
	   FROM dept_emp WHERE dept_no BETWEEN 'd002' AND 'd004'
	   GROUP BY dept_no
```

#### 8.3.4.4 인덱스 스킵 스캔

```
mysql> ALTER TABLE employees ADD INDEX ix_gender_birthdate (gender, birth_date);
// 인덱스를 사용하지 못하는 쿼리
mysql> SELECT * FROM employees WHERE birth_date >= '1965-02-01'
// 인덱스를 사용할 수 있는 쿼리
mysql> SELECT * FROM employees WHERE gender='M' AND birth_date>='1965-02-01';
mysql> SELECT gender, birth_date FROM employees
	   WHERE birth_date >= '1965-02-01';
// 아래 두 쿼리를 실행하는 것과 비슷한 형태의 최적화를 실행
mysql> SELECT gender, birth_date FROM employees WHERE gender = 'M' AND birth_date >= '1965-02-01';
mysql> SELECT gender, birth_date FROM employees WHERE gender = 'F' AND birth_date >= '1965-02-01';
```


- 인덱스의 핵심은 값이 정렬돼 있다

- gender 칼럼과 birth_date 칼럼 조건을 모두 가진 두 번재 쿼리는 인덱스를 효율적으로 사용할 수 있음
- gender 칼럼에 대한 비교 조건이 없는 첫 번째 쿼리는 인덱스를 사용할 수 업음 > birth_date 칼럼부터 시작하는 인덱스를 새로 생성해야 함
- 인덱스 스킵 스캔 최적화 기능 : 첫 칼럼을 뛰어넘어 두 번째 칼럼만으로 인덱스 검색을 가능하게 해줌
- 루스 인덱스 스캔은 GROUP BY 작업을 처리하기 위해 인덱스를 사용하는 경우에만 적용 가능
- 실행계획에서 type "index" : 인덱스 풀 스캔 / "range" : 필요한 부분만 읽음 (Extra에서 상세 내용 참고)

![](https://i.esdrop.com/d/f/5BqG1Oh0zF/I09Y1fBv6P.png "Title of image here")

- 단점
	WHERE 조건절에 조건이 없는 인덱스의 선행 칼럼의 유니크한 값의 개수가 적어야 함
		- 유니크한 값이 많다면 인덱스에서 스캔해야 할 시작 지점을 검색하는 작업이 많이 필요해짐
	쿼리가 인덱스에 존재하는 칼럼만으로 처리 가능해야 함(커버링 인덱스) - 나머지 칼럼이 필요해지면 인덱스 스킵 스캔이 아닌 풀 테이블 스캔으로 실행 계획 수립

### 8.3.5 다중 칼럼(Multi-column) 인덱스
![](https://i.esdrop.com/d/f/5BqG1Oh0zF/OiHL2eHJCf.png "다중 칼럼 인덱스")

- 두 개 이상의 칼럼으로 구성된 인덱스를 다중 칼럼 인덱스(복합 칼럼 인덱스)라고 함
- 인덱스의 두 번째 칼럼은 첫 번째 칼럼에 의존해서 정렬되어 있음

### 8.3.6 B-Tree 인덱스의 정렬 및 스캔 방향
- 인덱스의 키 값은 항상 오름차순이거나 내림차순으로 정렬되어 저장됨
- 인덱스를 거꾸로 읽으면 정렬한 방향과 반대 방향으로도 사용될 수 있음
- 인덱스를 어느 방향으로 읽을지는 쿼리에 따라 옵티마이저가 실시간으로 만들어내는 실행 계획에 따라 결정

#### 8.3.6.1 인덱스의 정렬
- MySQL 8.0부터 정렬 순서를 혼합한 인덱스도 생성 가능

##### 8.3.6.1.1 인덱스 스캔 방향
![](https://i.esdrop.com/d/f/5BqG1Oh0zF/5fbHAtVlTs.png "인덱스의 오름차순과 내림차순 읽기")
- 인덱스는 최솟값 최댓값 전부 접근 가능 (쿼리에서 내림차순인지 오름차순인지에 관계없이 인덱스를 읽는 순서만 변경해서 해결 가능)

##### 8.3.6.1.2 내림차순 인덱스
![](https://i.esdrop.com/d/f/5BqG1Oh0zF/CajZnWQ0Lz.png "인덱스 정순(포워드) 스캔과 인덱스 역순(백워드) 스캔")
- 역순 정렬 쿼리가 정순 정렬 쿼리보다 28.9% 더 걸림
- 페이지 잠금이 인덱스 정순 스캔에 적합한 구조
- 페이지 내에서 인덱스 레코드가 단방향으로만 연결된 구조
![](https://i.esdrop.com/d/f/5BqG1Oh0zF/0zPnrMLDZC.png "InnoDB 페이지 내에서 레코드들의 연결")
- InnoDB 페이지는 힙처럼 사용되기 때문에 물리적으로 저장이 순서대로 배치되진 않는다
- InnoDB 스토리지 엔진에서 데이터 파일은 프라이머리 키 인덱스 자체라는 것은 무슨 의미인가?

### 8.3.7 B-Tree 인덱스의 가용성과 효율성

#### 8.3.7.1 비교 조건의 종류와 효율성

```
mysql> SELECT * FROM dept_emp
WHERE dept_no='d002' AND emp_no >= 10114 ;
```

- 케이스 A : INDEX (dept_no, emp_no)
	-> dept_no='d002'인 레코드를 찾고 이후에는 dept_no가 'd002'가 아닐 때까지 인덱스를 읽음
- 케이스 B : INDEX (emp_no, dept_no)
	-> emp_no >= 10114 AND dept_no='d002' 인 레코드를 찾고 그 이후에 모든 레코드가 d0002인지 비교하는 과정을 거쳐야 함
- 필터링 : 인덱스를 통해 읽은 레코드가 나머지 조건에 맞는지 비교하면서 취사선택하는 작업

#### 8.3.7.2 인덱스의 가용성

![](https://i.esdrop.com/d/f/5BqG1Oh0zF/eoBfRqQSZt.png "")

```
// 왼쪽 기준 정렬 기반의 인덱스인 B-tree에서는 인덱스 효과 볼 수 없음
mysql> SELECT * FROM employees WHERE first_name LIKE '%mer';
```

#### 8.3.7.3 가용성과 효율성 판단
- B-Tree 인덱스를 사용할 수 없는 경우 (체크 조건으로는 사용 가능)
	1. NOT-EQUAL로 비교된 경우 (<>, NOT IN, NOT BETWEEN, IS NOT NULL)
	2. LIKE '%??' (앞부분이 아닌 뒷부분 일치) 형태로 문자열 패턴이 비교된 경우
	3. 스토어드 함수나 다른 연산자로 인덱스 칼럼이 변형된 후 비교된 경우 (SUBSTRING, DAYOFMONTH)
	4. NOT-DETERMINISTIC 속성의 스토어드 함수가 비교 조건에 사용된 경우
	5. 데이터 타입이 서로 다른 비교(인덱스 칼럼의 타입을 변환해야 비교가 가능한 경우)
	6. 문자열 데이터 타입의 콜레이션이 다른 경우
- 일반적인 DBMS에서는 NULL 값이 인덱스에 저장되지 않지만 MySQL에서는 NULL 값도 인덱스에 저장됨
- 다중 칼럼으로 만들어진 인덱스를 사용할 수 없는 경우
	작업 범위 결정 조건으로 인덱스를 사용하지 못하는 경우
- 작업 범위 결정 조건으로 인덱스를 사용하는 경우(i는 2보다 크고 n보다 작은 임의의 값을 의미)
	인덱스를 동등 비교 형태(=, IN)로 비교
	크다 작다 형태(>, <)
	LIKE로 좌측 일치 패턴


## 8.4 R-Tree 인덱스
- 공간인덱스 : R-Tree 인덱스 알고리즘을 이용해 2차원 데이터를 인덱싱하고 검색하는 목적의 인덱스
- 공간 확장 기능
	- 공간 데이터를 저장할 수 있는 데이터 타입
	- 공간 데이터 검색을 위한 공간 인덱스(R-Tree 알고리즘)
	- 공간 데이터의 연산 함수(거리 또는 포함 관계의 처리)

### 8.4.1 구조 및 특성
![](https://i.esdrop.com/d/f/5BqG1Oh0zF/6DPbx2THyv.png "")
- GEOMETRY 타입 : POINT, LINE, POLYGON의 슈퍼 타입
- MBR(Minimum Bounding Rectangle) : 해당 도형을 감싸는 최소 크기의 사각형
- MBR 사각형들의 포함 관계를 B-Tree 형태로 구현한 인덱스가 R-Tree 인덱스

![](https://i.esdrop.com/d/f/5BqG1Oh0zF/gwtFufOOvL.png "")
![](https://i.esdrop.com/d/f/5BqG1Oh0zF/4VCX8dz89W.png "")
- 최상위 레벨 : R1, R2
- 차상위 레벨 : R3, R4, R5, R6
- 최하위 레벨 : R7 ~ R14
![](https://i.esdrop.com/d/f/5BqG1Oh0zF/BAfLaL3jam.png "")

### 8.4.2 R-Tree 인덱스의 용도
- WGS84(GPS) 기준의 위도 경도 좌표 저장에 주로 사용됨
- ST_Contains(), ST_Within() 등과 같은 포함 관계를 비교하는 함수로 검색을 수행하는 경우에만 인덱스를 이용할 수 있음
![](https://i.esdrop.com/d/f/5BqG1Oh0zF/StzuIFsyID.png "")

```
// ST_Contains() 또는 ST_Within()을 이용해 "사각 상자"에 포함된 좌표 Px만 검색
// 비슷한 비교를 수행하지만 파라미터는 반대로 사용해야 함
mysql> SELECT * FROM tb_location WHERE ST_Contains(사각 상자, px);
mysql> SELECT * FROM tb_location WHERE ST_Within(px, 사각 상자);

// P6를 제거해야 한다면 ST_Distance_Sphere() 함수를 이용해서 필터링
mysql> SELECT * FROM tb_location WHERE ST_Contains(사각 상자, px) AND ST_Distance_Sphere(p, px)<=5*1000;
```

## 8.5 전문 검색 인덱스
- 전문 검색 : 문서의 내용 전체를 인덱스화해서 특정 키워드가 포함된 문서를 검색
- InnoDB, MyISAM 스토리지 엔진에서 제공하는 일반적인 용도의 B-Tree 인덱스를 사용할 수 없음

### 8.5.1 인덱스 알고리즘
- 문서의 키워드를 인덱스하는 기법에 따라 나누면 어근 분석 / n-gram 분석 알고리즘으로 구분 가능

#### 8.5.1.1 어근 분석 알고리즘
- 전문 검색 인덱스의 색인 작업
	- 불용어(Stop Word) 처리
		가치가 없는 단어를 모두 필터링해서 제거하는 작업
		불용어의 개수가 많지 않아 모두 상수로 정의해서 사용하는 경우가 많음
		유연성을 위해 불용어 자체를 데이터베이스화해서 사용자가 추가하거나 삭제할 수 있게 구현하는 경우도 있음
	- 어근 분석(Stemming)
		검색어로 선정된 단어의 뿌리인 원형을 찾는 작업
		오픈소스 형태소 분석 라이브러리 MeCab을 플러그인 형태로 사용할 수 있게 지원
		
#### 8.5.1.2 n-gram 알고리즘
- 단순히 키워드를 검색해내기 위한 인덱싱 알고리즘
- 본문을 무조건 몇 글자씩 잘라서 인덱싱하는 방법
- 형태소 분석보다 알고리즘이 단순하고 국가별 언어에 대한 이해와 준비 작업이 필요 없음
- 만들어진 인덱스의 크기는 상당히 큰 편
- n은 인덱싱할 키워드의 최소 글자 수를 의미한다
- 2-gram 방식이 많이 사용됨
- To be or not to be. That is the question
	띄어쓰기와 마침표를 기준으로 10개의 단어로 구분
	2글자씩 중첩해서 토큰으로 분리
- 중복된 토큰은 하나의 인덱스 엔트리로 병합되어 저장

```
// 서버에 내장된 불용어 조회
mysql> SELECT * FROM information_schema.INNODB_FT_DEFAULT_STOPWORD;
```

#### 8.5.1.3 불용어 변경 및 삭제
- 전문 검색 인덱스의 불용어 처리 무시/사용자 정의 불용어 사용
	- 스토리지 엔진에 관계없이 MySQL 서버의 모든 전문 검색 인덱스에 대해 불용어를 완전히 제거
		MySQL 서버의 설정 파일(my.cnf)의 ft_stopword_file시스템 변수에 빈 문자열 설정 후 재시작
		ft_stopword_file에 사용자가 정의한 불용어 목록을 저장한 파일의 경로를 설정해주면 해당 불용어를 적용할 수 있음
	- InnoDB 스토리지 엔진을 사용하는 테이블의 전문 검색 인덱스에 대해서만 불용어 처리를 무시
		innodb_ft_enable_stopword=OFF; -> 동적인 시스템 변수로 서버 실행중에도 변경 가능
		불용어 목록을 테이블에 저장해서 사용자 정의의 불용어를 사용할 수 있음 (innodb_ft_server_stopword_table)
		
### 8.5.2 전문 검색 인덱스의 가용성
- 전문 검색 인덱스를 사용하려면 갖춰야하는 조건
	- 쿼리 문장이 전문 검색을 위한 문법(MATCH ... AGAINST ...)을 사용
	- 테이블이 전문 검색 대상 칼럼에 대해서 전문 인덱스 보유

```
mysql> SELECT * FROM tb_test WHERE MATCH(doc_body) AGAINST ('애플' IN BOOLEAN MODE)
```