# GitLab 그룹 삭제 실패 이슈 분석 및 조치 내역

## 1. 개요 (Summary)

- **대상 시스템**: GitLab Self-Managed
- **버전**: 17.10.7-ce.0
- **증상**:
  - GitLab UI에서 그룹 삭제 시 "삭제 성공" 메시지는 표시됨
  - 실제로 그룹은 삭제되지 않음
  - Pending deletion / Deletion scheduled 표시 없음
- **영향**:
  - 그룹 삭제 불가
  - Sidekiq job 실패

---

## 2. 초기 증상 및 관찰 사항

### UI 동작

- 그룹 삭제 버튼 클릭 시:
  - POST `/groups/<group>/-/edit` 요청 발생
  - HTTP 302 → GET `/groups/<group>/-/edit` (200)
  - UI 상단에 "삭제 성공" 메시지 출력
- 실제로는 그룹이 유지됨

### 로그 관찰

| 로그 파일 | 관찰 내용 |
|-----------|-----------|
| `gitlab_access.log` | POST `/groups/manual/-/edit` → 302, GET → 200 |
| `gitlab_error.log` | 특이사항 없음 |
| `gitlab-workhorse` | 동일한 POST 요청 확인 |
| `gitlab-rails production.log` | 삭제 관련 로그 없음 |
| `production_json.log` | DELETE/POST 관련 에러 로그 없음 |

---

## 3. 1차 가설 및 배제

| 가설 | 결과 |
|------|------|
| Owner 권한 부족 | ❌ Owner 맞음 |
| Pending deletion 상태 | ❌ 없음 |
| Deletion delay 설정 | ❌ CE 버전, 해당 기능 없음 |
| Sidekiq 비정상 | ❌ 정상 동작 |
| UI 캐시/브라우저 문제 | ❌ 동일 |

---

## 4. 강제 삭제 시도 (Rails Console)

### 시도 1: 직접 실행

```ruby
GroupDestroyWorker.new.perform(group.id, user.id)
```

### 결과

```
Gitlab::Auth::CurrentUserMode::NonSidekiqEnvironmentError
```

→ **Sidekiq 환경에서만 실행 가능**

---

## 5. Sidekiq enqueue 시도

```ruby
GroupDestroyWorker.perform_async(group.id, user.id)
```

### 결과

- Job enqueue 됨
- Sidekiq 로그에서 **start → fail** 전환 확인
- 에러:

```
OpenSSL::Cipher::CipherError
```

---

## 6. 핵심 원인 분석

### 6.1 Sidekiq 로그 분석

- GroupDestroyWorker 실행 중 암호화 필드 복호화 실패
- 에러 원인: **암호화된 DB 값과 현재 GitLab 암호화 키 불일치**

### 6.2 Secrets 진단

```bash
sudo gitlab-rake gitlab:doctor:secrets VERBOSE=1
```

#### 결과

```
ApplicationSetting failures : 1
ApplicationSetting[1]:
- ci_jwt_signing_key
- ci_job_token_signing_key
- runners_registration_token
- error_tracking_access_token
```

---

## 7. 근본 원인 (Root Cause)

`/etc/gitlab/gitlab-secrets.json`이 **기존 DB를 암호화할 때 사용된 파일과 불일치**

### 원인 가능성

- GitLab 재설치
- 서버 이전
- DB 복구 시 secrets 파일 미복원

### 결과

- ApplicationSetting에 저장된 암호화 필드 복호화 불가
- GroupDestroyWorker 포함 여러 Sidekiq 작업 실패

---

## 8. 실패한 접근 방식 정리

### 8.1 reset_* 메서드 활용 시도

확인된 reset 메서드:

```ruby
reset_runners_registration_token!
reset_health_check_access_token!
reset_static_objects_external_storage_auth_token!
```

### 실패 이유

- 문제의 핵심 필드:
  - `ci_jwt_signing_key`
  - `ci_job_token_signing_key`
