# Django resolve() 함수 이해하기

Django 공식 문서를 기반으로 `resolve()` 함수의 동작 원리를 정리한 문서입니다.

## resolve()란?

`resolve()` 함수는 **URL 경로를 해당하는 View 함수로 변환**하는 함수입니다.

```python
from django.urls import resolve

resolve(path, urlconf=None)
```

- `path`: 변환할 URL 경로 (예: `/api/users/`)
- `urlconf`: URLconf 모듈 (기본값: 현재 스레드의 root URLconf)

## 기본 사용법

```python
from django.urls import resolve

# URL을 resolve
match = resolve('/api/users/123/')

# View 함수 정보 확인
print(match.func)        # <function user_detail at 0x...>
print(match.url_name)    # 'user-detail'
print(match.args)        # ()
print(match.kwargs)      # {'pk': '123'}
```

## ResolverMatch 객체

`resolve()` 함수는 `ResolverMatch` 객체를 반환합니다. 이 객체는 다양한 메타데이터를 제공합니다.

### 주요 속성

| 속성 | 설명 | 예시 |
|------|------|------|
| `func` | 매칭된 View 함수 | `<function user_detail>` |
| `args` | URL에서 파싱된 위치 인자 | `('2024',)` |
| `kwargs` | URL에서 파싱된 키워드 인자 | `{'pk': '123'}` |
| `url_name` | URL 패턴의 이름 | `'user-detail'` |
| `route` | 매칭된 URL 패턴 | `'users/<int:pk>/'` |
| `app_name` | 애플리케이션 네임스페이스 | `'api'` |
| `namespace` | 인스턴스 네임스페이스 | `'api:v1'` |
| `view_name` | 네임스페이스 포함 View 이름 | `'api:user-detail'` |

### 상세 속성 설명

```python
from django.urls import resolve

match = resolve('/admin/auth/user/')

# View 관련
match.func              # 호출될 View 함수
match.args              # 위치 인자 튜플
match.kwargs            # 키워드 인자 딕셔너리
match.captured_kwargs   # URL에서 캡처된 키워드 인자
match.extra_kwargs      # 추가 키워드 인자

# URL 패턴 관련
match.url_name          # URL 패턴 이름
match.route             # URL 패턴 문자열
match.tried             # 매칭 시도한 URL 패턴 목록

# 네임스페이스 관련
match.app_name          # 애플리케이션 네임스페이스 (예: 'admin')
match.app_names         # 네임스페이스 리스트 ['admin']
match.namespace         # 인스턴스 네임스페이스
match.namespaces        # 인스턴스 네임스페이스 리스트
match.view_name         # 전체 View 이름 (네임스페이스 포함)
```

## 튜플로 언패킹

`ResolverMatch` 객체는 튜플로 언패킹할 수 있습니다.

```python
from django.urls import resolve

# 튜플로 언패킹
func, args, kwargs = resolve('/api/users/123/')

print(func)     # View 함수
print(args)     # 위치 인자
print(kwargs)   # 키워드 인자 {'pk': '123'}
```

## resolve() vs reverse()

| 함수 | 방향 | 입력 | 출력 |
|------|------|------|------|
| `resolve()` | URL → View | URL 경로 | View 함수 + 인자 |
| `reverse()` | View → URL | View 이름 + 인자 | URL 경로 |

```python
from django.urls import resolve, reverse

# resolve: URL → View
match = resolve('/api/users/123/')
# match.func = user_detail, match.kwargs = {'pk': '123'}

# reverse: View → URL
url = reverse('api:user-detail', kwargs={'pk': '123'})
# url = '/api/users/123/'
```

## 실용적인 사용 예시

### 1. 리다이렉트 전 404 체크

```python
from urllib.parse import urlparse
from django.urls import resolve
from django.http import Http404, HttpResponseRedirect

def myview(request):
    # Referer URL 가져오기
    next_url = request.META.get('HTTP_REFERER', None) or '/'
    
    # URL 파싱해서 경로만 추출
    path = urlparse(next_url)[2]  # '/api/users/123/'
    
    # resolve로 View 함수 가져오기
    view, args, kwargs = resolve(path)
    kwargs['request'] = request
    
    try:
        # View 함수 호출해서 404 체크
        view(*args, **kwargs)
    except Http404:
        # 404 발생하면 홈으로 리다이렉트
        return HttpResponseRedirect('/')
    
    return HttpResponseRedirect(next_url)
```

