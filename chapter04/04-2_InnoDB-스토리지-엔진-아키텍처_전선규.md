# 4. 아키텍처(InnoDB 스토리지 엔진)

> [4.2 InnoDB 스토리지 엔진 아키텍처](#4.2-InnoDB-스토리지-엔진-아키텍처)
>
> - [4.2.1 프라이머리 키에 의한 클러스터링](#4.2.1-프라이머리-키에-의한-클러스터링)
> - [4.2.2 외래 키 지원](#4.2.2-외래-키-지원)
> - [4.2.3 MVCC](#4.2.3-MVCC)
> - [4.2.4 잠금 없는 일관된 읽기(Non-Locking Consistent Read)](#4.2.4-잠금-없는-일관된-읽기(Non-Locking-Consistent-Read))
> - [4.2.5 자동 데드락 감지](#4.2.5-자동-데드락-감지)
> - [4.2.6 자동화된 장애 복구](#4.2.6-자동화된-장애-복구)
> - [4.2.7 InnoDB 버퍼 풀](#4.2.7-InnoDB-버퍼-풀)
>   - [4.2.7.1 버퍼 풀의 크기 설정](#4.2.7.1-버퍼-풀의-크기-설정)
>   - [4.2.7.2 버퍼 풀의 구조](#4.2.7.2-버퍼-풀의-구조)
>   - [4.2.7.3 버퍼 풀과 리두 로그](#4.2.7.3-버퍼-풀과-리두-로그)
>   - [4.2.7.4 버퍼 풀 플러시(Buffer Pool Flush)](#4.2.7.4-버퍼-풀-플러시(Buffer-Pool-Flush))
>     - Flush_list 플러시
>     - LRU_list 플러시
>   - [4.2.7.5 버퍼 풀 상태 백업 및 복구](#4.2.7.5-버퍼-풀-상태-백업-및-복구)
> - [4.2.8 Double Write Buffer](#4.2.8-Double-Write-Buffer)

<br>

## 4.2 InnoDB 스토리지 엔진 아키텍처

![](./images/sun_4-9.jpg)

- InnoDB는 MySQL에서 사용할 수 있는 스토리지 엔진 중 거의 유일하게 레코드 기반의 잠금 제공

  (높은 동시성 처리 가능, 안정적, 고성능)

### 4.2.1 프라이머리 키에 의한 클러스터링

- InnoDB 모든 테이블은 기본적으로 PK 기준으로 클러스터링 되어 저장

  - 이는 PK 값의 순서대로 디스크에 저장된다는 뜻

  - 모든 세컨더리 인덱스는 레코드의 주소 대신 PK 값을 논리적인 주소로 사용

  - PK 이용한 range scan은 상당히 빠르게 처리 가능

  - 쿼리 실행 계획에서 PK는 기본적으로 다른 보조 인덱스에 비해 비중이 높게 설정

    (다른 보조 인덱스보다 PK가 선택될 확률이 높음)

  - MySQL의 일반적 테이블 구조가 ORACLE DBMS IOT(Index organized table)와 동일

- MyISAM 스토리지 엔진에서는 클러스터링 키 미지원

  - PK와 세컨더리 인덱스는 구조적으로 차이가 없음

    (PK는 unique 제약을 가진 세컨더리 인덱스에 불과)

  - 모든 인덱스는 물리적 레코드의 주소 값(ROWID)을 가짐

<br>

### 4.2.2 외래 키 지원

- InnoDB 스토리지 엔진 레벨에서 외래키 지원

  (MyISAM 또는 MEMORY 테이블에서 사용 불가)

- 개발 환경의 데이터베이스에서는 좋은 가이드 역할 가능

- 주의점

  - DB 서버 운영의 불편함 때문에 서비스용 DB에서는 생성하지 않는 경우도 있음

  - 부모 테이블과 자식 테이블 모두 해당 컬럼에 인덱스 생성이 필요하고, 변경 시에는 반드시 부모 테이블이나 자식 테이블에 데이터가 있는지 체크하는 작업이 필요함

    - **위 작업은 lock이 여러 테이블로 전파되고, deadlock 발생 가능성을 높임**

  - 수동으로 데이터 적재나 스키마 변경 등의 관리 작업이 실패할 수 있음

    - 외래키가 복잡하게 얽힌 경우는 쉽지 않음

    - 긴급하게 조치해야 하는 경우 주의 필요

    - 문제 발생 시 `foreign_key_checks` 시스템 변수를 OFF 설정

      ```sql
      SET foreign_key_checks=OFF
      ```

      - 외래키 관계에 대한 체크 작업을 일시 중지
      - 외래키 관계의 부모 테이블에 대한 작업(ON DELETE CASCADE, ON UPDATE CASCADE 옵션)도 무시하게 됨
      - 적용 범위를 GLOBAL, SESSION 모두 설정 가능
        - 반드시 현재 작업 실행 세션에서만 기능을 OFF 해야 함
        - default SESSION `SET (SESSION) foreign_key_checks=OFF`
      - 외래키 체크를 일시 중지한 상태에서 부모 테이블 레코드 삭제했다면, 반드시 자식 테이블의 레코드도 삭제해서 반드시 일관성을 맞춰준 후 외래키 체크 기능을 다시 활성화시켜야 함

<br>

### 4.2.3 MVCC

> MVCC(Multi Version Concurrency Control)
>
> - 일반적으로 레코드 레벨의 트랜잭션을 지원하는 DBMS가 제공하는 기능
> - MVCC 가장 큰 목적은 <b>잠금 사용하지 않는 일관된 읽기</b>를 제공하는 데 있음
> - multi version이란 하나의 레코드에 대해 여러 개의 버전이 동시에 관리된다는 의미

##### 예시(테이블 데이터 변경 시)

- InnoDB는 언두 로그(Undo log)를 이용해 mvcc 기능 구현

- MySQL서버: Isolation level -> READ_COMMITTED

- INSERT 실행

  ```sql
  INSERT INTO member (id, name, area) VALUES (12, '홍길동', '서울');
  ```

  <img src="./images/sun_4-10.jpg" alt="drawing" width="50%" align="left" />

- UPDATE 실행

  ```sql
  UPDATE member SET area='경기' WHERE id=12;
  ```

  <img src="./images/sun_4-11.jpg" alt="drawing" width="50%" align="left" />

  - InnoDB 버퍼 풀

    - commit 실행 여부와 관계없이 새로운 값인 '경기'로 업데이트

  - 데이터 파일(디스크)

    - 체크포인트나 write thread에 의해 새로운 값으로 업데이트 되었을 수도 아닐 수도 있음

      (InnoDB가 ACID를 보장하므로 일반적으로는 버퍼 풀과 데이터 파일은 동일한 상태라고 가정해도 무방)

  - commit or rollback 실행 전 select 시

    **시스템 변수(transaction_isolation)에 설정된 격리 수준(Isolation level)에 따라 레코드를 조회하는 메모리 영역이 다름**

    - READ_UNCOMMITTED
      - InnoDB 버퍼 풀이 가지고 있는 변경 데이터를 읽어서 반환
    - READ_COMMITTED / REPEATABLE_READ / SERIALIZABLE
      - 변경 이전의 데이터 보관하고 있는 undo log 영역의 데이터 반환

- COMMIT or ROLLBACK 실행

  - commit 시 현재 상태를 영구적인 데이터로 만듦
  - rollback 시 undo log 영역에 있는 백업 데이터를 InnoDB 버퍼 풀로 복구하고, undo log 영역 내용을 삭제 (undo log 영역 데이터를 필요로 하는 트랜잭션이 더 이상 없을 경우 삭제)

- MVCC란 위와 같은 과정에서, 하나의 레코드에 대해 2개의 버전이 유지되고, 필요에 따라 어떤 데이터가 보여지는지 상황에 따라 달라지는 구조를 말함

- 트랜잭션이 길어지면 undo log에서 관리하는 예전 데이터가 삭제되지 못하고 오랫동안 관리되어야 해서, 자연히 undo log 영역에 저장되는 시스템 테이블스페이스의 공간이 많이 늘어나는 상황 발생 가능

<br>

### 4.2.4 잠금 없는 일관된 읽기(Non-Locking Consistent Read)

- InnoDB 스토리지 엔진은 MVCC 기술을 이용해 잠금을 걸지 않고 읽기 작업 수행
- SERIALIZABLE 격리 수준이 아닐 경우 insert와 연결되지 않은 순수 읽기(select) 작업은 다른 트랜잭션의 변경 작업과 관계없이 항상 잠금을 대기하지 않고 즉시 실행
- 주의점
  - 오랜 시간 활성 상태인 트랜잭션으로 인해 MySQL 서버가 느려지거나 문제가 발생할 때는, 일관된 읽기를 위해 undo log를 삭제하지 못하고 계속 유지하기 때문일 수 있음
  - 트랜잭션이 시작되었다면 가능한 빨리 롤백이나 커밋을 통해 트랜잭션을 완료하는 것이 좋음

<br>

### 4.2.5 자동 데드락 감지

- 내부적으로 잠금이 교착 상태에 빠지지 않았는지 체크하기 위해 잠금 대기 목록을 그래프(Wait-for List) 형태로 관리
- 데드락 감지 스레드를 가지고 있어 주기적으로 잠금 대기 그래프를 검사해 교착 상태에 빠진 트랜잭션을 찾아 하나를 강제 종료
  - undo log가 적은 트랜잭션을 강제 종료
- 일반적으로 데드락 감지 스레드가 데드락 찾는 작업은 부담되지 않음
  - 하지만 동시 처리 스레드가 매우 많아지거나 각 트랜잭션이 가진 잠금의 개수가 많으면 감지 스레드가 느려짐
  - 잠금 목록 검사 시 잠금 상태가 변경되지 않도록 잠금 목록이 저장된 리스트에 새로운 lock을 걸면, 서비스 쿼리를 처리 중인 스레드는 더는 작업 진행 못하여 서비스에 악영향 있을 수도
- 주의점
  - InnoDB 스토리지 엔진은 상위 레이어인 MySQL 엔진에서 관리되는 테이블 잠금(LOCK TABLES 명령으로 잠긴 테이블) 볼 수 없어 데드락 감지가 불확실할 수도 있음
    - `innodb_table_locks` 시스템 변수 활성화 시 테이블 레벨의 잠금까지 활성화 가능(가급적 활성화 권고)
  - 동시 처리 스레드가 매우 많아지거나 각 트랜잭션이 가진 잠금의 개수가 많으면 감지 스레드가 느려짐
    - 일반적으로 데드락 감지 스레드가 데드락 찾는 작업은 부담되지 않음
    - 잠금 목록 검사 시 잠금 상태가 변경되지 않도록 잠금 목록이 저장된 리스트에 새로운 lock을 걸 때, 서비스 쿼리를 처리 중인 스레드는 더는 작업 진행 못하여 서비스에 악영향 있을 수도 있음
    - 데드락 미감지 필요 + timeout 설정
      - `innodb_deadlock_detect=OFF` 
      - `innodb_lock_wait_timeout=50`(50 보다 낮은 시간으로 변경해서 사용)

<br>

### 4.2.6 자동화된 장애 복구

> InnoDB에는 손실 및 장애로부터 데이터 보호를 위해 여러 매커니즘 탑재
>
> MySQL 서버 시작 시 완료되지 못한 트랜잭션이나 디스크에 일부만 기록된(partial write) 데이터 페이지 등에 대한 복구 작업 자동 진행
>
> 서버 시작 시 항상 자동 복구 수행하지만, 자동 복구할 수 없는 손상이 있다면 자동 복구를 멈추고 서버를 종료시킴

##### 자동 복구 미작동 시

- `innodb_force_recovery` 시스템 변수 설정(1~6)

  - 해당 설정값으로 손상 여부 검사 과정을 선별적으로 진행함
  - 값이 클수록 심각한 손상으로, 복구 가능성이 낮아짐
  - 0이 아닌 복구 모드에서는 select 외 insert, update, delete 쿼리 수행 불가

- 복구 프로세스

  - InnoDB 로그 파일이 손상되었다면 6

  - InnoDB 테이블의 데이터 파일이 손상되었다면 1

  - 어떤 부분의 손상인지 모르면 1~6으로 변경해가면서 시작해봄

  - 그래도 서버 재시작 안되면

    - 백업 후 서버 구축

    - 바이너리 로그로 최대한 장애 시점까지의 데이터 복구

      (풀 백업 보다는 바이너리 로그로 복구하는 것이 데이터 손실이 더 적을 수 있음)

##### innodb_force_recovery

- 1(SRV_FORCE_**IGNORE_CORRUPT**)
  - 테이블스페이스의 데이터나 인덱스 페이지 손상 발견되어도 무시
  - 에러 로그에 'Database page corruption on disk or a failed' 메시지 출력 시 설정
  - mysqldump 프로그램 또는 SELECT INTO OUTFILE... 명령으로 덤프하여 데이터베이스 재구축 필요

- 2(SRV_FORCE_**NO_BACKGROUND**)

  - 메인 스레드 시작하지 않고 서버 시작
  - tx commit 후 불필요 undo log 삭제(Undo purge) 시, InnoDB 메인 스레드 장애 발생하는 경우

- 3(SRV_FORCE_**NO_TRX_UNDO**)

  - commit 되지 않은 트랜잭션 작업을 롤백하지 않음
  - mysqldump 필요
  - 정상 프로세스의 경우
    - 롤백에 대비해 변경 전 데이터를 undo log에 기록
    - 서버 시작 시 undo log 데이터를 데이터 파일에 적용
    - redo log 내용을 다시 덮어써서 장애 시점의 데이터 상태를 만듦
    - 트랜잭션 롤백 수행(3 레벨 설정 시 해당 단계 생략)

- 4(SRV_FORCE_**NO_IBUF_MERGE**)

  - insert buffer 내용 무시하고 강제 서버 시작
  - insert buffer는 실제 데이터와 관련된 것이 아닌 인덱스와 관련된 부분이므로 dump 후 db 재구축하면 데이터 손실 없이 복구 가능
  - 정상 프로세스의 경우
    - insert, update, delete 등의 데이터 변경으로 인한 인덱스 변경 작업 즉시 처리하기도, insert buffer에 저장해두고 나중에 처리할 수도 있음
    - insert buffer에 기록된 내용은 언제 데이터 파일에 병합(merge)될지 알 수 없음
    - 서버 재시작 시 insert buffer 손상 감지하면 에러 발생

- 5(SRV_FORCE_**NO_UNDO_LOG_SCAN**)

  - undo log 무시하고 서버 시작
  - 종료 시점 커밋되지 않은 작업도 모두 커밋된 것처럼 처리
  - mysqldump 필요

- 6(SRV_FORCE_**NO_LOG_REDO**)

  - redo log 무시하고 서버 시작

  - commit 되었더라도 redo log에만 기록되고 데이터 파일에 기록되지 않은 데이터는 모두 무시

    (즉, 마지막 체크포인트 시점의 데이터만 남게 됨)

  - 기존 리두 로그 모두 삭제(또는 백업) 후 서버 재시작 시 서버가 자동으로 파일 생성
  - mysqldump 필요

<br>

### 4.2.7 InnoDB 버퍼 풀

> InnoDB 스토리지 엔진에서 가장 핵심적인 부분
>
> - 디스크 데이터 파일, 인덱스 정보를 메모리에 캐시해 두는 공간
> - 쓰기 작업을 지연시켜 일괄 작업을 처리할 수 있게 해주는 버퍼 역할도 함

##### 4.2.7.1 버퍼 풀의 크기 설정

변경 전 [매뉴얼](https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool-resize.html)을 참고

```sql
innodb_buffer_pool_size=20G  --메모리 상황에 맞게 할당
```

- 운영체제와 각 클라이언트 스레드가 사용할 메모리 고려하여 설정하여야 함

- InnoDB 버퍼 풀의 크기를 동적으로 조절하는 방법

  - 운영체제 전체 메모리 공간이 8GB 미만

    - 50%를 InnoDB 버퍼 풀로 할당

      (나머지 공간은 MySQL 서버, 운영체제, 다른 프로그램 공간으로 확보)

  - 8GB 이상

    - 전체 메모리의 50%에서 시작해 조금씩 올려가면서 최적점을 찾아야 함

  - 50GB 이상

    - 15GB ~ 30GB를 운영체제 및 다른 프로그램에 할당하고 이외를 버퍼풀로 할당

- 버퍼풀 변경은 크리티컬한 변경이므로 서버가 한가한 시점에 하는 것이 좋음

  - 버퍼 풀을 줄이는 변경이 시스템 영향도가 더 큼

    (가능하면 줄이는 변경은 하지 않는 것이 좋음)

- 버퍼풀은 내부적으로 128MB 청크 단위로 쪼개어 관리되므로 해당 단위로 크기를 조정

  - 버퍼 풀을 여러 개로 쪼개어 관리하면서, 버퍼 풀 전체를 관리하는 잠금(semaphore) 경합도 분산됨

- 버퍼 풀 인스턴스

  ```sql
  innodb_buffer_pool_instances=8
  ```

  - default 8개로 초기화
  - 버퍼 풀 메모리 크기가 1GB 미만이면 버퍼 풀 인스턴스는 1개만 생성
  - 메모리 공간 40GB 이하 수준이면 8개 유지하고, 그 이상이면 인스턴스당 5GB 정도로 갯수 조정

##### 4.2.7.2 버퍼 풀의 구조

- 버퍼 풀이라는 거대한 메모리 공간을 페이지 크기(`innodb_page_size`) 조각으로 쪼개어 관리

- 스토리지 엔진이 데이터를 필요로 할 때 해당 데이터 페이지를 읽어서 각 조각에 저장

- 페이지 크기 조각 관리 자료구조

  - **LRU(Least Recently Used) / MRU(Most Recently Used) 리스트**

    <img src="./images/sun_4-13.jpg" alt="drawing" width="50%" align="left" />

    - LRU & MRU 리스트가 결합된 형태

    - 'Old sublist' 영역이 LRU, 'New sublist' 영역이 MRU

    - LRU 리스트 관리 목적

      - 디스크로부터 한 번 읽어온 페이지를 최대한 오랫동안 InnoDB 버퍼 풀의 메모리에 유지해서 디스크 읽기를 최소화

    - InnoDB 스토리지 엔진에서 데이터를 찾는 과정

      1. 필요한 레코드가 저장된 '데이터 페이지' 버퍼 풀에 있는지 검사

         1-1. InnoDB 어댑티브 해시 인덱스 이용해 페이지 검색

         1-2. 해당 테이블의 인덱스(B-Tree) 이용해 버퍼 풀에서 페이지 검색

         1-3. 버퍼 풀에 이미 데이터 페이지가 있었다면 해당 페이지의 포인터를 MRU 방향으로 승급

      2. 필요한 데이터페이지를 디스크에서 가져와 버퍼 풀에 적재하고, 적재된 페이지에 대한 포인터를 LRU Header에 추가

      3. LRU head 부분에 적재된 데이터 페이지가 실제로 읽히면 MRU header 부분으로 이동

         (Read Ahead와 같은 대량 읽기의 경우 디스크의 데이터 페이지가 버퍼 풀로 적재는 되지만, 실제 쿼리에서 사용되지는 않을 수 있고, 이 때는 MRU로 이동하지 않음)

      4. 버퍼 풀에 상주하는 데이터 페이지는 사용자 쿼리가 얼마나 최근에 접근했었는지에 따라 나이(age) 부여

         aging 데이터 페이지는 버퍼 풀에서 제거(eviction)

         데이터 페이지가 쿼리에 의해서 사용되면 age 초기화 및 MRU header로 옮겨짐

         `즉, 버퍼 풀 내부에서 최근 접근 여부에 따라 데이터 페이지는 서로 경쟁하며 MRU or LRU로 이동함. InnoDB 스토리지 엔진은 LRU 끝으로 밀려난 데이터 페이지를 버퍼 풀에서 제거해 새로운 데이터 페이지를 적재할 수 있는 빈 공간 준비`

      5. 필요한 데이터가 자주 접근됐다면 해당 페이지 인덱스 키를 어댑티브 해시 인덱스에 추가

  - **Flush 리스트**

    - 디스크로 동기화되지 않은 데이터를 가진 데이터 페이지(dirty page)의 변경 시점 기준 페이지 목록 관리

    - 데이터 변경이 가해진 데이터 페이지는 flush 리스트에서 관리

      (데이터 변경이 없다면 관리되지 않음)

    - 프로세스

      - 데이터 변경 시 redo log에 기록

      - 버퍼 풀의 데이터 페이지에도 변경 내용 반영

        (redo log의 각 엔트리는 특정 데이터 페이지와 연결)

      - 예외

        일반적으로 (redo log -> data page -> disk)이지만, 체크포인트(리두 로그의 어느 부분부터 복구 실행할지 판단하는 기준) 발생 시 redo log가 데이터 페이지 기준으로 변경되기도 함

  - **Free 리스트**

    - 버퍼 풀에서 실제 사용자 데이터로 채워지지 않은 비어 있는 페이지 목록
    - 사용자 쿼리가 새롭게 디스크의 데이터 페이지를 읽어올 때 사용

##### 4.2.7.3 버퍼 풀과 리두 로그

<img src="./images/sun_4-14.jpg" alt="drawing" width="50%" align="left" />

- **buffer pool**

  - InnoDB 버퍼 풀과 redo log는 매우 밀접한 관계

    - InnoDB 버퍼 풀은 서버 메모리가 허용하는 만큼 크게 설정할수록 쿼리 성능 빨라짐

      (디스크의 데이터가 버퍼 풀 메모리로 적재를 많이 될수록)

    - 하지만 버퍼 풀 메모리 공간만 단순히 늘리는 것은 데이터 캐시 기능만 향상시킴

  - InnoDB 버퍼 풀의 DB 성능 향상

    - data cache
    - write buffering

  - 버퍼 풀의 데이터 페이지
    - Clean Page: 디스크에서 읽은 상태로 전혀 변경되지 않은 페이지
    - Dirty Page: insert, update, delete 명령으로 변경된 데이터를 가진 페이지(버퍼 풀에 무한정 머무를 수 없음)

- **redo log**

  - 1개 이상의 고정 크기 파일을 연결해서 순환 고리처럼 사용

    (데이터 변경이 계속 발생하면 리두 로그 파일에 기록됐던 로그 엔트리는 어느 순간 다시 새로운 로그 엔트리로 덮어 써짐)

  - 따라서 redo log 파일에서 재사용 가능한 공간과 당장 재사용 불가능한 공간을 구분해서 관리

    - 재사용 불가능한 공간(Active Redo Log)

      (위 그림에서 화살표를 가진 엔트리)

  - LSN(Log Sequence Number)

    - 재사용되어 기록될 때마다 로그 포지션이 계속 증가된 값을 갖게 되는 것

- **checkpoint**

  - Checkpoint Event

    - redo log & buffer pool의 dirty page를 디스크로 동기화

      *checkpoint LSN보다 작은 redo log entry 및 연관 dirty page는 모두 디스크로 동기화*

    - 최근 체크포인트 지점의 LSN이 활성 리두 로그 공간의 시작점이 됨

  - Checkpoint Age

    - 최근 체크포인트의 LSN과 마지막 리두 로그 엔트리의 LSN 차이
    - 활성 리두 로그 공간의 크기가 됨

- 예제

  - buffer pool 100GB / redo log 100MB

    - redo log가 100MB 이므로 checkpoint age도 최대 100MB만 허용
    - 예를 들어 평균 redo log entry가 4KB면 25600개(100MB/4KB) dirty page만 버퍼 풀에 보관
    - 데이터 페이지가 16KB라고 가정하면, 허용 가능한 전체 dirty page 크기는 400MB 수준밖에 안되는 것
    - 이 경우 버퍼 풀의 크기는 매우 크지만, <u>실제 쓰기 버퍼링을 위한 효과는 거의 못 보는 상황</u>

  - buffer pool 100MB / redo log 100GB

    - 위와 같은 계산에서 400GB dirty page 가질 수 있지만, 버퍼 풀의 크기가 100MB이므로 최대 허용 dirty page도 100MB

  - 위 두 사례 모두 잘못된 설정

    - write buffering 성능도 같이 올리기 위해선 이론적으로 redo log 크기를 크게 가져가는 것이 좋지만 단점도 있음

    - 갑작스러운 디스크 쓰기가 발생할 가능성이 높음

      (버퍼 풀에 더티 페이지 비율이 너무 높은 상태에서 갑자기 버퍼 풀이 필요해지면 InnoDB 스토리지 엔진은 매우 많은 더티 페이지를 한 번에 기록해야 함)

    - <u>버퍼 풀의 크기가 100GB 이하</u> MySQL 서버에서는 <u>redo log 전체 크기를 대략 5~10GB 수준</u>으로 선택하고, 필요 시 조금씩 늘려 최적값을 선택

      (일반적으로 redo log는 변경분만 가지고, buffer pool은 데이터 페이지를 통째로 가지므로 redo log가 더 작은 공간만 있어도 됨)

##### 4.2.7.4 버퍼 풀 플러시(Buffer Pool Flush)

> InnoDB 스토리지 엔진은 버퍼 풀에서 아직 디스크로 기록되지 않은 dirty page들을 성능상의 악영향 없이 디스크에 동기화하기 위해 2개(Flush_list, LRU_list) 플러시 기능을 백그라운드로 실행

- **Flush_list **플러시

  - 오래된 redo log 공간이 지워지려면 반드시 buffer pool의 dirty page가 먼저 디스크로 동기화되어야 함

  - 이를 위해 주기적으로 Flush_list 플러시 함수를 호출해 플러시 리스트에서 오래전에 변경된 데이터 페이지 순서대로 디스크에 동기화 작업 수행(dirty page 양에 따라 성능 결정)

  - 시스템 변수

    - `innodb_page_cleaners`

      - cleaner thread: dirty page 디스크 동기화 스레드

      - 해당 스레드 개수 조정 변수

      - 해당 설정 값이 버퍼 풀 인스턴스 개수보다 많은 경우에는 해당 설정값으로 자동 변경

        (더 적은 경우 하나의 클리너 스레드가 여러 개의 버퍼 풀 인스턴스 처리하므로 인스턴스 개수와 동일하게 설정 권고)

    - `innodb_max_dirty_pages_pct`

      - 버퍼 풀은 dirty page를 90%까지 가질 수 있음
      - 해당 비율 조정 변수
      - 일반적으로 버퍼 풀이 dirty page를 많이 가질 경우 write buffering을 통해 디스크 쓰기 작업 횟수를 줄여 성능을 높이므로, 해당 설정값 기본값 유지 권고

    - `innodb_max_dirty_pages_pct_lwm`

      - 버퍼 풀에 dirty page가 많을수록 Disk IO Burst 현상 발생 가능성 높음
      - 일정 수준 이상 dirty page 발생 시 조금씩 버퍼 풀의 dirty page를 disk write
      - 기본값 10% 수준인데, 만약 비율이 얼마 되지 않은 상태에서 디스크 쓰기가 많이 발생하고, dirty page 비율이 너무 낮은 상태로 계속 머무르면 해당 변수를 조금 더 높은 값으로 설정

    - `innodb_io_capacity(_max)=1000(5000)`

      - DB 서버에서 디스크 read/write capacity 설정

      - 해당 값이 1000이라고해서 초당 1000번의 디스크 쓰기가 보장되지 않음

        (내부적으로 최적화 알고리즘이 있어서 설정값 기준으로 적당히 계산된 횟수만큼 dirty page write 실행)

    - `innodb_adaptive_flushing=ON`

      - default ON
      - 해당 기능 활성화 시 스토리지 엔진은 단순히 버퍼 풀의 더티 페이지 비율이나 io_capacity 설정값에 의존하지 않고 새로운 알고리즘 사용
      - redo log의 증가 속도를 분석해 적절 수준의 dirty page가 버퍼 풀에 유지될 수 있도록 디스크 쓰기 실행

    - `innodb_adaptive_flushing=10`

      - default 10(%)
      - 전체 리두 로그 공간에서 활성 리두 공간이 10% 미만이면 adaptive flush 기능 동작하지 않다가, 10% 넘어가면 동작

    - `innodb_flush_neighbors=OFF`

      - default OFF
      - dirty page disk write 시에 근접 dirty page를 디스크에 같이 쓰는 설정
      - HDD에서는 필요가 있었으나, SDD에서는 기본값 OFF로 유지하는 것을 권고

- **LRU_list** 플러시

  - LRU 리스트에서 사용 빈도가 낮은 데이터 페이지 제거하여 새로운 페이지 읽어올 공간을 만드는 함수
  - `innodb_lru_scan_depth=8`
    - LRU 리스트 끝부분부터 설정 개수만큼 페이지 스캔
    - 스캔 시 dirty page는 디스크 동기화하며, 클린 페이지는 즉시 Free list로 페이지를 옮김
    - 버퍼 풀 인스턴스별 최대 scan depth 개수만큼 스캔하므로, 실질적 개수는 buffer_pool_instances * lru_scan_depth

##### 4.2.7.5 버퍼 풀 상태 백업 및 복구

- Warming Up

  - 디스크의 데이터가 버퍼 풀에 적재되어 있는 상태

  - 서버 셧다운 후 쿼리 처리 성능이 떨어지는 것은 버퍼 풀에 데이터가 없기 때문

    (warming up 되어 있지 않으면 최대 몇십 배 성능 차이 발생)

- 버퍼 풀 덤프 및 적재 기능

  - 5.6 버전부터 도입
  - 시스템 변수
    - `innodb_buffer_pool_dump_now=ON` 서버 셧다운 전 버퍼 풀 상태 백업
    - `innodb_buffer_pool_load_now=ON` 서버 재시작 후 백업 버퍼 풀 상태 복구

- ib_buffer_pool 파일로 생성

  - 버퍼 풀이 아무리 커도 몇십 MB 이하로 유지

  - 이는 버퍼 풀의 LRU 리스트에 적재된 데이터 페이지의 메타 정보만 가져와서 저장하기 때문

  - 백업 자체는 빨리 완료되나, 복구 과정은 버퍼 풀 크기에 따라 상당한 시간이 걸릴 수도 있음

    (백업 데이터를 디스크에서 읽어와야 하므로)

  - 버퍼 풀 복구 진행상태 확인

    ```sql
    SHOW STATUS LIKE 'Innodb_buffer_pool_dump_status';
    ```

  - 버퍼 풀 적재 작업 중단

    ```sql
    SET GLOBAL innodb_buffer_pool_load_abort=ON;
    ```

    - 복구 실행 중 상태에서 서비스 재개는 좋지 않지만, 급하게 해야 한다면 활성화

- 백업과 복구 자동화

  `innodb_buffer_pool_dump_at_shutdown=ON`

  `innodb_buffer_pool_load_at_startup=ON`

- 테이블 전체(인덱스 포함) 페이지 중에서 대략 어느 정도 비율이 InnoDB 버퍼 풀에 적재돼 있는지 추측할 수도 있음

<br>

### 4.2.8 Double Write Buffer

<img src="./images/sun_4-15.jpg" alt="drawing" width="50%" align="left" />

- 문제(Partial-page or Torn-page)
  - redo log 공간 낭비 막기 위해 페이지 변경 부문만 기록
  - dirty page를 디스크 파일로 flush 할 때 일부만 기록되는 문제 발생하면, 그 페이지는 복구할 수 없을 수도 있음
  - 하드웨어 오작동 또는 시스템 비정상 종료 등으로 발생 가능

- 해결(Double-Write 기법)

  - 위 그림에서 'A' ~ 'E'까지의 dirty page를 디스크로 플러시 가정

  - 실제 데이터 파일 변경 내용 기록 전 dirty page 들을 묶어 한 번의 디스크 쓰기로 시스템 테이블 스페이스의 DoubleWrite 버퍼에 기록

  - 이후 각 dirty page를 파일의 적당한 위치에 하나씩 랜덤 쓰기 실행

  - DoubleWrite 버퍼의 내용은 실제 데이터 파일의 쓰기가 중간에 실패할 때만 사용

    - 재시작 시 항상 DoubleWrite 버퍼 내용과 데이터 파일의 페이지들을 모두 비교
    - 페이지 내용이 다르면 DoubleWrite 버퍼의 내용을 데이터 파일의 페이지로 복사

  - 시스템 변수

    `innodb_doublewrite=ON`

- 기타

  - DoubleWrite 버퍼는 데이터 안정성을 위해 사용

  - HDD처럼 자기 원판(Platter)이 회전하는 저장 시스템에서는 한 번의 순차 디스크 쓰기이므로 별로 부담되지 않지만, SSD처럼 랜덤 IO나 순차 IO의 비용이 비슷한 저장 시스템에서는 상당히 부담스러움

  - 데이터 무결성이 매우 중요한 서비스에서는 DoubleWrite 활성화 고려

  - 만약 DB 서버 성능을 위해 InnoDB 리두 로그 동기화 설정`innodb_flush_log_at_trx_commit`을 1이 아닌 값으로 설정했다면 DoubleWrite도 비활성화하는 것이 좋음

    (리두 로그는 동기화하지 않으면서 DoubleWrite만 비활성화하는 것은 맞지 않음)