### 4.2.9 언두 로그
- **<u>높은 동시성 제공(트랜잭션 보장, 격리 수준 보장) / 백업 데이터</u>**
- 트랜잭션 롤백 시 변경된 데이터를 변경 전 데이터로 복구하기 위해 언두 로그 백업데이터 사용
- 특정 커넥션에서 데이터 변경 도중 다른 커넥션에서 데이터를 조회하면 트랜잭션 격리 수준에 맞게 변경 중인 레코드를 읽지 않고 언두 로그를 활용하기도 함 (격리 수준)
- 관리 비용↑

#### 4.2.9.1 언두 로그 모니터링
- 언두 영역 : 데이터 변경 시 변경 전 데이터를 보관하는 곳
- 언두 로그 공간↑ -> 디스크 사용량↑ -> 쿼리 성능 저하 가능성↑
- 언두 로그 공간 문제점은 5.7버전부터 해결됨 (현재는 디스크 공간을 순차적, 자동으로 줄일 수 있음)

```
// 언두 로그 건수 조회
// 트랜잭션이 장시간 유지되는 것은 좋지 않으므로 언두 로그 모니터링이 필요함
mysql> SELECT count FROM information_schema.innodb_metrics WHERE SUBSYSTEM='transaction' AND NAME = 'trx_rseg_history_len';
```

#### 4.2.9.2 언두 테이블 스페이스 관리
- 언두 테이블 스페이스 : 언두 로그 저장 공간
- 언두 테이블스페이스 변화 과정
	1. 5.6버전 이전 : 시스템 테이블스페이스(ibdata.ibd)에 저장 -> 확장의 한계(서버가 초기화될 때 생성되어서)
	2. 5.6버전 : innodb_undo_tablespaces 시스템 변수 도입 -> innodb_undo_tablespaces를 0으로 설정하면 시스템 테이블 스페이스에 저장되는 문제
	3. 8.0버전 : innodb_undo_tablespaces를 사용하지 않고 언두로그는 항상 시스템 테이블스페이스 외부 별도 로그 파일에 기록되도록 개선
