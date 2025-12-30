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

### 10.2 CI 서명 키 수동 재생성 (핵심 조치)

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

### 10.3 서비스 재시작

```bash
sudo gitlab-ctl restart
```

### 10.4 재진단

```bash
sudo gitlab-rake gitlab:doctor:secrets VERBOSE=1
```

#### 기대 결과

```
ApplicationSetting failures : 0
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

> GitLab 그룹 삭제 실패의 원인은 **ApplicationSetting 암호화 키 불일치로 인한 Sidekiq 복호화 실패**였으며, Runner 미사용 환경에서는 CI 서명 키를 재생성하는 것이 가장 안전하고 현실적인 해결책이었다.