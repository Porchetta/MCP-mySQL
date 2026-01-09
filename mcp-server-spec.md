# MySQL DBA Assistant MCP Server 명세서

이 문서는 MySQL 데이터베이스 운영 및 관리 자동화를 위한 MCP 서버의 핵심 기능과 DB 연결 관리 전략을 정의합니다.

## 1. DB 연결 및 리스트 관리 전략
다수의 DB를 안전하고 체계적으로 관리하기 위해 환경 변수와 설정 파일을 혼합하여 사용합니다.

* **비민감 정보 (YAML/JSON):** `databases.yaml` 파일에 DB 별칭(Alias), 호스트, 포트, 용도(운영/개발) 등을 리스트 형태로 저장합니다.
* **민감 정보 (Environment Variables):** 접속 계정 및 패스워드는 `.env` 파일에 저장하거나 시스템 환경 변수를 참조하여 코드에 노출되지 않도록 합니다.
* **동적 확장성:** MCP 서버 시작 시 해당 리스트를 로드하여 LLM이 "현재 어떤 DB들에 접속 가능한지" 즉시 파악할 수 있도록 컨텍스트를 제공합니다.

## 2. 핵심 기능 리스트 (Tools)

### ① 모니터링 및 상태 조회 (Read-Only)
* **get_server_status**: 커넥션 수, 업타임, 버퍼 풀 히트율 등 서버 핵심 지표 확인.
* **list_databases**: 전체 DB 목록 및 각 데이터베이스별 사용 용량 조회.
* **show_processlist**: 현재 실행 중인 모든 세션 목록(ID, User, State, Time, Query) 조회.
* **check_replication**: 복제 상태(Master-Slave) 및 Slave Lag 발생 여부 체크.

### ② 성능 분석 및 튜닝 (Diagnostic)
* **analyze_slow_queries**: Slow Query Log를 분석하여 시스템 부하의 주원인인 쿼리 식별.
* **get_table_statistics**: 테이블 인덱스 구성 및 데이터 단편화(Fragmentation) 상태 확인.
* **explain_query**: 특정 SQL의 실행 계획을 분석하여 인덱스 사용 적절성 진단.
* **check_innodb_locks**: 현재 발생한 트랜잭션 락 및 데드락(Deadlock) 상황 상세 분석.

### ③ 세션 및 리소스 관리 (Write/Action)
* **kill_session**: 특정 `Process ID`를 강제 종료하여 롱러닝 쿼리나 서비스 장애 유발 세션 차단.
* **terminate_idle_connections**: 설정된 시간 이상 활동이 없는(Sleep) 세션들을 일괄 정리하여 커넥션 풀 확보.

### ④ 안전 및 감사 (Safety & Audit)
* **dry_run_mode**: DML/DDL 실행 전 영향받는 레코드 수를 사전 계산하여 리스크 방지.
* **get_audit_log**: MCP 서버를 통해 수행된 모든 DB 작업 이력 로깅 및 조회.

---
**보안 주의사항:** - `kill_session` 구현 시, 복제 스레드나 시스템 필수 프로세스는 종료 대상에서 제외하는 내부 필터링 로직이 반드시 포함되어야 합니다.
- DB 계정은 작업에 필요한 최소 권한(Least Privilege)만 부여된 전용 계정을 사용하십시오.