- 해당 필드에 대한 `reset_*` 메서드 **의도적으로 제공되지 않음**
- GitLab CE 정책상 자동 복구 불가

---

## 9. 최종 해결 전략

### 전제 조건

- GitLab Runner **미사용**
- CI/CD는 **GitLab Webhook → Jenkins** 구조
- `.gitlab-ci.yml`, GitLab CI 파이프라인 사용 안 함

→ CI JWT / Job Token 재생성 영향 **없음**

---

## 10. 최종 조치 내용

### 10.1 reset 가능한 항목 초기화

```ruby
s = ApplicationSetting.current
s.reset_runners_registration_token!
s.reset_health_check_access_token!
s.reset_static_objects_external_storage_auth_token!
s.save!
```

### 10.2 CI 서명 키 수동 재생성 시도 (실패)

```bash
sudo gitlab-rails runner -e production '
require "securerandom"
s = ApplicationSetting.current
s.update!(
  ci_jwt_signing_key: SecureRandom.hex(64),
  ci_job_token_signing_key: SecureRandom.hex(64)
)
'
```

#### 실패 원인

```
OpenSSL::Cipher::CipherError
```

- `ApplicationSetting.current` 호출 시 암호화된 필드를 자동으로 복호화 시도
- secrets 키 불일치로 복호화 자체가 실패
- 새 값을 쓰기 전에 읽기 단계에서 에러 발생

---

### 10.3 암호화 컬럼 직접 초기화 시도 (실패)

```bash
sudo gitlab-rails runner -e production '
require "securerandom"

ApplicationSetting.where(id: 1).update_all(
  encrypted_ci_jwt_signing_key: nil,
  encrypted_ci_jwt_signing_key_iv: nil,
  encrypted_ci_job_token_signing_key: nil,
  encrypted_ci_job_token_signing_key_iv: nil
)

s = ApplicationSetting.find(1)
s.ci_jwt_signing_key = SecureRandom.hex(64)
s.ci_job_token_signing_key = SecureRandom.hex(64)
s.save!
'
```

#### 실패 원인

```
Validation failed: Ci jwt signing key is not a valid RSA key
```

- `SecureRandom.hex(64)`는 단순 hex 문자열
- GitLab은 RSA PEM 형식의 키를 요구

---

### 10.4 RSA 키로 변경 후 save! 시도 (실패)

```ruby
s.ci_jwt_signing_key = OpenSSL::PKey::RSA.new(2048).to_pem
s.ci_job_token_signing_key = OpenSSL::PKey::RSA.new(2048).to_pem
s.save!
```

#### 실패 원인

```
OpenSSL::Cipher::CipherError
```

- `save!`는 모든 콜백/validation 실행
- 다른 암호화 필드 접근 시 복호화 실패

---

### 10.5 update_columns로 콜백 우회 (성공)

`/tmp/fix_secrets.rb`:

```ruby
require "openssl"

# Step 1: 모든 encrypted 컬럼 NULL 처리
cols = ApplicationSetting.column_names.select { |c| c.start_with?("encrypted_") }
set_clause = cols.map { |c| "#{c} = NULL" }.join(", ")
ApplicationSetting.connection.execute("UPDATE application_settings SET #{set_clause} WHERE id = 1")

# Step 2: RSA 키 생성 후 update_columns로 저장 (콜백 우회)
s = ApplicationSetting.find(1)
s.ci_jwt_signing_key = OpenSSL::PKey::RSA.new(2048).to_pem
s.ci_job_token_signing_key = OpenSSL::PKey::RSA.new(2048).to_pem

s.update_columns(
  encrypted_ci_jwt_signing_key: s.encrypted_ci_jwt_signing_key,
  encrypted_ci_jwt_signing_key_iv: s.encrypted_ci_jwt_signing_key_iv,
  encrypted_ci_job_token_signing_key: s.encrypted_ci_job_token_signing_key,
  encrypted_ci_job_token_signing_key_iv: s.encrypted_ci_job_token_signing_key_iv
)

puts "CI signing keys regenerated successfully"
```

