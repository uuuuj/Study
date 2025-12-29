# Django get_response 이해하기

Django 공식 문서를 기반으로 `get_response`의 동작 원리를 정리한 문서입니다.

## get_response란?

Django 공식 문서에 따르면:

> **Middleware factory**는 `get_response` callable을 받아서 middleware를 반환하는 callable입니다.
> **Middleware**는 request를 받아서 response를 반환하는 callable입니다. (view와 동일)

```python
def simple_middleware(get_response):
    # 일회성 설정 및 초기화 (서버 시작 시 1번만 실행)
    
    def middleware(request):
        # View 호출 전에 실행되는 코드
        
        response = get_response(request)  # ★ 다음 단계로 요청 전달
        
        # View 호출 후에 실행되는 코드
        
        return response
    
    return middleware
```

## get_response의 정체

`get_response`는 **미들웨어 체인에서 다음 단계를 호출하는 callable**입니다.

- 현재 미들웨어가 **마지막이 아니면**: 다음 미들웨어
- 현재 미들웨어가 **마지막이면**: 실제 View 함수

```
                    get_response가 가리키는 대상
                    
Middleware_1  →  get_response  →  Middleware_2
Middleware_2  →  get_response  →  Middleware_3
Middleware_3  →  get_response  →  View (실제 뷰 함수)
```

## 양파(Onion) 모델

Django 공식 문서는 미들웨어를 **양파 껍질**에 비유합니다.

```
요청 방향 →
┌─────────────────────────────────────────────────────────────┐
│  Middleware 1 (가장 바깥)                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Middleware 2                                       │    │
│  │  ┌─────────────────────────────────────────────┐    │    │
│  │  │  Middleware 3                               │    │    │
│  │  │  ┌─────────────────────────────────────┐    │    │    │
│  │  │  │          View (핵심)                 │    │    │    │
│  │  │  └─────────────────────────────────────┘    │    │    │
│  │  └─────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                                                ← 응답 방향
```

**동작 방식:**
1. 요청은 **위에서 아래로** (MIDDLEWARE 리스트 순서대로) 전달
2. 각 미들웨어는 `get_response(request)`를 호출해 다음 단계로 전달
3. View에서 응답 생성
4. 응답은 **아래에서 위로** (역순) 전달

## 클래스 기반 미들웨어에서의 get_response

```python
class SimpleMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
        # 일회성 설정 (서버 시작 시)
    
    def __call__(self, request):
        # 요청 처리 전 (View 호출 전)
        print(f"Before view: {request.path}")
        
        response = self.get_response(request)  # ★ 다음 단계 호출
        
        # 응답 처리 후 (View 호출 후)
        print(f"After view: {response.status_code}")
        
        return response
```

### __init__ vs __call__

| 메서드 | 호출 시점 | 용도 |
|--------|----------|------|
| `__init__` | 웹 서버 시작 시 **1번만** | 초기화, 설정 |
| `__call__` | **매 요청마다** | 실제 요청/응답 처리 |

## Short-circuit (단락)

미들웨어가 `get_response`를 호출하지 않으면, **그 안쪽의 모든 미들웨어와 View는 실행되지 않습니다**.

```python
class BlockMobileMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        if 'Mobile' in request.META.get('HTTP_USER_AGENT', ''):
            # get_response를 호출하지 않음 → View까지 도달 X
            return HttpResponse("Mobile not supported", status=400)
        
        return self.get_response(request)  # 정상 흐름
```

## 예외 처리

Django는 미들웨어나 View에서 발생한 예외를 자동으로 HTTP 응답으로 변환합니다.

```python
# 미들웨어에서 try/except가 필요 없음!
response = self.get_response(request)

# 다음 미들웨어에서 Http404가 발생해도
# 여기서는 status_code=404인 HttpResponse를 받게 됨
```

## 전체 요청/응답 흐름에서 get_response 위치

```
클라이언트 요청
      │
      ▼
┌─────────────────────────────────────────────────────────┐
│  WSGIHandler.__call__(environ, start_response)          │
│                                                         │
│  request = WSGIRequest(environ)                         │
│  response = self.get_response(request)  ◀── 여기!       │
│                                                         │
└─────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────┐
│  BaseHandler.get_response(request)                      │
│                                                         │
│  → 미들웨어 체인 시작                                     │
│  → 각 미들웨어가 get_response 호출                        │
│  → 최종적으로 View 호출                                   │
│  → 응답 반환                                             │
└─────────────────────────────────────────────────────────┘
      │
      ▼
클라이언트 응답
```

## 실제 사용 예시: 로깅 미들웨어

```python
import logging
import time

logger = logging.getLogger(__name__)

class RequestLoggingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        # 요청 시작 시간
        start_time = time.time()
        
        # 다음 단계 호출 (다음 미들웨어 또는 View)
        response = self.get_response(request)
        
        # 응답 후 로깅
        duration = time.time() - start_time
        logger.info(
            f"{request.method} {request.path} "
            f"- {response.status_code} "
            f"- {duration:.2f}s"
        )
        
        return response
```

## 핵심 정리

| 개념 | 설명 |
|------|------|
| `get_response` | 미들웨어 체인에서 다음 단계(다음 미들웨어 또는 View)를 호출하는 callable |
| 호출 시점 | `__init__`은 서버 시작 시 1번, `__call__`은 매 요청마다 |
| 양파 모델 | 요청은 바깥→안쪽, 응답은 안쪽→바깥쪽으로 흐름 |
| Short-circuit | `get_response`를 호출하지 않으면 안쪽 레이어 실행 안 됨 |
| 예외 처리 | Django가 자동으로 예외를 HttpResponse로 변환 |

## 참고 자료

- [Django 공식 문서 - Middleware](https://docs.djangoproject.com/en/5.2/topics/http/middleware/)