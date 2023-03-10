# 8. 인덱스(the others)

> [8.5 전문 검색 인덱스](#8.5-전문-검색-인덱스)
>
> - 어근 분석(stemming)
> - n-gram

<br>

## 8.5 전문 검색 인덱스

> B/R-Tree 인덱스
>
> - 일반적으로 크지 않은 데이터 또는 이미 키워드화한 작은 값에 대한 인덱싱 알고리즘
> - B-Tree 인덱스는 실제 컬럼 값이 1MB 더라도 전체를 인덱스 키로 사용하는 것이 아니라, 3,072byte(InnoDB)까지만 잘라서 인덱스 키로 사용
> - 전체 또는 좌측 일치와 같은 검색만 가능
>
> 전문 검색 인덱스(Full Text Index)
>
> - 문서 전체에 대한 분석과 검색을 위한 인덱싱 알고리즘

<br>

### 8.5.1 인덱스 알고리즘

- 전문 검색에서는 문서 본문의 내용에서 사용자가 검색하게 될 키워드를 분석하고, 빠른 검색용으로 사용할 수 있게 해당 키워드로 인덱스 구축
- 키워드 분석 및 인덱스 구축 방법
  - 단어의 어근 분석
  - n-gram 분석 알고리즘

##### 8.5.1.1 어근 분석 알고리즘

전문 검색 인덱스는 2가지 중요한 과정을 거쳐 색인 작업 수행

- 불용어(Stop Word) 처리

  - 검색에서 가치 없는 단어 필터링하여 제거
  - 불용어 갯수가 많지 않으므로 알고리즘 구현한 코드에 상수로 정의해 사용하는 경우 많음
  - 유연성을 위해 불용어 자체를 데이터베이스화해서 사용자가 추가/삭제하도록 구현하는 경우도 있음
  - 현재는 불용어가 소스코드로 정의되어 있지만, 이를 무시하고 사용자가 별도로 불용어 정의할 수 있는 기능 제공

- 어근 분석(Stemming)

  - 검색어로 선정된 단어의 뿌리인 원형 찾는 작업

  - MeCab(오픈소스 형태소 분석 라이브러리)을 플러그인 형태로 사용하도록 지원

    - 단어 사전 필요

    - 문장 해체하여 각 단어 품사 식별할 수 있는 문장 구조 인식 필요

      (실제 언어 샘플을 이용해 언어 학습하는 과정 수행하는데, 상당한 시간이 필요한 작업)

  - 한글이나 일본어는 영어와 같이 단어 변형 자체는 거의 없으므로, 어근 분석 보다는 문장의 형태소 분석해서 명사와 조사 구분하는 기능이 더 중요한 편임

    (서구권 언어 형태소 분석기: MongoDB에서 사용하는 Snowball 오픈소스)

##### 8.5.1.2 n-gram 알고리즘

- 전문적 검색 엔진을 고려하면 시간과 노력이 필요해서 범용적으로 적용하기 쉽지 않음. 이러한 단점 보완을 위해 n-gram 알고리즘 도입(단순히 키워드 검색만 함)

- 본문을 무조건 몇 글자씩 잘라서 인덱싱하는 방법

  - 알고리즘이 단순하고 국가별 언어에 대한 이해와 준비 작업 필요 없음

  - 하지만 만들어진 인덱스 크기는 상당히 큼

  - n은 인덱싱 할 키워드의 최소 글자 수

    (일반적으로 2글자로 쪼개는 2-gram 또는 Bi-gram 사용)

- 예시

  `To be or not to be. That is the question`

  - 각 글자를 중첩해서 2글자씩 토큰으로 구분

    (10글자라면 (10-1)개의 토큰으로 구분)

  - 구분된 토큰을 인덱스에 저장

    (중복 토큰은 하나의 인덱스 엔트리로 병합되어 저장)

- 불용어 filter

  - 생성 토큰들에 대해 불용어 걸러내는 작업 수행

  - 불용어와 동일하거나 불용어 포함하는 경우 걸러서 버림

  - MySQL 서버 내장 불용어 확인

    ```sql
    SELECT * FROM information_schema.INNODB_FT_DEFAULT_STOPWORD;
    ```

- 기타
  - 전문 검색 더 빠르게 하기 위해 2단계 인덱싱(프론트엔드/백엔드 인덱스) 방법도 있긴 함
  - 성능 향상을 위한 Merge-Tree 기능도 있긴 함

<br>

##### 8.5.1.3 불용어 변경 및 삭제

- 불용어 처리가 더 혼란스럽게 만들 수도 있어서 불용어를 무시하거나 사용자 정의 불용어를 등록하는 방법 권장

- 불용어 처리 무시

  - 스토리지 엔진 관계 없이 모든 불용어 완전 제거

    ```txt
    # my.cnf
    ft_stopword_file=''
    ```

    - 서버 설정 파일(my.cnf)의 ft_stopword_file 시스템 변수 빈 문자열 설정
    - 서버 시작시에만 인지하므로 설정 변경 시 서버 재시작 필요

  - InnoDB 스토리지 엔진 사용 테이블에만 적용

    ```sql
    SET GLOBAL innodb_ft_enable_stopword=OFF|ON;
    ```

    - 동적 시스템 변수로 서버 실행 중에도 변경 가능

- 사용자 정의 불용어 사용

  - 설정 파일

    ```txt
    # my.cnf
    ft_stopword_file={file_path}
    ```

  - 불용어 목록 테이블로 저장

    ```sql
    CREATE TABLE my_stopword(value VARCHAR(30)) ENGINE = INNODB;
    INSERT INTO MY_STOPWORD(value) VALUES ('MySQL');
    
    SET GLOBAL innodb_ft_server_stopwrod_table='mydb/my_stopword';
    
    ALTER TABLE tb_bi_gram
    ADD FULLTEXT INDEX fx_title_body(title, body) WITH PARSER ngram;
    ```

    - 불용어 테이블 생성 후 `innodb_ft_server_stopword_table` 시스템 변수에 불용어 테이블 설정
    - 목록 변경 이후 전문 검색 인덱스가 생성되어야만 변경된 불용어 적용

<br>

### 8.5.2 전문 검색 인덱스의 가용성

- 전문 검색 인덱스 사용 조건(2가지)

  - 쿼리 문장이 전문 검색을 위한 문법 사용

    `MATCH ... AGAINST ...`

  - 테이블이 전문 검색 대상 컬럼에 대해서 전문 인덱스 보유

- 예시

  ```sql
  CREATE TABLE tb_test (
    doc_id INT,
    doc_body TEXT,
    PRIMARY KEY (doc_id),
    FULLTEXT KEY fx_docbody (doc_body) WITH PARSER ngram
  ) ENGINE = InnoDB;
  ```

  - 전문 검색 인덱스를 사용한 효율적 검색

    ```sql
    SELECT * FROM tb_test
    WHERE MATCH(doc_body) AGAINST('애플' IN BOOLEAN MODE);
    ```

  - table full scan을 통한 비효율적 검색

    ```sql
    SELECT * FROM tb_test WHERE doc_body LIKE '%애플%';
    ```

<br>