```bash
sudo gitlab-rails runner -e production /tmp/fix_secrets.rb
```

#### 핵심 포인트

| 메서드 | 동작 | 콜백/복호화 |
|--------|------|-------------|
| `save!` | 모든 콜백, validation 실행 | ⚠️ 발생 |
| `update_columns` | 지정한 컬럼만 직접 UPDATE | ✅ 우회 |

---

### 10.6 나머지 토큰 정리

CI 키 해결 후 재진단:

```bash
sudo gitlab-rake gitlab:doctor:secrets VERBOSE=1
```

```
ApplicationSetting failures : 1
- runners_registration_token
- error_tracking_access_token
```

#### 컬럼명 확인

```bash
sudo gitlab-rails runner -e production '
cols = ApplicationSetting.column_names.select { |c| c.include?("runner") || c.include?("error") }
puts cols.sort
'
```

실제 컬럼명: `runners_registration_token_encrypted`, `error_tracking_access_token_encrypted`

#### NULL 처리

```ruby
ApplicationSetting.where(id: 1).update_all(
  runners_registration_token_encrypted: nil,
  error_tracking_access_token_encrypted: nil
)
```

---

### 10.7 Project runners_token 정리

재진단 결과:

```
Project failures : 19
Project[n]: runners_token
```

그룹 삭제 시 하위 프로젝트도 삭제되며 `runners_token` 복호화 시도 → 실패

#### 전체 프로젝트 토큰 NULL 처리

```bash
sudo gitlab-rails runner -e production '
Project.update_all(runners_token_encrypted: nil)
puts "Done"
'
```

---

### 10.8 서비스 재시작 및 검증

```bash
sudo gitlab-ctl restart
sudo gitlab-rake gitlab:doctor:secrets VERBOSE=1
```

#### 최종 결과

```
ApplicationSetting failures : 0
Project failures : 0
```

---

## 11. 결과

- ✅ ApplicationSetting 암호화 상태 정상화
- ✅ Sidekiq OpenSSL::Cipher::CipherError 해결
- ✅ GroupDestroyWorker 정상 동작
- ✅ 그룹 삭제 정상 수행

---

## 12. 교훈 및 재발 방지

### 운영 교훈

- **gitlab-secrets.json은 DB 백업과 별도로 반드시 보관**
- GitLab 백업 tar에 secrets가 자동 포함된다고 가정하면 안 됨

### 권장 사항

- `/etc/gitlab/gitlab-secrets.json` 정기 백업
- 서버 이전/복구 시 반드시 함께 복원할 항목:
  - DB
  - repositories
  - **secrets 파일**

---

## 13. 요약

> GitLab 그룹 삭제 실패의 원인은 **ApplicationSetting 및 Project의 암호화 키 불일치로 인한 Sidekiq 복호화 실패**였으며, Runner 미사용 환경에서는 `update_columns`를 활용해 콜백을 우회하고 CI 서명 키(RSA)를 재생성, 나머지 토큰들은 NULL 처리하는 것이 해결책이었다.

---

## 부록 A. GitLab 아키텍처 개념

### A.1 GitLab-Workhorse란?

GitLab의 **리버스 프록시** 역할을 하는 Go 언어로 작성된 컴포넌트.

```
[사용자 브라우저] → [Nginx] → [GitLab-Workhorse] → [Puma/Rails]
```

| 역할 | 설명 |
|------|------|
| 정적 파일 처리 | CSS, JS, 이미지 등 Rails 거치지 않고 직접 응답 |
| 대용량 파일 업로드 | Git LFS, 아티팩트 업로드 시 스트리밍 처리 |
| Git HTTP 요청 | `git clone`, `git push` 등의 HTTP 요청 처리 |
| WebSocket 프록시 | 실시간 기능 (터미널, 로그 스트리밍) |

Workhorse가 처리할 수 없는 동적 요청만 Rails(Puma)로 전달.

---

### A.2 Sidekiq의 역할

