# MySQL 8.4 Master-Replica (GTID) — Docker Compose

MySQL 8.4 LTS 기반 **Master-Replica(구 Master-Slave) 복제** 구성입니다.
`master`와 `replica`를 각각 별도의 Docker Compose로 띄우고, 하나의 external 네트워크(`mysql-repl-net`)로 묶어 GTID 기반 복제를 연결.

---

## 디렉터리 구조

```text
mysql/
└── 8.4/
    ├── master/
    │   ├── docker-compose.yml   # mysql-master (host 3306)
    │   ├── .env                 # MYSQL_ROOT_PASSWORD
    │   ├── conf/master.cnf      # server-id=1, log-bin, gtid-mode=ON ...
    └── replica/
        ├── docker-compose.yml   # mysql-replica (host 3307)
        ├── .env                 # MYSQL_ROOT_PASSWORD
        └── conf/replica.cnf     # server-id=2, relay-log, read-only ...
```

---

## 사전 요구사항

- Docker Engine + Docker Compose plugin
- 두 compose가 공유하는 external 네트워크 (`mysql-repl-net`)

---

## 빠른 시작

아래는 **한 호스트에서 master + replica를 모두 Docker로** 띄우는 로컬 기준.
(별도 VM/네이티브라면 `SOURCE_HOST`를 실제 IP로 바꾸면 된다.. 맨 아래 참고.)

### 1. 공유 네트워크 생성 (최초 1회)

두 `docker-compose.yml`이 `mysql-repl-net`을 `external: true`로 참조하므로 **먼저 만들어야** 한다.

```bash
docker network create mysql-repl-net
```

### 2. `.env` 설정

`master/.env`, `replica/.env` 각각에 root 비밀번호를 지정.

```dotenv
MYSQL_ROOT_PASSWORD=비밀번호_입력
```

> 비밀번호에 `#`이 있으면 Compose 버전에 따라 주석으로 잘릴 수 있으니 `"..."`로 감싸야함.

### 3. Master 기동

```bash
cd 8.4/master
docker compose up -d
docker logs -f mysql-master     # "ready for connections" 확인
```

### 4. Replica 기동

```bash
cd 8.4/replica
docker compose up -d
docker logs -f mysql-replica    # "ready for connections" 확인
```

### 5. Master에 복제 전용 계정 생성

```bash
docker exec -it mysql-master mysql -uroot -p
```

```sql
CREATE USER 'da_repl'@'%' IDENTIFIED WITH caching_sha2_password BY '복제전용_비밀번호';
GRANT REPLICATION SLAVE ON *.* TO 'da_repl'@'%';
FLUSH PRIVILEGES;

-- 확인
SHOW BINARY LOG STATUS\G   -- (구버전은 SHOW MASTER STATUS)
```

### 6. Replica에서 복제 연결

```bash
docker exec -it mysql-replica mysql -uroot -p
```

```sql
STOP REPLICA;

CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='mysql-master',      -- 같은 네트워크라 컨테이너 이름으로 접속
  SOURCE_PORT=3306,                -- 컨테이너 내부 포트
  SOURCE_USER='da_repl',
  SOURCE_PASSWORD='복제전용_비밀번호',
  SOURCE_AUTO_POSITION=1,          -- GTID 기반: 로그 파일/포지션 대신 사용
  GET_SOURCE_PUBLIC_KEY=1;         -- caching_sha2_password + 비SSL 연결 대응

START REPLICA;
SHOW REPLICA STATUS\G
```

### 7. 성공 판정

```text
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
Last_IO_Error:                 (비어 있음)
Seconds_Behind_Source: 0
```

두 스레드가 모두 `Yes`면 복제 파이프라인 완성입니다.

### 8. 동작 확인

```bash
docker exec -it mysql-master  mysql -uroot -p -e "CREATE DATABASE repl_test;"
docker exec -it mysql-replica mysql -uroot -p -e "SHOW DATABASES;"   # repl_test 보이면 성공
```

---

## 접속 정보

| 대상 | 호스트에서 | 컨테이너끼리 | server-id |
|---|---|---|---|
| master | `127.0.0.1:3306` | `mysql-master:3306` | `1` |
| replica | `127.0.0.1:3307` | `mysql-replica:3306` | `2` |

