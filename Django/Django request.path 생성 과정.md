# Nginx → Gunicorn → Django request.path 생성 과정

Django의 `request.path`가 Nginx 프록시 환경에서 어떻게 만들어지는지 단계별로 정리한 문서입니다.

## 전체 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────┐
│  클라이언트                                                               │
│  GET http://172.123.123.123:8585/api/users/?page=1                      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Nginx (172.123.123.123:8585)                                           │
│  - 리버스 프록시                                                          │
│  - 정적 파일 서빙                                                         │
│  - SSL 종료                                                              │
└─────────────────────────────────────────────────────────────────────────┘
                                    │ proxy_pass http://127.0.0.1:7000
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Gunicorn (127.0.0.1:7000)                                              │
│  - WSGI 서버                                                             │
│  - 워커 프로세스 관리                                                      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │ WSGI environ
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Django Application                                                      │
│  - WSGIRequest 생성                                                      │
│  - request.path 설정                                                     │
└─────────────────────────────────────────────────────────────────────────┘
```

## 단계별 상세 흐름

### 1단계: 클라이언트 → Nginx

클라이언트가 HTTP 요청을 전송합니다.

```
GET /api/users/?page=1 HTTP/1.1
Host: 172.123.123.123:8585
User-Agent: Mozilla/5.0
Accept: application/json
```

### 2단계: Nginx 설정 및 처리

```nginx
# /etc/nginx/sites-available/myapp.conf

upstream django_app {
    server 127.0.0.1:7000;
}

server {
    listen 8585;
    server_name 172.123.123.123;

    location / {
        proxy_pass http://django_app;
        
        # 중요한 헤더들 전달
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host:$server_port;
    }
}
```

**Nginx가 하는 일:**

1. 요청 수신: `GET /api/users/?page=1`
2. `location /` 매칭
3. 프록시 헤더 추가/수정
4. `127.0.0.1:7000`으로 요청 전달

### 3단계: Nginx → Gunicorn 전달

Nginx가 Gunicorn에게 보내는 실제 HTTP 요청:

```
GET /api/users/?page=1 HTTP/1.1
Host: 172.123.123.123
X-Real-IP: 203.0.113.50          ← 실제 클라이언트 IP
X-Forwarded-For: 203.0.113.50
X-Forwarded-Proto: http
X-Forwarded-Host: 172.123.123.123:8585
Connection: close
```

### 4단계: Gunicorn이 WSGI environ 생성

```python
# Gunicorn이 생성하는 WSGI environ 딕셔너리

environ = {
    # ★ PATH 관련 - request.path의 원천
    'PATH_INFO': '/api/users/',
    'QUERY_STRING': 'page=1',
    'SCRIPT_NAME': '',
    
    # 요청 메타 정보
    'REQUEST_METHOD': 'GET',
    'SERVER_NAME': '127.0.0.1',
    'SERVER_PORT': '7000',
    'SERVER_PROTOCOL': 'HTTP/1.1',
    
    # Nginx가 추가한 헤더들 (HTTP_ 접두사 붙음)
    'HTTP_HOST': '172.123.123.123',
    'HTTP_X_REAL_IP': '203.0.113.50',
    'HTTP_X_FORWARDED_FOR': '203.0.113.50',
    'HTTP_X_FORWARDED_PROTO': 'http',
    'HTTP_X_FORWARDED_HOST': '172.123.123.123:8585',
    
    # WSGI 관련
    'wsgi.input': '<파일 객체>',
    'wsgi.url_scheme': 'http',
    # ...
}
```

### 5단계: Django WSGIRequest 생성

```python
# django/core/handlers/wsgi.py

class WSGIRequest(HttpRequest):
    def __init__(self, environ):
        self.environ = environ
        self.META = environ
        
        # ★ PATH 설정
        self.path = environ.get('PATH_INFO', '/')  # '/api/users/'
        self.path_info = self.path
        
        # 기타 속성들
        self.method = environ['REQUEST_METHOD']    # 'GET'
        # ...