**백그라운드 작업 처리기** (Ruby 기반 job processor).

GitLab에서 시간이 오래 걸리거나 즉시 응답이 필요 없는 작업을 비동기로 처리:

| 작업 유형 | 예시 |
|-----------|------|
| 이메일 발송 | 알림, 초대 메일 |
| Repository 작업 | fork, mirror, cleanup |
| CI/CD | 파이프라인 실행, 아티팩트 정리 |
| **그룹/프로젝트 삭제** | `GroupDestroyWorker`, `ProjectDestroyWorker` |
| 검색 인덱싱 | Elasticsearch 동기화 |

```
[Rails 애플리케이션] → [Redis Queue] → [Sidekiq Worker] → [작업 실행]
```

UI에서 "삭제 성공" 메시지가 바로 뜨는 이유: Rails는 Sidekiq에 job을 **enqueue만** 하고 즉시 응답. 실제 삭제는 Sidekiq가 비동기로 처리.

---

### A.3 그룹 삭제 프로세스

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. 사용자가 UI에서 "그룹 삭제" 클릭                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. POST /groups/<group>/-/edit 요청                             │
│    → Nginx → Workhorse → Rails(Puma)                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Rails Controller가 GroupDestroyWorker.perform_async 호출     │
│    → Redis에 job enqueue                                        │
│    → 사용자에게 "삭제 예약됨" 응답 (HTTP 302)                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. Sidekiq이 Redis에서 job 가져옴                               │
│    → GroupDestroyWorker.perform 실행                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. 삭제 작업 수행                                                │
│    - 하위 프로젝트 삭제 (ProjectDestroyWorker)                   │
│    - Repository 파일 삭제                                        │
│    - PostgreSQL에서 레코드 DELETE                                │
│    - 연관 데이터 정리 (이슈, MR, 멤버십 등)                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. 완료 또는 실패                                                │
│    - 성공: 그룹이 DB와 파일시스템에서 완전히 제거                 │
│    - 실패: Sidekiq 로그에 에러 기록, 그룹은 그대로 유지          │
└─────────────────────────────────────────────────────────────────┘
```

**이번 장애의 문제점**: 5단계에서 암호화된 필드 복호화 실패 → Sidekiq job fail → 그룹 유지

---

### A.4 gitlab-rake gitlab:doctor:secrets 명령어

```bash
sudo gitlab-rake gitlab:doctor:secrets VERBOSE=1
```

| 구성 요소 | 설명 |
|-----------|------|
| `gitlab-rake` | GitLab 전용 Rake task 실행기 |
| `gitlab:doctor:secrets` | 암호화된 DB 필드 상태 점검 task |
| `VERBOSE=1` | 상세 출력 (어떤 필드가 실패했는지 표시) |

**동작 원리**:

1. DB에서 암호화된 필드가 있는 모든 레코드 조회
2. 각 필드를 현재 `gitlab-secrets.json` 키로 복호화 시도
3. 실패한 필드 목록 출력

```
# 정상
ApplicationSetting failures : 0
Project failures : 0

