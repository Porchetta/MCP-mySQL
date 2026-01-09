# MySQL DBA Assistant MCP Server 명세서

이 문서는 MySQL 데이터베이스 운영 및 관리 자동화를 위한 MCP 서버의 핵심 기능과 DB 연결 관리 전략을 정의합니다.

## 1. DB 연결 및 리스트 관리 전략
다수의 DB를 안전하고 체계적으로 관리하기 위해 환경 변수와 설정 파일을 혼합하여 사용합니다.

* **비민감 정보 (YAML/JSON):** `databases.yaml` 파일에 DB 별칭(Alias), 호스트, 포트, 용도(운영/개발) 등을 리스트 형태로 저장합니다.
* **민감 정보 (Environment Variables):** 접속 계정 및 패스워드는 `.env` 파일에 저장하거나 시스템 환경 변수를 참조하여 코드에 노출되지 않도록 합니다.
* **동적 확장성:** MCP 서버 시작 시 해당 리스트를 로드하여 LLM이 "현재 어떤 DB들에 접속 가능한지" 즉시 파악할 수 있도록 컨텍스트를 제공합니다.

## 2. DB 선택/컨텍스트 규칙
* 모든 Tool 호출은 대상 DB를 명시해야 하며, 공통 입력으로 `db_alias`를 사용합니다.
* `db_alias`는 `databases.yaml`에 정의된 별칭과 1:1로 매핑됩니다.
* 서버는 각 요청마다 `db_alias` 검증을 수행하고, 미정의 별칭은 `INVALID_DB_ALIAS` 오류로 거절합니다.

## 3. 인증/권한 모델 (RBAC)
* 인증 방식은 API Key 또는 mTLS 중 하나를 선택하여 구성합니다.
* 역할(Role) 기반 권한을 적용합니다.
  * `READ_ONLY`: 모니터링/진단 도구만 허용
  * `OPS`: 세션/리소스 관리 도구 허용
  * `ADMIN`: 모든 도구 허용
* 위험 작업(`kill_session`, `terminate_idle_connections`)은 최소 `OPS` 권한이 필요합니다.

## 4. MCP Tool 계약 (입력/출력 스키마)
모든 Tool은 다음 공통 규칙을 따릅니다.

* 요청 공통 필드: `db_alias` (필수)
* 응답 공통 필드: `status` (success|error)
* 오류 응답 스키마: `{ "status": "error", "error": { "code": "...", "message": "...", "details": { ... } } }`

### 예시 스키마

```json
{
  "tool": "get_server_status",
  "input": {
    "db_alias": "prod-main"
  },
  "output": {
    "status": "success",
    "data": {
      "uptime_seconds": 123456,
      "connections": 82,
      "buffer_pool_hit_rate": 0.98
    }
  }
}
```

## 5. 핵심 기능 리스트 (Tools)

### ① 모니터링 및 상태 조회 (Read-Only)
* **get_server_status**: 커넥션 수, 업타임, 버퍼 풀 히트율 등 서버 핵심 지표 확인.
  * 입력: `{ db_alias }`
  * 출력: `{ uptime_seconds, connections, buffer_pool_hit_rate }`
* **list_databases**: 전체 DB 목록 및 각 데이터베이스별 사용 용량 조회.
  * 입력: `{ db_alias }`
  * 출력: `{ databases: [ { name, size_bytes } ] }`
* **show_processlist**: 현재 실행 중인 모든 세션 목록(ID, User, State, Time, Query) 조회.
  * 입력: `{ db_alias }`
  * 출력: `{ sessions: [ { id, user, state, time_seconds, query } ] }`
* **check_replication**: 복제 상태(Master-Slave) 및 Slave Lag 발생 여부 체크.
  * 입력: `{ db_alias }`
  * 출력: `{ role, seconds_behind_master, io_thread_running, sql_thread_running }`

### ② 성능 분석 및 튜닝 (Diagnostic)
* **analyze_slow_queries**: Slow Query Log를 분석하여 시스템 부하의 주원인인 쿼리 식별.
  * 입력: `{ db_alias, since_minutes }`
  * 출력: `{ top_queries: [ { sql_fingerprint, avg_time_ms, count } ] }`
* **get_table_statistics**: 테이블 인덱스 구성 및 데이터 단편화(Fragmentation) 상태 확인.
  * 입력: `{ db_alias, schema, table }`
  * 출력: `{ indexes: [ { name, columns } ], fragmentation_percent }`
* **explain_query**: 특정 SQL의 실행 계획을 분석하여 인덱스 사용 적절성 진단.
  * 입력: `{ db_alias, sql }`
  * 출력: `{ plan: [ { id, select_type, table, type, key, rows } ] }`
* **check_innodb_locks**: 현재 발생한 트랜잭션 락 및 데드락(Deadlock) 상황 상세 분석.
  * 입력: `{ db_alias }`
  * 출력: `{ locks: [ { trx_id, waiting, blocked_by, query } ] }`

### ③ 세션 및 리소스 관리 (Write/Action)
* **kill_session**: 특정 `Process ID`를 강제 종료하여 롱러닝 쿼리나 서비스 장애 유발 세션 차단.
  * 입력: `{ db_alias, process_id, confirm_token }`
  * 출력: `{ killed: true }`
* **terminate_idle_connections**: 설정된 시간 이상 활동이 없는(Sleep) 세션들을 일괄 정리하여 커넥션 풀 확보.
  * 입력: `{ db_alias, idle_seconds, confirm_token }`
  * 출력: `{ terminated: number }`

### ④ 안전 및 감사 (Safety & Audit)
* **dry_run_mode**: DML/DDL 실행 전 영향받는 레코드 수를 사전 계산하여 리스크 방지.
  * 입력: `{ db_alias, sql }`
  * 출력: `{ estimated_affected_rows }`
* **get_audit_log**: MCP 서버를 통해 수행된 모든 DB 작업 이력 로깅 및 조회.
  * 입력: `{ db_alias, since, until, tool, actor }`
  * 출력: `{ entries: [ { timestamp, actor, tool, db_alias, summary, result, error } ] }`

## 6. 안전 장치 및 실행 정책
* 위험 작업은 `confirm_token`을 요구하며, 토큰은 서버가 1회성으로 발급합니다.
* `dry_run_mode`는 DML/DDL에 대한 영향도 추정 도구이며, 실행 전에 반드시 호출하여 결과를 검토해야 합니다.
* 복제 환경에서 `role`이 `replica`인 경우, 기본 정책은 읽기 전용 도구만 허용합니다.

## 7. 감사 로그 스키마 및 보존 정책
* 로그 필드: `timestamp`, `actor`, `tool`, `db_alias`, `summary`, `result`, `error`
* 민감 정보(SQL 파라미터 등)는 마스킹 후 저장합니다.
* 보존 기간은 기본 90일이며, 운영 정책에 따라 조정 가능합니다.

## 8. 표준 오류 코드
* `DB_CONN_FAILED`: DB 연결 실패
* `PERMISSION_DENIED`: 권한 부족
* `INVALID_DB_ALIAS`: 유효하지 않은 DB 별칭
* `QUERY_TIMEOUT`: 쿼리 타임아웃
* `REPLICA_READ_ONLY`: 복제 노드에서 쓰기 시도

---
**보안 주의사항:** - `kill_session` 구현 시, 복제 스레드나 시스템 필수 프로세스는 종료 대상에서 제외하는 내부 필터링 로직이 반드시 포함되어야 합니다.
- DB 계정은 작업에 필요한 최소 권한(Least Privilege)만 부여된 전용 계정을 사용하십시오.
