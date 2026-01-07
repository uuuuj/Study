# Django 테스트 및 유틸리티 함수 가이드

## 목차
1. [유틸리티 함수 작성 방식](#1-유틸리티-함수-작성-방식)
2. [DB 조회 방식 비교](#2-db-조회-방식-비교)
3. [유틸리티 함수 예시](#3-유틸리티-함수-예시)
4. [Django 테스트 기초](#4-django-테스트-기초)
5. [Factory Boy 사용법](#5-factory-boy-사용법)
6. [pytest 설정](#6-pytest-설정)

---

## 1. 유틸리티 함수 작성 방식

### 모델 메서드
이미 메모리에 로드된 객체의 속성을 검사할 때 사용

```python
# models.py
class Order(models.Model):
    status = models.CharField(max_length=20)
    
    def is_completed(self):
        return self.status == 'completed'
```

```python
# views.py
order = Order.objects.get(pk=1)  # DB 조회 1번
order.is_completed()              # DB 조회 없음 (이미 가진 데이터 사용)
```

### 유틸리티 함수
존재 여부/상태만 확인할 때 사용

```python
# utils.py
def check_order_status(order_id, expected_status):
    return Order.objects.filter(pk=order_id, status=expected_status).exists()
```

### Manager 메서드
복잡한 쿼리를 여러 곳에서 재사용할 때 사용

```python
# models.py
class OrderManager(models.Manager):
    def pending(self):
        return self.filter(status='pending')

class Order(models.Model):
    objects = OrderManager()
```

### 선택 기준

| 상황 | 추천 방식 |
|------|----------|
| 객체를 이미 가져온 상태에서 검사 | 모델 메서드 |
| 존재 여부/상태만 확인 | 유틸리티 함수 + `exists()` |
| 복잡한 쿼리를 여러 곳에서 재사용 | Manager 메서드 |

---

## 2. DB 조회 방식 비교

### exists()가 효율적인 이유

```python
# 방법 A: 객체 전체 조회
order = Order.objects.get(pk=1)  # SELECT * FROM order WHERE id=1
if order.status == 'completed':
    ...

# 방법 B: exists() 사용
Order.objects.filter(pk=1, status='completed').exists()
# SELECT 1 FROM order WHERE id=1 AND status='completed' LIMIT 1
```

| 비교 | get() | exists() |
|------|-------|----------|
| 가져오는 데이터 | 모든 컬럼 | 1 또는 없음 |
| 메모리 사용 | 객체 생성 | 없음 |
| 용도 | 데이터가 더 필요할 때 | 존재 여부만 확인할 때 |

### values_list 사용법

```python
# flat=True 없으면 → 튜플 리스트
Department.objects.values_list('name')
# [('개발팀',), ('인사팀',), ('마케팅팀',)]

# flat=True 있으면 → 값만 리스트
Department.objects.values_list('name', flat=True)
# ['개발팀', '인사팀', '마케팅팀']
```

| 메서드 | 반환 타입 | Serializer 필요 |
|--------|----------|-----------------|
| `objects.all()` | 객체 리스트 | API 응답 시 필요 |
| `objects.values()` | dict 리스트 | 불필요 |
| `objects.values_list()` | tuple/값 리스트 | 불필요 |

---

## 3. 유틸리티 함수 예시

### 상태 확인 함수

```python
# utils.py
from .models import UserProfile


def is_account_active(user_info: dict) -> bool:
    """
    계정 활성화 상태 확인
    
    Args:
        user_info: 사용자 정보가 담긴 JSON 데이터
            - user_id: 사용자 고유 ID
    
    Returns:
        is_active가 True면 True, False면 False 반환
    """
    user_id = user_info.get("user_id")
    return UserProfile.objects.filter(pk=user_id, is_active=True).exists()
```

### 데이터 복사 함수 (중복 방지 포함)

```python
# utils.py
from .models import Department, UserProfile


def copy_department_to_user_profile() -> bool:
    """
    Department의 name을 UserProfile에 row로 생성 (중복 방지)
    
    Returns:
        성공 시 True, 실패 시 False
    """
    try:
        source_values = set(
            Department.objects.values_list('name', flat=True)
        )
        
        existing_values = set(
            UserProfile.objects.values_list('name', flat=True)
        )
        
        new_values = source_values - existing_values
        
        if not new_values:
            return True
        
        new_rows = [
            UserProfile(
                name=value,
                is_active=True,
                created_by='system'
            ) 
            for value in new_values
        ]
        
        UserProfile.objects.bulk_create(new_rows)
        return True
        
    except Exception:
        return False
```

### utils.py vs views.py

| 상황 | 위치 |
|------|------|
| 그 view 파일에서만 사용 | views.py 상단 |
| 여러 view에서 사용 | utils.py |
| 테스트 코드에서 따로 테스트 | utils.py |

---

## 4. Django 테스트 기초

### 테스트 파일 구조

```python
# tests.py
from django.test import TestCase
from .models import Department, UserProfile
from .utils import copy_department_to_user_profile


class CopyDepartmentToUserProfileTest(TestCase):
    """copy_department_to_user_profile 함수 테스트"""
    
    def setUp(self):
        """테스트 데이터 생성 - 각 테스트 전에 실행"""
        self.dept1 = Department.objects.create(name='개발팀')
        self.dept2 = Department.objects.create(name='인사팀')
    
    def tearDown(self):
        """테스트 데이터 삭제 - 각 테스트 후에 실행"""
        Department.objects.filter(id__in=[self.dept1.id, self.dept2.id]).delete()
    
    def test_copy_success(self):
        """정상적으로 복사되는지 확인"""
        result = copy_department_to_user_profile()
        
        self.assertTrue(result)
        self.assertEqual(UserProfile.objects.count(), 2)
```

### 실행 순서

```
CopyDepartmentToUserProfileTest
├── setUp()
├── test_copy_success()
├── tearDown()
├── setUp()
├── test_duplicate_prevention()
├── tearDown()
```

- 클래스: 알파벳 순
- 메서드: 알파벳 순
- 매 테스트마다 `setUp()` → `test_xxx()` → `tearDown()` 반복

### 실행 방법

```bash
# 전체 테스트
python manage.py test

# 특정 앱만
python manage.py test app_name

# 특정 클래스만
python manage.py test app_name.tests.ClassName

# 특정 메서드만
python manage.py test app_name.tests.ClassName.test_method
```

### Assert 메서드

```python
self.assertTrue(result)                    # result가 True인지
self.assertFalse(result)                   # result가 False인지
self.assertEqual(a, b)                     # a == b 인지
self.assertEqual(count, 3, "에러 메시지")   # 두 번째 인자는 실패 시 메시지
```

### TestCase vs TransactionTestCase

| 종류 | 동작 | DB 데이터 |
|------|------|----------|
| `TestCase` | 롤백 | 테스트 끝나면 삭제 |
| `TransactionTestCase` | 커밋 | 테스트 끝나도 유지 |

---

## 5. Factory Boy 사용법

### 설치

```bash
pip install factory-boy
```

### 기본 사용법

```python
# factories.py
import factory
from factory.fuzzy import FuzzyChoice
from .models import UserProfile


class UserProfileFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = UserProfile
    
    user_id = factory.Sequence(lambda n: f'user{n:03d}')
    name = factory.Faker('name', locale='ko_KR')
    is_active = True
```

```python
# tests.py
from .factories import UserProfileFactory

# 하나 생성
user = UserProfileFactory()

# 여러 개 생성
users = UserProfileFactory.create_batch(10)

# 특정 값 지정
user = UserProfileFactory(is_active=False)
```

### 필드 타입별 예시

```python
import factory
from factory.fuzzy import FuzzyChoice, FuzzyInteger, FuzzyFloat, FuzzyDate
from django.utils import timezone


class UserProfileFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = UserProfile
    
    # CharField - 순차적
    user_id = factory.Sequence(lambda n: f'user{n:03d}')
    
    # CharField - 랜덤 이름
    name = factory.Faker('name', locale='ko_KR')
    
    # CharField - 고정값
    department = '개발팀'
    
    # CharField - 순환
    status = factory.Iterator(['active', 'pending', 'inactive'])
    
    # CharField - 랜덤 선택
    role = FuzzyChoice(['admin', 'user', 'guest'])
    
    # EmailField - 다른 필드 참조
    email = factory.LazyAttribute(lambda o: f'{o.user_id}@test.com')
    
    # IntegerField - 범위 내 랜덤
    score = FuzzyInteger(0, 100)
    
    # BooleanField
    is_active = True
    
    # DateTimeField - 현재
    created_at = factory.LazyFunction(timezone.now)
    
    # ForeignKey - 연관 모델 자동 생성
    department_obj = factory.SubFactory(DepartmentFactory)
```

### Iterator vs FuzzyChoice

```python
# Iterator - 순서대로 순환
name = factory.Iterator(['개발팀', '인사팀', '마케팅팀'])
# 첫 번째: 개발팀, 두 번째: 인사팀, 세 번째: 마케팅팀, 네 번째: 개발팀...

# FuzzyChoice - 랜덤 선택
name = FuzzyChoice(['개발팀', '인사팀', '마케팅팀'])
# 매번 랜덤
```

### django_get_or_create 사용 시 주의

```python
class ModelsFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = YourModel
        django_get_or_create = ('id',)  # 콤마 필수! (튜플)
    
    # Sequence는 lambda n 필수
    id = factory.Sequence(lambda n: f'U{n:08d}')
```

---

## 6. pytest 설정

### 설치

```bash
pip install pytest pytest-django pytest-html
```

### 설정 파일

```ini
# pytest.ini (manage.py 있는 위치)
[pytest]
DJANGO_SETTINGS_MODULE = config.settings
python_files = tests.py test_*.py
pythonpath = .
```

```python
# conftest.py (manage.py 있는 위치)
import os
import sys
import django

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')
django.setup()
```

### pytest.ini 설명

| 설정 | 의미 |
|------|------|
| `DJANGO_SETTINGS_MODULE` | Django 설정 파일 위치 |
| `python_files` | 테스트 파일 패턴 |
| `pythonpath = .` | 현재 폴더를 Python 경로에 추가 |

### conftest.py 설명

| 코드 | 역할 |
|------|------|
| `sys.path.insert(0, ...)` | import 경로 설정 |
| `os.environ.setdefault(...)` | Django 설정 위치 지정 |
| `django.setup()` | Django 앱, 모델 로드 |

### 테스트 DB 설정

```python
# config/settings.py 맨 아래 추가
import sys
if 'test' in sys.argv or 'pytest' in sys.modules:
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.sqlite3',
            'NAME': '/tmp/test_db.sqlite3',
        }
    }
```

### 실행

```bash
# Django 테스트
python manage.py test

# pytest + HTML 리포트
pytest sso_client/tests.py -v --html=report.html -s

# DB 유지
python manage.py test --keepdb
```

### DB 직접 확인

```bash
sqlite3 /tmp/test_db.sqlite3

.tables
SELECT * FROM sso_client_userprofile;
.quit
```

---

## 7. 기타 팁

### 타입 힌트

```python
def is_account_active(user_info: dict) -> bool:
    ...
```

- `user_info: dict` → 파라미터 타입
- `-> bool` → 반환 타입

### Model Meta 클래스

```python
class Response_user_info(models.Model):
    ...
    
    class Meta:
        app_label = 'sso_client'      # 앱 라벨 명시
        db_table = 'custom_table'     # 테이블명 직접 지정
        ordering = ['-created_at']    # 기본 정렬
```

### 딕셔너리 언패킹

```python
target_column = 'name'
value = '개발팀'

# 아래 두 개가 같은 의미
Model(**{target_column: value})
Model(name='개발팀')
```

### 폴더명 주의사항

Python에서 하이픈(-)은 마이너스 연산자로 인식되므로 폴더명에 사용 불가

```bash
dxp-dev/   # ❌ 안 됨
dxp_dev/   # ✅ 가능
```

---

## 참고 링크

- [Django Testing Documentation](https://docs.djangoproject.com/en/stable/topics/testing/)
- [Factory Boy Documentation](https://factoryboy.readthedocs.io/)
- [pytest-django Documentation](https://pytest-django.readthedocs.io/)