# 비정상
ApplicationSetting failures : 1
ApplicationSetting[1]:
- ci_jwt_signing_key        ← 이 필드 복호화 실패
- ci_job_token_signing_key
```

---

### A.5 암호화 토큰 필드 설명

| 필드명 | 용도 | Runner 미사용 시 |
|--------|------|------------------|
| `ci_jwt_signing_key` | GitLab CI job에서 외부 서비스 인증용 JWT 토큰 서명 | 불필요 |
| `ci_job_token_signing_key` | CI job 간 인증 (다른 프로젝트 API 호출 등) | 불필요 |
| `runners_registration_token` | 새 Runner를 GitLab에 등록할 때 사용하는 토큰 | 불필요 |
| `error_tracking_access_token` | Sentry 연동 시 인증 토큰 | 불필요 (미사용 시) |
| `runners_token` (Project) | 프로젝트별 Runner 인증 토큰 | 불필요 |

모두 **GitLab CI/CD 기능**과 관련된 토큰들. Jenkins webhook 방식을 사용하면 전부 불필요.

---

## 부록 B. Ruby 스크립트 해석

### B.1 Ruby와 GitLab의 관계

GitLab은 **Ruby on Rails** 프레임워크로 작성된 웹 애플리케이션.

| 기술 | 역할 |
|------|------|
| **Ruby** | 프로그래밍 언어 |
| **Rails** | Ruby 웹 프레임워크 (MVC 패턴) |
| **ActiveRecord** | Rails의 ORM (DB 테이블 ↔ Ruby 객체) |
| **Sidekiq** | Ruby 백그라운드 job 처리 |

`gitlab-rails runner`는 GitLab Rails 환경에서 Ruby 코드를 직접 실행하는 명령어.

---

### B.2 fix_secrets.rb 코드 해석

```ruby
require "openssl"
```
OpenSSL 라이브러리 로드. RSA 키 생성에 필요.

---

```ruby
cols = ApplicationSetting.column_names.select { |c| c.start_with?("encrypted_") }
```
- `ApplicationSetting.column_names`: 테이블의 모든 컬럼명 배열 반환
- `.select { |c| ... }`: 조건에 맞는 것만 필터링
- `c.start_with?("encrypted_")`: "encrypted_"로 시작하는 컬럼만 선택

결과 예시: `["encrypted_ci_jwt_signing_key", "encrypted_ci_jwt_signing_key_iv", ...]`

---

```ruby
set_clause = cols.map { |c| "#{c} = NULL" }.join(", ")
```
- `.map { |c| "#{c} = NULL" }`: 각 컬럼을 "컬럼명 = NULL" 형태로 변환
- `.join(", ")`: 쉼표로 연결

결과: `"encrypted_ci_jwt_signing_key = NULL, encrypted_ci_jwt_signing_key_iv = NULL, ..."`

---

```ruby
ApplicationSetting.connection.execute("UPDATE application_settings SET #{set_clause} WHERE id = 1")
```
- `ApplicationSetting.connection`: DB 연결 객체
- `.execute(...)`: Raw SQL 직접 실행
- ActiveRecord 콜백/validation **완전 우회** → 복호화 시도 없음

---

```ruby
s = ApplicationSetting.find(1)
```
- `id = 1`인 ApplicationSetting 레코드 조회
- 암호화된 필드가 NULL이므로 복호화 시도 없음 → 에러 없음

---

```ruby
s.ci_jwt_signing_key = OpenSSL::PKey::RSA.new(2048).to_pem
s.ci_job_token_signing_key = OpenSSL::PKey::RSA.new(2048).to_pem
```
- `OpenSSL::PKey::RSA.new(2048)`: 2048비트 RSA 키 쌍 생성
- `.to_pem`: PEM 형식 문자열로 변환
- 속성에 할당하면 Rails가 자동으로 암호화해서 `encrypted_*` 컬럼에 저장 준비

---

```ruby
s.update_columns(
  encrypted_ci_jwt_signing_key: s.encrypted_ci_jwt_signing_key,
  encrypted_ci_jwt_signing_key_iv: s.encrypted_ci_jwt_signing_key_iv,
  encrypted_ci_job_token_signing_key: s.encrypted_ci_job_token_signing_key,
  encrypted_ci_job_token_signing_key_iv: s.encrypted_ci_job_token_signing_key_iv
)
```
- `update_columns`: 지정한 컬럼만 직접 UPDATE (콜백/validation 우회)
- `s.encrypted_*`: 위에서 평문 키를 할당했을 때 Rails가 자동 암호화한 값
- `*_iv`: 암호화에 사용된 Initialization Vector (복호화에 필요)

---

### B.3 save! vs update_columns 차이

```ruby
# save! - 모든 콜백과 validation 실행
s.save!
# 1. before_validation 콜백
# 2. validation 실행
# 3. before_save 콜백
# 4. DB UPDATE (모든 변경된 속성)
# 5. after_save 콜백
# → 이 과정에서 다른 암호화 필드 접근 시 복호화 시도 → 에러