```

### 6단계: 최종 request 객체 상태

```python
# views.py에서 접근 가능한 request 객체

def my_view(request):
    # PATH 관련
    request.path              # '/api/users/'
    request.path_info         # '/api/users/'
    request.get_full_path()   # '/api/users/?page=1'
    
    # META에서 Nginx 헤더 접근
    request.META['HTTP_HOST']              # '172.123.123.123'
    request.META['HTTP_X_REAL_IP']         # '203.0.113.50' (실제 클라이언트)
    request.META['HTTP_X_FORWARDED_FOR']   # '203.0.113.50'
    request.META['HTTP_X_FORWARDED_PROTO'] # 'http'
    request.META['SERVER_NAME']            # '127.0.0.1' (Gunicorn)
    request.META['SERVER_PORT']            # '7000'
    
    # Django 편의 메서드
    request.get_host()        # '172.123.123.123' (HTTP_HOST 사용)
```

## 흐름 요약 다이어그램

```
클라이언트: GET http://172.123.123.123:8585/api/users/?page=1
                │
                ▼
┌───────────────────────────────────────────────────────────┐
│  Nginx (172.123.123.123:8585)                             │
│                                                           │
│  1. URL 파싱: /api/users/?page=1                          │
│  2. location / 매칭                                       │
│  3. 헤더 추가 (X-Real-IP, X-Forwarded-*)                  │
│  4. proxy_pass → 127.0.0.1:7000                          │
└───────────────────────────────────────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────────────────────┐
│  Gunicorn (127.0.0.1:7000)                                │
│                                                           │
│  1. HTTP 요청 파싱                                         │
│  2. WSGI environ 딕셔너리 생성                             │
│     - PATH_INFO = '/api/users/'                           │
│     - QUERY_STRING = 'page=1'                             │
│     - HTTP_X_REAL_IP = '203.0.113.50'                     │
│  3. Django 애플리케이션 호출                                │
└───────────────────────────────────────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────────────────────┐
│  Django                                                   │
│                                                           │
│  WSGIRequest(environ)                                     │
│     - self.path = environ['PATH_INFO']                    │
│     - self.META = environ                                 │
│                                                           │
│  ★ 최종 결과:                                              │
│     request.path = '/api/users/'                          │
└───────────────────────────────────────────────────────────┘
```

## 실무 팁

### 실제 클라이언트 IP 가져오기

Nginx 뒤에 있으면 `REMOTE_ADDR`이 `127.0.0.1`이 되므로, 실제 IP는 다음과 같이 가져와야 합니다.

```python
def get_client_ip(request):
    x_forwarded_for = request.META.get('HTTP_X_FORWARDED_FOR')
    if x_forwarded_for:
        ip = x_forwarded_for.split(',')[0].strip()
    else:
        ip = request.META.get('HTTP_X_REAL_IP') or request.META.get('REMOTE_ADDR')
    return ip
```

### Django 설정

```python
# settings.py
USE_X_FORWARDED_HOST = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
```

## path 관련 속성 비교

| 속성 | 값 | 설명 |
|------|-----|------|
| `request.path` | `/api/users/` | SCRIPT_NAME + PATH_INFO |
| `request.path_info` | `/api/users/` | PATH_INFO만 |
| `request.get_full_path()` | `/api/users/?page=1` | 쿼리스트링 포함 |

## 핵심 요약

1. **Nginx**: URL을 받아 `proxy_pass`로 Gunicorn에 전달하며, `X-Forwarded-*` 헤더 추가
2. **Gunicorn**: HTTP 요청을 파싱하여 WSGI `environ` 딕셔너리 생성 (`PATH_INFO` 포함)
3. **Django**: `environ['PATH_INFO']`를 `request.path`에 할당

결국 `request.path`는 **WSGI 표준의 `PATH_INFO` 환경 변수**에서 직접 가져오는 것이고, 최초에는 Nginx가 원본 URL에서 쿼리스트링을 제외한 경로 부분을 파싱해서 만들어줍니다.