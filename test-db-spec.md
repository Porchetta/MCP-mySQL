# 테스트용 MySQL DB 컨테이너 사양서

이 문서는 MCP 서버 테스트를 위해 Docker로 구동할 두 개의 MySQL 컨테이너 사양을 정의합니다. 각 컨테이너는 서로 다른 역할(운영/개발)로 구성되며, 더미 데이터 로딩 절차와 초기화 스크립트를 포함합니다.

## 1. 컨테이너 구성 개요

| 항목 | test-mysql-primary | test-mysql-replica |
| --- | --- | --- |
| 역할 | 운영(Primary) | 개발(Replica) |
| 이미지 | mysql:8.0 | mysql:8.0 |
| 컨테이너명 | test-mysql-primary | test-mysql-replica |
| 포트 | 3306:3306 | 3307:3306 |
| 볼륨 | ./data/primary:/var/lib/mysql | ./data/replica:/var/lib/mysql |
| 초기화 SQL | ./init/primary | ./init/replica |

## 2. 공통 환경 변수

모든 컨테이너에 아래 환경 변수를 적용합니다.

* `MYSQL_ROOT_PASSWORD`: 루트 비밀번호 (테스트 전용 값)
* `MYSQL_USER`: 애플리케이션 전용 계정
* `MYSQL_PASSWORD`: 애플리케이션 계정 비밀번호
* `MYSQL_DATABASE`: 기본 데이터베이스 이름

## 3. 데이터베이스 및 계정 초기화

초기화 SQL은 `docker-entrypoint-initdb.d` 경로에 마운트합니다.

* `./init/primary/*.sql` → `/docker-entrypoint-initdb.d`
* `./init/replica/*.sql` → `/docker-entrypoint-initdb.d`

초기 SQL에는 아래 항목이 포함되어야 합니다.

1. 테스트용 스키마 생성 (예: `mcp_test`)
2. 더미 데이터 삽입용 테이블 생성
3. 최소 권한 테스트 계정 생성 (예: `mcp_reader`, `mcp_ops`)
4. 읽기 전용/쓰기 가능 권한 분리

## 4. 더미 데이터 요구사항

테스트 시나리오를 검증하기 위해 다음과 같은 더미 데이터를 포함합니다.

* 사용자 테이블: 최소 10만 건
* 주문 테이블: 최소 50만 건
* 로그/이력 테이블: 최소 100만 건

샘플 컬럼 예시:

* `users(id, email, name, created_at)`
* `orders(id, user_id, total_amount, status, created_at)`
* `audit_logs(id, actor, tool, db_alias, summary, created_at)`

## 5. 복제 테스트 설정(옵션)

복제 테스트가 필요한 경우 아래 설정을 추가합니다.

* primary: `server-id=1`, `log_bin=ON`, `binlog_format=ROW`
* replica: `server-id=2`, `read_only=ON`, `relay_log=ON`

복제 계정은 최소 권한으로 생성합니다.

## 6. MCP 연동을 위한 databases.yaml 예시

```yaml
databases:
  - alias: test-primary
    host: 127.0.0.1
    port: 3306
    role: primary
  - alias: test-replica
    host: 127.0.0.1
    port: 3307
    role: replica
```

## 7. 추가 고려 사항

* 컨테이너 네트워크는 `mcp-test-net` 등 전용 브리지 네트워크 사용을 권장합니다.
* 더미 데이터는 테스트 성능에 영향을 주므로, 필요 시 규모를 조정할 수 있도록 스크립트 파라미터화를 권장합니다.
* 데이터 초기화 및 복제 설정은 단일 `docker-compose.yml`로 자동화하는 것을 권장합니다.