# update_columns - 콜백/validation 완전 우회
s.update_columns(col1: val1, col2: val2)
# 1. DB UPDATE (지정한 컬럼만)
# → 다른 필드 접근 안 함 → 에러 없음
```

| 메서드 | 콜백 | Validation | 속성 접근 | 용도 |
|--------|------|------------|-----------|------|
| `save!` | ✅ 실행 | ✅ 실행 | 전체 | 일반적인 저장 |
| `update_columns` | ❌ 우회 | ❌ 우회 | 지정한 것만 | 특수 상황 (이번 케이스) |

---

## 부록 C. 주니어 개발자를 위한 추가 개념

### C.1 ORM (Object-Relational Mapping)

DB 테이블을 프로그래밍 언어의 객체로 다루는 기술.

```
# SQL 직접 작성
SELECT * FROM users WHERE id = 1;

# ORM (ActiveRecord) 사용
User.find(1)
```

| 장점 | 단점 |
|------|------|
| SQL 몰라도 DB 조작 가능 | 복잡한 쿼리는 비효율적 |
| 코드 가독성 향상 | 내부 동작 이해 필요 |
| DB 종류 변경 용이 | 추상화로 인한 성능 저하 가능 |

GitLab은 **ActiveRecord** (Rails 기본 ORM) 사용.

---

### C.2 콜백 (Callback)과 Validation

#### Validation (유효성 검사)

데이터 저장 전 규칙 검증:

```ruby
class User < ApplicationRecord
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :age, numericality: { greater_than: 0 }
end

user = User.new(email: "invalid", age: -1)
user.save  # false 반환, 저장 안 됨
```

#### Callback (콜백)

특정 시점에 자동 실행되는 메서드:

```ruby
class User < ApplicationRecord
  before_save :encrypt_password      # 저장 전 실행
  after_create :send_welcome_email   # 생성 후 실행
  before_destroy :cleanup_files      # 삭제 전 실행
end
```

**이번 장애와의 관계**: `save!` 호출 시 콜백이 실행되면서 다른 암호화 필드에 접근 → 복호화 실패 → 에러

---

### C.3 암호화 기초 개념

#### 대칭키 vs 비대칭키

| 구분 | 대칭키 (AES) | 비대칭키 (RSA) |
|------|-------------|----------------|
| 키 개수 | 1개 (암호화 = 복호화) | 2개 (공개키 + 개인키) |
| 속도 | 빠름 | 느림 |
| 용도 | 데이터 암호화 | 인증, 서명, 키 교환 |

GitLab은 DB 필드 암호화에 **AES** (대칭키), JWT 서명에 **RSA** (비대칭키) 사용.

#### IV (Initialization Vector)

같은 키로 같은 데이터를 암호화해도 **다른 결과**가 나오게 하는 랜덤 값.

```
평문: "hello"
키: "secret123"

IV 없이: 항상 "abc123xyz" → 패턴 분석 가능 (보안 취약)
IV 사용: 매번 다른 결과 → 패턴 분석 불가 (보안 강화)
```

그래서 `encrypted_ci_jwt_signing_key`와 `encrypted_ci_jwt_signing_key_iv`가 쌍으로 존재.

#### PEM 형식

암호화 키를 텍스트로 저장하는 표준 형식:

```
-----BEGIN RSA PRIVATE KEY-----
MIIEpQIBAAKCAQEA0Z3VS5JJcds3xfn/ygWyF8PbnGy...
(Base64 인코딩된 키 데이터)
...
-----END RSA PRIVATE KEY-----
```

`OpenSSL::PKey::RSA.new(2048).to_pem`이 이 형식으로 변환.

---

### C.4 gitlab-secrets.json 구조

```json
{
  "gitlab_rails": {
    "secret_key_base": "abc123...",        // 세션 암호화
    "db_key_base": "def456...",            // DB 필드 암호화 (핵심!)
    "otp_key_base": "ghi789..."            // 2FA OTP 암호화
  },
  "gitlab_workhorse": {
    "secret_token": "jkl012..."
  }
}
```

**`db_key_base`**: DB의 `encrypted_*` 컬럼을 암호화/복호화하는 마스터 키.

이 파일이 바뀌면 → 기존 암호화된 데이터 복호화 불가 → 이번 장애 원인

---

### C.5 Rails Runner vs Rails Console

| 명령어 | 용도 | 특징 |
|--------|------|------|
| `gitlab-rails console` | 대화형 실행 | 한 줄씩 입력, 결과 즉시 확인 |
| `gitlab-rails runner` | 스크립트 실행 | 파일 또는 문자열로 코드 실행 |

```bash
# Console - 대화형
$ sudo gitlab-rails console
irb> User.count
=> 150
irb> exit

