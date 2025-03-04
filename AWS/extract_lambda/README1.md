## 작동 방식

1. 사이트 별 검색 기능 활용하여 해당 키워드를 검색하고, 기간 내의 게시글의 url, 페이지 넘버를 수집함

2. 수집한 url을 방문해 크롤링하고자 하는 정보가 담겨있는 html Body 태그를 {"url": "html..."} 형태를 배열로 저장함

3. 1, 2번 과정을 4개의 multiprocessing Manager dictionary를 통해 4개의 함수를 비동기적으로 실행해 간접적인 멀티프로세싱을 구현함

4. 오류 발생 시 재시도하고, 그럼에도 불구하고 오류가 발생하는 경우 log_error를 통해 Step function에 오류를 전달하여 재시작함

## 구현 특이사항

- 동적 크롤링 구현을 위해 Playwright라는 E2E 프레임워크 사용
- AWS Lambda는 컨테이너 환경에서 구현되어 멀티프로세싱을 지원하지 않음. 크롤링은 Network IO Bound 작업이므로 대기 시간 동안 컴퓨팅 자원을 낭비하게 됨
- multiprocessing Queue는 지원하지 않으므로 처리할 수 있는 방법이 3가지였음.
    - Pipe
        - AWS Lambda에서 공식적으로 지원하는 방법
        - 프로세스가 처리하는 데이터 사이즈가 커지면 버퍼링이 걸리는 현상 발생
    - Async 구현
        - async Playwright 패키지를 이용하여 코루틴을 통한 비동기 처리 구현
        - 가장 속도 측면에서도 빠르고 멀티프로세싱을 지원하지 않는 Lambda 환경에 적합
        - 코드 가독성이 떨어지고, Browser Context가 Destroy되는 현상이 있어 안정성이 떨어짐
    - Multiprocessing Manager를 통한 구현
        - Manager를 이용한 Dict, List를 이용해 공유 딕셔너리, 공유 리스트를 통해 간접적으로 비동기 구현
        - 셋 중 가장 안정성이 좋았고, 코드 가독성이 높은 편이어서 여러 사이트에 재사용해야 하는 개발환경에서 적합했다.