### 2. 현재 URL의 View 이름 확인

```python
def my_middleware(get_response):
    def middleware(request):
        # 현재 요청의 URL resolve
        match = resolve(request.path)
        
        # View 이름으로 특정 처리
        if match.url_name == 'admin-only':
            if not request.user.is_staff:
                return HttpResponseForbidden()
        
        return get_response(request)
    
    return middleware
```

### 3. URL 패턴 기반 권한 체크

```python
from django.urls import resolve

def check_permission(request):
    match = resolve(request.path)
    
    # 앱 네임스페이스로 권한 체크
    if match.app_name == 'admin':
        return request.user.is_staff
    
    # URL 이름으로 권한 체크
    if match.url_name in ['user-create', 'user-delete']:
        return request.user.has_perm('users.manage_users')
    
    return True
```

### 4. API 버전 확인

```python
from django.urls import resolve

def get_api_version(request):
    match = resolve(request.path)
    
    # namespace가 'api:v1' 또는 'api:v2' 형태일 때
    if 'v2' in match.namespaces:
        return 'v2'
    return 'v1'
```

## 예외 처리

URL이 매칭되지 않으면 `Resolver404` 예외가 발생합니다.

```python
from django.urls import resolve, Resolver404

try:
    match = resolve('/invalid/path/')
except Resolver404:
    print("URL을 찾을 수 없습니다")
```

## Django 요청 처리에서 resolve() 위치

```
클라이언트 요청: GET /api/users/123/
        │
        ▼
┌─────────────────────────────────────────────────────┐
│  WSGIHandler                                        │
│  request = WSGIRequest(environ)                     │
│  request.path = '/api/users/123/'                   │
└─────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────┐
│  BaseHandler.get_response(request)                  │
│                                                     │
│  ★ resolve(request.path)                            │
│     → ResolverMatch 반환                            │
│     → func: user_detail                             │
│     → kwargs: {'pk': '123'}                         │
└─────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────┐
│  View 호출                                          │
│  response = user_detail(request, pk='123')          │
└─────────────────────────────────────────────────────┘
        │
        ▼
클라이언트 응답
```

## URL 패턴 예시와 resolve 결과

```python
# urls.py
from django.urls import path

urlpatterns = [
    path('users/', views.user_list, name='user-list'),
    path('users/<int:pk>/', views.user_detail, name='user-detail'),
    path('users/<int:pk>/posts/<slug:slug>/', views.user_post, name='user-post'),
]
```

```python
# resolve 결과

match = resolve('/users/')
# match.func = user_list
# match.args = ()
# match.kwargs = {}
# match.url_name = 'user-list'
# match.route = 'users/'

match = resolve('/users/123/')
# match.func = user_detail
# match.args = ()
# match.kwargs = {'pk': 123}
# match.url_name = 'user-detail'
# match.route = 'users/<int:pk>/'

match = resolve('/users/123/posts/hello-world/')
# match.func = user_post
# match.args = ()
# match.kwargs = {'pk': 123, 'slug': 'hello-world'}
# match.url_name = 'user-post'
# match.route = 'users/<int:pk>/posts/<slug:slug>/'
```

## 핵심 정리

| 개념 | 설명 |
|------|------|
| `resolve()` | URL 경로를 View 함수와 인자로 변환 |
| `ResolverMatch` | 매칭 결과를 담은 객체 (func, args, kwargs 등) |
| `Resolver404` | URL 매칭 실패 시 발생하는 예외 |
| 튜플 언패킹 | `func, args, kwargs = resolve(path)` 형태로 사용 가능 |
| 역함수 | `reverse()`는 resolve의 반대 동작 (View → URL) |

## 참고 자료

- [Django 공식 문서 - django.urls utility functions](https://docs.djangoproject.com/en/5.2/ref/urlresolvers/)
- [Django 공식 문서 - URL dispatcher](https://docs.djangoproject.com/en/5.2/topics/http/urls/)