# Runner - 스크립트 실행
$ sudo gitlab-rails runner 'puts User.count'
150

# Runner - 파일 실행
$ sudo gitlab-rails runner /tmp/script.rb
```

---

### C.6 Puma란?

Ruby 웹 애플리케이션 서버. Rails 앱을 실행하고 HTTP 요청을 처리.

```
[Nginx] → [Workhorse] → [Puma] → [Rails 애플리케이션]
                          ↑
                    여기서 Ruby 코드 실행
```

| 웹 서버 | 역할 |
|---------|------|
| Nginx | 정적 파일, SSL, 로드밸런싱 |
| Workhorse | Git 요청, 대용량 업로드 |
| Puma | Rails 앱 실행, 동적 요청 처리 |

---

### C.7 Redis의 역할

인메모리 키-값 저장소. GitLab에서 여러 용도로 사용:

| 용도 | 설명 |
|------|------|
| Sidekiq Queue | 백그라운드 job 대기열 |
| 캐시 | 페이지, 쿼리 결과 캐싱 |
| 세션 | 사용자 로그인 세션 저장 |
| 실시간 기능 | ActionCable (WebSocket) |

```
# Sidekiq job 흐름
Rails: GroupDestroyWorker.perform_async(1, 2)
  ↓
Redis: LPUSH queue:default '{"class":"GroupDestroyWorker","args":[1,2]}'
  ↓
Sidekiq: BRPOP queue:default → job 꺼내서 실행
```

---

### C.8 HTTP 상태 코드

| 코드 | 의미 | 이번 장애에서 |
|------|------|---------------|
| 200 | 성공 | 페이지 정상 로드 |
| 302 | 리다이렉트 | 삭제 요청 후 다른 페이지로 이동 |
| 404 | Not Found | - |
| 500 | 서버 에러 | Sidekiq 실패 시에도 UI는 302 |

**이번 장애의 함정**: UI는 302 (성공처럼 보임) 반환했지만, 실제 삭제는 Sidekiq에서 실패

---

### C.9 MVC 패턴

```
┌─────────────────────────────────────────────────────────┐
│                      사용자 요청                         │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│ Controller (app/controllers/)                           │
│ - 요청 받아서 처리                                       │
│ - Model에 데이터 요청                                    │
│ - View에 결과 전달                                       │
└─────────────────────────────────────────────────────────┘
          │                               │
          ▼                               ▼
┌──────────────────────┐    ┌──────────────────────────────┐
│ Model (app/models/)  │    │ View (app/views/)            │
│ - DB 접근            │    │ - HTML 생성                   │
│ - 비즈니스 로직      │    │ - 사용자에게 보여줄 화면      │
│ - ApplicationSetting │    │                              │
│ - Project            │    │                              │
└──────────────────────┘    └──────────────────────────────┘
```

`ApplicationSetting`, `Project`는 **Model** 클래스.

---

### C.10 Rake Task란?

Ruby 빌드 도구. 반복 작업을 명령어로 정의:

```ruby
# lib/tasks/example.rake
namespace :gitlab do
  namespace :doctor do
    desc "Check secrets health"
    task secrets: :environment do
      # 암호화 필드 점검 로직
    end
  end