호스트 포트(3306/3307)는 DBeaver 등 외부 툴 접속용이고, 복제는 컨테이너끼리 내부 3306으로 통신

---

## 설정 파일 요약

### `master/conf/master.cnf`

| 항목 | 값 | 설명 |
|---|---|---|
| `server-id` | `1` | Replica와 반드시 다른 값 |
| `log-bin` | `mysql-bin` | 바이너리 로그 (복제 원천) |
| `binlog_format` | `ROW` | 바뀐 값 자체를 기록 (실무 표준) |
| `gtid-mode` | `ON` | GTID 기반 복제 |
| `enforce-gtid-consistency` | `ON` | GTID 안전성 강제 |
| `bind-address` | `0.0.0.0` | 원격 접속 허용 |

### `replica/conf/replica.cnf`

| 항목 | 값 | 설명 |
|---|---|---|
| `server-id` | `2` | Master와 다른 값 |
| `relay-log` | `relay-bin` | 릴레이 로그(받기/적용 분리) |
| `gtid-mode` / `enforce-gtid-consistency` | `ON` | Master와 동일하게 |
| `read-only` / `super-read-only` | `ON` | Replica 쓰기 차단 |
| `relay-log-recovery` | `ON` | 재시작 시 릴레이 로그 자동 복구 |

> ⚠️ `read-only`/`super-read-only`는 **데이터 디렉터리 최초 초기화가 끝난 뒤** 적용.
> 빈 `data/`로 첫 기동할 때 켜져 있으면, 컨테이너 초기화 단계의 임시 서버가 root 비밀번호 설정 SQL을 거부해 초기화가 깨짐.
> 최초 구성 시에는 두 줄을 주석 처리했다가, 복제 연결 후 아래로 켜는 방법도 있다..

```sql
SET PERSIST super_read_only = ON;
SET PERSIST read_only = ON;
```

---

## 운영 / 관리

### 상태 점검

```sql
SHOW REPLICA STATUS\G
SELECT @@server_id, @@gtid_mode, @@super_read_only;
```

### 중지 / 재시작

```bash
docker compose stop        # 컨테이너 중지 (데이터 유지)
docker compose up -d       # 재기동
docker compose down        # 컨테이너 제거 (data/ 볼륨은 유지)
```

### 데이터 완전 초기화 (주의)

초기화가 깨졌거나 처음부터 다시 구성할 때만.

```bash
docker compose down
sudo rm -rf ./data         # 컨테이너가 만든 파일은 uid 999 소유라 sudo 필요
docker compose up -d
```

---

## 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| 컨테이너가 아예 안 뜸 | `.env`의 `MYSQL_ROOT_PASSWORD` 변수명 오타 | 변수명 정확히 확인 |
| `network mysql-repl-net not found` | external 네트워크 미생성 | `docker network create mysql-repl-net` |
| `unknown variable` 후 crash loop | 8.4에서 제거된 옵션(`*-info-repository`, `default-authentication-plugin`) | 해당 라인 삭제 |
| `Table 'mysql.user' doesn't exist` | 초기화 도중 죽어 반쯤 초기화된 `data/` | `data/` 비우고 재초기화 |
| 초기화 단계 `--super-read-only ... cannot execute` | cnf의 read-only가 첫 초기화 방해 | 초기화 후 `SET PERSIST`로 적용 |
| `Access denied (using password: YES)` | `SOURCE_PASSWORD` 문자열 불일치 (특수문자 이스케이프) | 직접 로그인 테스트로 격리, 비밀번호에 `\` `'` 회피 |

복제 계정 자체를 격리 검증하려면 Replica에서 Master로 직접 로그인 시도.

```bash
docker exec -it mysql-replica mysql -uda_repl -p'비밀번호' -h mysql-master -e "SELECT 1;"
```

---

## VM/네이티브로 옮길 때

이 저장소는 한 호스트에서 둘 다 Docker로 띄우는 구성입니다.
Master를 별도 서버(예: `192.168.2.10`)에 네이티브로 두는 경우:

- `mysql-repl-net` 공유 네트워크는 필요 없음 (Replica만 Docker)
- Replica의 복제 연결에서 `SOURCE_HOST`를 컨테이너 이름 대신 **실제 IP**로 지정

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='192.168.2.10',
  SOURCE_PORT=3306,
  ...
```