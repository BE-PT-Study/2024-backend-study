## RateLimiter
* 단위 시간동안 얼만큼의 실행을 허용할 것인지 제한
* rateLimiter는 JVM의 시작부터의 모든 나노초를 사이클로 분할
* 각 사이클은 limitRefreshPeriod 시간동안 limitForPeriod 만큼의 요청만 허용

### AtomicRateLimiter
* RateLimiter의 default 구현체
* AtomicRateLimiter.State는 변경 불가능
    * activeCycle - 마지막 호출의 사이클 번호
    * activePermissions - 맘지막 호출 후 사용 가능한 호출 수
    * nanosToWait - 마지막 호출에 대한 사용 권한을 기다리는 시간(nanoSec)
### SemaphoreBasedRateLimiter
* Semaphore과 scheduler를 사용하여 권한 refresh

## Retry
실패한 실행을 재시도하는 메커니즘


### retry 구성
* maxAttempts - 최대 시도 횟수(default: 3)
* waitDuration - 재시도 호출 간격(default: 500ms)
* intervalFunction - 장애 후 waitDuration을 변경하는 function
* retryOnResultPredicate -
* retryExceptionPredicate
* retryExceptions
* ignoreExceptions
* failAfterMaxAttempts

## timelimiter
실행 시간을 제한
## Cache
결과를 캐싱
*주의할 점: 동시성 이슈로 인해 'JCache Reference Implementation' 사용 권장하지 않음