end
```

```bash
# 실행
rake gitlab:doctor:secrets

# GitLab에서는 gitlab-rake 사용 (환경 자동 설정)
sudo gitlab-rake gitlab:doctor:secrets
```

---

### C.11 ActiveRecord 메서드 비교

```ruby
# find - 단일 레코드 조회 (없으면 에러)
User.find(1)  # id=1인 User 반환

# where - 조건 검색 (결과는 배열)
User.where(active: true)  # 모든 active 유저

# update_all - 조건에 맞는 모든 레코드 업데이트 (콜백 우회)
User.where(id: 1).update_all(name: "New")  # SQL 직접 실행

# update_columns - 단일 레코드 특정 컬럼만 업데이트 (콜백 우회)
user.update_columns(name: "New")  # 이 컬럼만 UPDATE

# save! - 레코드 저장 (콜백/validation 실행, 실패 시 예외)
user.name = "New"
user.save!

# update! - 속성 변경 + 저장 (콜백/validation 실행)
user.update!(name: "New")
```

| 메서드 | 콜백 | Validation | 예외 발생 |
|--------|------|------------|-----------|
| `update_all` | ❌ | ❌ | ❌ |
| `update_columns` | ❌ | ❌ | ❌ |
| `save` | ✅ | ✅ | ❌ (false 반환) |
| `save!` | ✅ | ✅ | ✅ |
| `update` | ✅ | ✅ | ❌ |
| `update!` | ✅ | ✅ | ✅ |

---

### C.12 perform vs perform_async

```ruby
# perform - 동기 실행 (즉시, 현재 프로세스에서)
GroupDestroyWorker.new.perform(group_id, user_id)
# → 완료될 때까지 대기
# → Rails Console에서 실행 불가 (Sidekiq 환경 필요)

# perform_async - 비동기 실행 (Redis에 넣고 바로 반환)
GroupDestroyWorker.perform_async(group_id, user_id)
# → Redis에 job 추가 후 즉시 반환
# → Sidekiq이 나중에 처리
# → job ID 반환 (예: "abc123")
```

---

### C.13 GitLab 로그 파일 위치

| 로그 파일 | 경로 | 내용 |
|-----------|------|------|
| Rails 프로덕션 | `/var/log/gitlab/gitlab-rails/production.log` | 웹 요청, 에러 |
| Sidekiq | `/var/log/gitlab/sidekiq/current` | 백그라운드 job |
| Nginx | `/var/log/gitlab/nginx/` | HTTP 접근/에러 |
| PostgreSQL | `/var/log/gitlab/postgresql/` | DB 쿼리, 에러 |
| Workhorse | `/var/log/gitlab/gitlab-workhorse/` | Git, 업로드 |

```bash
# 실시간 로그 보기
sudo gitlab-ctl tail sidekiq

# 최근 로그 검색
sudo grep "error" /var/log/gitlab/sidekiq/current | tail -20
```

---

### C.14 자주 쓰는 GitLab 관리 명령어

```bash
# 서비스 관리
sudo gitlab-ctl status          # 전체 상태 확인
sudo gitlab-ctl restart         # 전체 재시작
sudo gitlab-ctl restart sidekiq # Sidekiq만 재시작

# 설정 적용
sudo gitlab-ctl reconfigure     # gitlab.rb 변경 후 적용

# 상태 점검
sudo gitlab-rake gitlab:check                    # 전체 점검
sudo gitlab-rake gitlab:doctor:secrets VERBOSE=1 # 암호화 점검

# Rails 환경
sudo gitlab-rails console       # 대화형 콘솔
sudo gitlab-rails runner '...'  # 스크립트 실행
sudo gitlab-rails dbconsole     # PostgreSQL 직접 접속

# 백업/복원
sudo gitlab-backup create       # 백업 생성
sudo gitlab-backup restore      # 백업 복원
```