- 구성 : 1개 이상 128개 이하의 롤백 세그먼트를 가지고, 롤백 세그먼트는 1개 이상의 언두 슬롯을 가짐
![](https://i.esdrop.com/d/f/5BqG1Oh0zF/vgSmwy5Mkl.png "언두 테이블스페이스 구조")
	- 하나의 롤백 세그먼트가 가지는 언두 슬롯 수 = (InnoDB 페이지 크기) / 16byte
	- 최대 동시 트랜잭션 수 = (InnoDB 페이지 크기) / 16 * (롤백 세그먼트 개수) * (언두 테이블스페이스 개수)
- 8.0버전부터 새로운 언두 테이블 스페이스를 동적으로 추가하고 삭제 가능
- Undo tablespace truncate : 언두 테이블 스페이스 공간을 필요한 만큼만 남기고 불필요하거나 과도하게 할당된 공간을 운영체제로 반납
	1. 자동 모드
		InnoDB 스토리지 엔진의 퍼지 스레드는 주기적으로 언두 로그 공간에서 커밋된 데이터 등 불필요해진 언두 로그를 삭제하는 작업을 실행(언두 퍼지 작업)
		언두 퍼지 작업 활성화 : innodb_undo_log_truncate = ON
		덜 빈번하게 실행하려면 : innodb_purge_rseg_truncate_frequency
	2. 수동 모드
		언두 테이블스페이스가 최소 3개 이상은 되어야 작동
		innodb_undo_log_truncate = OFF인 경우, 언두 퍼지 작업이 부진한 경우 언두 테이블스페이스를 비활성화해서 언두 테이블스페이스가 더 사용되지 않도록 설정 -> 퍼지 스레드가 비활성 상태의 언두 테이블스페이스 탐색 후 삭제 -> 운영체제로 반납 완료 후 언두 테이블스페이스 활성화
		
```
// 언두 테이블스페이스 비활성화
mysql> ALTER UNDO TABLESPACE tablespace_name SET INACTIVE;
// 퍼지 스레드에 의해 언두 테이블스페이스 공간이 반납되면 다시 활성화
mysql> ALTER UNDO TABLESPACE tablespace_name SET ACTIVE;
```


### 4.2.10 체인지 버퍼
- 체인지 버퍼 : 인덱스 업데이트 작업에 필요한 인덱스 페이지를 저장해둘 임시 공간
- 유니크 인덱스는 체인지 버퍼 사용 불가능(중복여부를 체크해야 하므로)
- 버퍼 머지 스레드 : 체인지 버퍼에 임시로 저장된 인덱스 레코드 조각을 병합하는 작업 수행
- innodb_change_buffering 설정 값
	- all : 모든 인덱스 관련 작업(inserts deletes + purges)을 버퍼링
	- none : 버퍼링 안함
	- inserst : 인덱스에 새로운 아이템 추가 작업만 버퍼링
	- deletes : 인덱스에서 기존 아이템 삭제하는 작업(삭제됐다는 마킹 작업)만 버퍼링
	- changes : 인덱스에 추가하고 삭제하는 작업(inserts + deletes)만 버퍼링
	- purges : 인덱스 아이템을 영구적으로 삭제하는 작업만 버퍼링(백그라운드 작업)
- 사용 가능한 공간은 InnoDB 버퍼 풀로 설정된 메모리 공간의 25%(기본)에서 최대 50%까지

```
// 체인지 버퍼가 사용중인 메모리 공간의 크기
mysql > SELECT EVENT_NAME, CURRENT_NUMBER_OF_BYTES_USED FROM performance_schema.memory_summary_global_by_event_name WHERE EVENT_NAME ='memory/innodb_ibuf0ibuf';
// 체인지 버퍼 관련 오퍼레이션 처리 횟수
mysql> SHOW ENGINE INNODB STATUS \G
```


### 4.2.11 리두 로그 및 로그 버퍼
- 리두 로그 : 영속성(Durable)과 관련있으며 서버가 비정상적으로 종료됐을 때 데이터 파일에 기록되지 못한 데이터를 잃지 않게 해주는 안전장치
- 거의 모든 DBMS에서 데이터 파일은 쓰기보다 읽기 성능을 고려한 자료 구조를 가짐 -> 데이터 파일 쓰기는 디스크 랜덤 액세스 필요 -> 쓰기 작업은 큰 비용이 필요 -> 성능 저하를 막기 위해 쓰기 비용이 낮은 리두 로그를 활용
- 트랜잭션이 커밋돼도 데이터 파일은 즉시 디스크로 동기화되지 않지만 리두 로그(트랜잭션 로그)는 항상 디스크로 기록됨
- 서버가 비정상 종료되는 경우 발생하는 일관되지 않은 데이터
	1. 커밋됐지만 데이터 파일에 기록되지 않은 데이터
		리두 로그에 저장된 데이터를 데이터 파일에 다시 복사하면 해결
	2. 롤백됐지만 데이터 파일에 이미 기록된 데이터
		리두 로그로 해결 불가능, 변경되기 전 데이터를 가진 언두 로그의 내용을 가져와 데이터 파일에 복사
		트랜잭션의 상태만 리두 로그로 확인
- 리두 로그는 트랜잭션이 커밋되면 즉시 디스크로 기록되도록 시스템 변수를 설정하는 것을 권장
- 트랜잭션 커밋마다 리두 로그를 디스크에 기록하면 부하가 생기므로 innodb_flush_log_at_trx_commit(디스크 동기화 결정) 변수 활용
	- innodb_flush_log_at_trx_commit = 0 : 1초에 한 번씩 리두 로그를 디스크로 기록(write)하고 동기화(sync) 실행 (최대 1초 동안의 트랜잭션은 커밋됐더라도 해당 트랜잭션에서 변경된 데이터는 사라질 수 있음)
	- innodb_flush_log_at_trx_commit = 1 : 매번 트랜잭션이 커밋될 때마다 디스크로 기록되고 동기화 수행
	- innodb_flush_log_at_trx_commit = 2 : 매번 트랜잭션이 커밋될 때마다 디스크로 기록되지만 동기화는 1초에 한 번 실행 (서버가 비정상 종료됐더라도 운영체제가 정상작동 한다면 해당 트랜잭션의 데이터가 사라지지 않음)
- 전체 리두 로그 파일의 크기 = (innodb_log_files_in_group) * (innodb_log_file_size) 	

#### 4.2.11.1 리두 로그 아카이빙
- MySQL 엔터프라이즈 백업, Xtrabackup 툴 : 데이터 파일을 복사하는 동안 InnoDB 스토리지 엔진의 리두 로그에 쌓인 내용을 계속 추적하면서 새로 추가된 리두 로그 엔트리를 복사 -> 복사하는 동안 추가된 리두 로그 엔트리가 백업되지 않으면 복사된 데이터 백업 파일은 일관된 상태를 유지하지 못함
- 아카이빙 시작한 세션이 종료전에 연결이 끊어지면 InnoDB 스톨지ㅣ 엔진이 리두 로그 아카이빙을 멈추고 아카이빙 파일도 자동으로 삭제함
- 아카이빙된 리두 로그를 정상 사용하기 위해서는 커넥션을 그대로 유지해야 하고, 작업 완료 시 UDF를 호출해서 정상종료 해야한다
```
// 리두 로그 아카이빙 사용
// 운영체제의 MySQL 서버를 실행하는 유저만 접근 가능
linux> mkdir /var/log/mysql_redo_archive
linux> cd /var/log/mysql_redo_archive
linux> mkdir 20200722
linux> chmod 700 20200722

mysql> SET GLOBAL innodb_redo_log_archive_dirs='backup:/var/log/mysql_redo_archive';
// 리두 로그 아카이빙을 시작하도록 UDF(사용자 정의 함수)를 실행
mysql> DO innodb_redo_log_archive_start('backup', '20200722');
// 리두 로그 아카이빙 종료
mysql> DO innodb_redo_log_archive_stop();
```

#### 4.2.11.2 리두 로그 활성화 및 비활성화
- MySQL 서버는 재시작할 때 리두 로그에서 데이터 파일에 기록되지 못한 데이터가 있는지 검사하는데 리두 로그 비활성화되고 비정상 종료되면 리두 로그를 통한 복구가 불가능하므로 주의해야 한다
```
mysql> ALTER INSTANCE DISABLE INNODB REDO_LOG;
// 리두 로그를 비활성화한 후 대량 데이터 적재 실행
mysql> LOAD DATA...
mysql> ALTER INSTANCE ENABLE INNODB REDO_LOG;
// 리두 로그 활성화여부 확인
mysql> SHOW GLOBAL STATUS LIKE 'Innodb_redo_log_enabled';
```

### 4.2.12 어댑티브 해시 인덱스
- 일반적인 '인덱스' = 테이블에 사용자가 생성해둔 B-Tree 인덱스
- 어댑티브 해시 인덱스 = InnoDB 스토리지 엔진에서 사용자가 자주 요청하는 데이터에 대해 자동으로 생성하는 인덱스
- innodb_adaptive_hash_index로 기능 활성화, 비활성화 변경 가능
- 자주 읽히는 데이터 페이지의 키 값을 이용해 해시 인덱스를 만들고 필요할 때마다 어댑티브 해시 인덱스를 검색해서 레코드가 저장된 페이지를 즉시 찾아갈 수 있음
	검색 시간↓, 컴퓨터는 더 많은 쿼리를 동시실행할 수 있음
- 해시 인덱스 : 인덱스 키 값(B-Tree 인덱스 고유번호(id), B-Tree 인덱스 실제 키 값) + 데이터 페이지 주소(실제 키 값이 저장된 데이터 페이지의 메모리 주소 = InnoDB 버퍼 풀에 로딩된 페이지의 주소)
- 모든 B-Tree 인덱스에 대한 어댑티브 해시 인덱스가 하나의 해시 인덱스에 저장 (B-Tree 인덱스 : 어댑티브 해시 인덱스 : 해시 인덱스 = 1 : 1 : 1)
- 어댑티브 해시 인덱스는 버퍼 풀에 올려진 데이터 페이지에 대해서만 관리됨 (버퍼 풀에서 해당 데이터 페이지가 없어지면 어댑티브 해시 인덱스에서도 사라짐)
- 어댑티브 해시 인덱스의 성능 효과
	초당 실행 가능 쿼리 수↑, CPU사용률↓, InnoDB 내부잠금(세마포어)↓
- 어댑티브 해시 인덱스가 하나의 메모리 객체인 이유로 인덱스 경합이 심했음 -> 인덱스의 파티션 기능 제공
- innodb_adaptive_hash_index_parts : 파티션 개수 변경(기본값 8개)
- 어댑티브 해시 인덱스가 성능 향상에 도움이 되지 않는 경우
	- 디스크 읽기가 많은 경우 -> 어댑티브 해시 인덱스가 페이지를 메모리(버퍼 풀) 내에서 접근하는 것을 빠르게 만드는 기능이기 때문
	- 특정 패턴의 쿼리가 많은 경우(조인, LIKE 패턴 검색)
	- 매우 큰 데이터를 가진 테이블의 레코드를 폭 넓게 읽는 경우
- 어댑티브 해시 인덱스가 성능 향상에 도움되는 경우
	- 디스크의 데이터가 InnoDB 버퍼 풀 크기와 비슷한 경우(디스크 읽기가 많지 않은 경우)
	- 동등 조건 검색(동등 비교와 IN 연산자)이 많은 경우
	- 쿼리가 데이터 중에서 일부 데이터에만 집중되는 경우
- 테이블 삭제와 변경 시에 상당히 많은 CPU 자원을 사용하고, DB 서버 처리 성능이 느려짐

```
// 어댑티브 해시 인덱스가 도움이 되는지 확인하기 위해 MySQL 서버의 상태값을 살펴보면 좋다
// 어댑티브 해시 인덱스가 비활성화라면 hash searches/s 의 값이 0으로 표시됨
// 해시 인덱스 사용률이 28%고 서버의 CPU 사용량이 100%에 근접하다면 어댑티브 해시 인덱스가 효율적이라고 볼 수 있음
mysql> SHOW ENGINE INNODB STATUS\G
// 어댑티브 해시 인덱스의 메모리 사용량 확인(performance_schema)
mysql> SELECT EVENT_NAME, CURRENT_NUMBER_OF_BYTES_USED FROM performace_schema.memory_summary_global_by_event_name WHERE EVENT_NAME='memory/innodb/adaptive hash index';
```


### 4.2.13 InnoDB와 MyISAM, MEMORY 스토리지 엔진 비교
- MySQL 8.0부터 MySQL 서버의 모든 시스템이 InnoDB 스토리지 엔진으로 교체됨 (이전에는 서버의 시스템 테이블, 전문 검색 등은 MyISAM을 사용했음)
- MEMORY 스토리지 엔진 또한 동시 처리 성능에 있어서 InnoDB 엔진을 따라갈 수 없음
- MySQL 서버는 일반적으로 온라인 트랜잭션 처리를 위한 목적으로 사용되며 동시 처리 성능이 중요함
- MEMORY 스토리지가 가변 길이 타입의 컬럼을 지원하지 않아 8.0버전부터는 TempTable 스토리지 엔진이 대체하고 있음 (internal_tmp_mem_storage_engine)
