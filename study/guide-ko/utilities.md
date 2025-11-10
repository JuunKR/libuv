# Utilities

이 장은 일반적인 작업에 유용한 도구와 기법을 카탈로그화합니다. `libev man page`_는
이미 libuv로 간단한 API 변경을 통해 채택할 수 있는 일부 패턴을 다룹니다. 또한 전체
장을 할애할 필요가 없는 libuv API의 일부를 다룹니다.

## Timers

타이머는 타이머가 시작된 후 특정 시간이 경과한 후 콜백을 호출합니다. libuv 타이머는
한 번만이 아니라 정기적인 간격으로 호출되도록 설정할 수도 있습니다.

간단한 사용은 감시자를 초기화하고 ``timeout`` 및 선택적 ``repeat``로 시작하는 것입니다.
타이머는 언제든지 중지할 수 있습니다.

```c
uv_timer_t timer_req;

uv_timer_init(loop, &timer_req);
uv_timer_start(&timer_req, callback, 5000, 2000);
```

반복 타이머를 시작하며, ``uv_timer_start`` 실행 후 5초(``timeout``) 후에 먼저 시작한
다음 2초(``repeat``)마다 반복합니다. 사용:

```c
uv_timer_stop(&timer_req);
```

타이머를 중지합니다. 이것은 콜백 내에서도 안전하게 사용할 수 있습니다.

반복 간격은 다음을 사용하여 언제든지 수정할 수 있습니다:

```c
uv_timer_set_repeat(uv_timer_t *timer, int64_t repeat);
```

**가능할 때** 적용됩니다. 이 함수가 타이머 콜백에서 호출되면 다음을 의미합니다:

* 타이머가 반복되지 않는 경우 타이머가 이미 중지되었습니다. 다시 ``uv_timer_start``를
  사용하세요.
* 타이머가 반복되는 경우 다음 타임아웃이 이미 스케줄되었으므로 타이머가 새 간격으로 전환되기
  전에 이전 반복 간격이 한 번 더 사용됩니다.

유틸리티 함수:

```c
int uv_timer_again(uv_timer_t *)
```

**반복 타이머에만** 적용되며 타이머를 중지한 다음 초기 ``timeout``과 ``repeat``를 모두
이전 ``repeat`` 값으로 설정하여 시작하는 것과 같습니다. 타이머가 시작되지 않은 경우
실패(오류 코드 ``UV_EINVAL``)하고 -1을 반환합니다.

실제 타이머 예제는 :ref:`참조 카운트 섹션 <reference-count>`에 있습니다.

## Event loop reference count

이벤트 루프는 활성 핸들이 있는 한 실행됩니다. 이 시스템은 모든 핸들이 시작될 때 이벤트
루프의 참조 카운트를 증가시키고 중지될 때 참조 카운트를 감소시키는 방식으로 작동합니다.
다음을 사용하여 핸들의 참조 카운트를 수동으로 변경할 수도 있습니다:

```c
void uv_ref(uv_handle_t*);
void uv_unref(uv_handle_t*);
```

이 함수는 감시자가 활성 상태일 때도 루프가 종료되도록 허용하거나 사용자 정의 객체를
사용하여 루프를 활성 상태로 유지하는 데 사용할 수 있습니다.

후자는 간격 타이머와 함께 사용할 수 있습니다. X초마다 실행되는 가비지 수집기가 있거나
네트워크 서비스가 다른 서비스에 주기적으로 하트비트를 보낼 수 있지만 모든 깨끗한 종료
경로나 오류 시나리오에서 중지할 필요가 없습니다. 또는 다른 모든 감시자가 완료되면
프로그램이 종료되기를 원합니다. 이 경우 생성 직후 타이머의 참조를 해제하여 실행 중인
유일한 감시자인 경우 ``uv_run``이 여전히 종료되도록 합니다.

이것은 node.js에서도 사용되며 일부 libuv 메서드가 JS API로 버블링되고 있습니다.
JS 객체당 ``uv_handle_t``(모든 감시자의 상위 클래스)가 생성되며 참조/참조 해제할 수
있습니다.

```c
// ref-timer/main.c
// (코드 예제는 원본 참조)
```

가비지 수집기 타이머를 초기화한 다음 즉시 ``unref``합니다. 9초 후 가짜 작업이 완료되면
가비지 수집기가 여전히 실행 중임에도 불구하고 프로그램이 자동으로 종료되는 것을 관찰하세요.

## Idler pattern

idle 핸들의 콜백은 이벤트 루프당 한 번 호출됩니다. idle 콜백은 매우 낮은 우선순위 활동을
수행하는 데 사용할 수 있습니다. 예를 들어, 유휴 기간 동안 개발자에게 분석을 위해 일일
애플리케이션 성능 요약을 전달하거나 애플리케이션의 CPU 시간을 사용하여 SETI 계산을 수행할
수 있습니다 :) idle 감시자는 GUI 애플리케이션에서도 유용합니다. 파일 다운로드에 이벤트
루프를 사용한다고 가정해봅시다. TCP 소켓이 여전히 설정 중이고 다른 이벤트가 없으면 이벤트
루프가 일시 중지(**블로킹**)되므로 진행 표시줄이 고정되고 사용자는 응답하지 않는 애플리케이션을
직면하게 됩니다. 이러한 경우 idle 감시자를 큐에 넣어 UI를 작동 상태로 유지하세요.

```c
// idle-compute/main.c
// (코드 예제는 원본 참조)
```

여기서 idle 감시자를 초기화하고 관심 있는 실제 이벤트와 함께 큐에 넣습니다. ``crunch_away``는
이제 사용자가 무언가를 입력하고 Return을 누를 때까지 반복적으로 호출됩니다. 그런 다음
루프가 입력 데이터를 처리하는 동안 짧은 시간 동안 중단된 다음 idle 콜백을 계속 호출합니다.

## Passing data to worker thread

``uv_queue_work``를 사용할 때 일반적으로 워커 스레드를 통해 복잡한 데이터를 전달해야
합니다. 해결책은 ``struct``를 사용하고 ``uv_work_t.data``를 가리키도록 설정하는 것입니다.
약간의 변형은 ``uv_work_t`` 자체를 이 구조의 첫 번째 멤버로 가지는 것입니다(배턴이라고
함). 이것은 하나의 free 호출로 작업 요청과 모든 데이터를 정리할 수 있게 합니다.

```c
struct ftp_baton {
    uv_work_t req;
    char *host;
    int port;
    char *username;
    char *password;
}

ftp_baton *baton = (ftp_baton*) malloc(sizeof(ftp_baton));
baton->req.data = (void*) baton;
baton->host = strdup("my.webhost.com");
baton->port = 21;
// ...

uv_queue_work(loop, &baton->req, ftp_session, ftp_cleanup);
```

여기서 배턴을 만들고 작업을 큐에 넣습니다.

이제 작업 함수는 필요한 데이터를 추출할 수 있습니다:

```c
void ftp_session(uv_work_t *req) {
    ftp_baton *baton = (ftp_baton*) req->data;

    fprintf(stderr, "Connecting to %s\n", baton->host);
}

void ftp_cleanup(uv_work_t *req) {
    ftp_baton *baton = (ftp_baton*) req->data;

    free(baton->host);
    // ...
    free(baton);
}
```

그런 다음 감시자도 해제하는 배턴을 해제합니다.

## External I/O with polling

일반적으로 타사 라이브러리는 자체 I/O를 처리하고 내부적으로 소켓 및 기타 파일을 추적합니다.
이 경우 표준 스트림 I/O 작업을 사용할 수 없지만 라이브러리는 여전히 libuv 이벤트 루프에
통합될 수 있습니다. 필요한 것은 라이브러리가 기본 파일 디스크립터에 액세스할 수 있게 하고
애플리케이션이 결정한 대로 작은 증분으로 작업을 처리하는 함수를 제공하는 것입니다. 그러나
일부 라이브러리는 그러한 액세스를 허용하지 않으며 전체 I/O 트랜잭션을 수행한 다음에만 반환하는
표준 블로킹 함수만 제공합니다. 이벤트 루프 스레드에서 이것을 사용하는 것은 현명하지 않으며
대신 :ref:`스레드 풀`을 사용하세요. 물론 이것은 라이브러리에 대한 세밀한 제어를 잃는 것을
의미합니다.

libuv의 ``uv_poll`` 섹션은 운영 체제 알림 메커니즘을 사용하여 파일 디스크립터를 감시합니다.
어떤 의미에서 libuv 자체가 구현하는 모든 I/O 작업도 ``uv_poll``과 같은 코드로 지원됩니다.
OS가 폴링되는 파일 디스크립터의 상태 변경을 감지할 때마다 libuv는 관련 콜백을 호출합니다.

여기서 libcurl_을 사용하여 파일을 다운로드하는 간단한 다운로드 관리자를 살펴봅니다. libcurl에
모든 제어를 주는 대신 libuv 이벤트 루프를 사용하고 논블로킹, 비동기 multi_ 인터페이스를
사용하여 libuv가 I/O 준비 상태를 알릴 때마다 다운로드를 진행합니다.

.. _libcurl: https://curl.haxx.se/libcurl/
.. _multi: https://curl.haxx.se/libcurl/c/libcurl-multi.html

```c
// uvwget/main.c - The setup
// (코드 예제는 원본 참조)
```

각 라이브러리가 libuv와 통합되는 방식은 다릅니다. libcurl의 경우 두 가지 콜백을 등록할
수 있습니다. 소켓 콜백 ``handle_socket``은 소켓의 상태가 변경될 때마다 호출되며 폴링을
시작해야 합니다. ``start_timeout``은 I/O 상태에 관계없이 libcurl을 앞으로 구동해야 하는
다음 타임아웃 간격을 알리기 위해 libcurl에 의해 호출됩니다. 이것은 libcurl이 오류를
처리하거나 다운로드를 진행하는 데 필요한 다른 작업을 수행할 수 있도록 합니다.

다운로더는 다음과 같이 호출됩니다:

    $ ./uvwget [url1] [url2] ...

따라서 각 인수를 URL로 추가합니다.

libcurl이 데이터를 파일에 직접 쓰도록 하지만 원하는 경우 훨씬 더 많은 것이 가능합니다.

``start_timeout``은 libcurl에 의해 처음으로 즉시 호출되므로 작업이 시작됩니다. 이것은
단순히 libuv `타이머 <#timers>`_를 시작하며 타임아웃될 때마다 ``CURL_SOCKET_TIMEOUT``과
함께 ``curl_multi_socket_action``을 구동합니다. ``curl_multi_socket_action``은 libcurl을
구동하는 것이며 소켓 상태가 변경될 때마다 호출하는 것입니다. 하지만 그 전에 ``handle_socket``이
호출될 때마다 소켓을 폴링해야 합니다.

소켓 fd ``s``와 ``action``에 관심이 있습니다. 모든 소켓에 대해 존재하지 않는 경우
``uv_poll_t`` 핸들을 만들고 ``curl_multi_assign``을 사용하여 소켓과 연결합니다. 이렇게
하면 콜백이 호출될 때마다 ``socketp``가 이를 가리킵니다.

다운로드가 완료되거나 실패하는 경우 libcurl은 폴링 제거를 요청합니다. 따라서 폴링을
중지하고 폴링 핸들을 해제합니다.

libcurl이 감시하기를 원하는 이벤트에 따라 ``UV_READABLE`` 또는 ``UV_WRITABLE``로
폴링을 시작합니다. 이제 libuv는 소켓이 읽기 또는 쓰기 준비가 될 때마다 폴링 콜백을 호출합니다.
같은 핸들에서 ``uv_poll_start``를 여러 번 호출하는 것은 허용되며 이벤트 마스크를 새 값으로
업데이트할 뿐입니다. ``curl_perform``은 이 프로그램의 핵심입니다.

먼저 타이머를 중지합니다. 간격 동안 일부 진행이 있었기 때문입니다. 그런 다음 콜백을 트리거한
이벤트에 따라 올바른 플래그를 설정합니다. 그런 다음 진행한 소켓과 발생한 이벤트에 대한 정보를
알리는 플래그와 함께 ``curl_multi_socket_action``을 호출합니다. 이 시점에서 libcurl은
작은 증분으로 모든 내부 작업을 수행하며 가능한 한 빨리 반환하려고 시도하며, 이것은 이벤트
프로그램이 메인 스레드에서 원하는 것입니다. libcurl은 전송 진행에 대한 메시지를 자체 큐에
계속 큐에 넣습니다. 우리의 경우 완료된 전송에만 관심이 있습니다. 따라서 이러한 메시지를 추출하고
전송이 완료된 핸들을 정리합니다.

## Check & Prepare watchers

TODO

## Loading libraries

libuv는 `공유 라이브러리`_를 동적으로 로드하기 위한 크로스 플랫폼 API를 제공합니다.
이것은 자체 플러그인/확장/모듈 시스템을 구현하는 데 사용할 수 있으며 node.js에서
바인딩에 대한 ``require()`` 지원을 구현하는 데 사용됩니다. 라이브러리가 올바른 심볼을
내보내는 한 사용법은 매우 간단합니다. 타사 코드를 로드할 때 정신 건강 및 보안 검사를
주의 깊게 수행하세요. 그렇지 않으면 프로그램이 예측할 수 없게 동작합니다. 이 예제는
플러그인 이름을 인쇄하는 것 외에는 아무것도 하지 않는 매우 간단한 플러그인 시스템을 구현합니다.

먼저 플러그인 작성자에게 제공되는 인터페이스를 살펴봅시다.

```c
// plugin/plugin.h
// (코드 예제는 원본 참조)
```

유사하게 플러그인 작성자가 애플리케이션에서 유용한 작업을 수행하는 데 사용할 수 있는
더 많은 함수를 추가할 수 있습니다. 이 API를 사용하는 샘플 플러그인:

```c
// plugin/hello.c
// (코드 예제는 원본 참조)
```

우리 인터페이스는 모든 플러그인이 애플리케이션에 의해 호출될 ``initialize`` 함수를
가져야 한다고 정의합니다. 이 플러그인은 공유 라이브러리로 컴파일되며 애플리케이션을 실행하여
로드할 수 있습니다:

    $ ./plugin libhello.dylib
    Loading libhello.dylib
    Registered plugin "Hello World!"

.. NOTE::

    공유 라이브러리 파일 이름은 플랫폼에 따라 다릅니다. Linux에서는 ``libhello.so``입니다.

이것은 ``uv_dlopen``을 사용하여 먼저 공유 라이브러리 ``libhello.dylib``를 로드하여
수행됩니다. 그런 다음 ``uv_dlsym``을 사용하여 ``initialize`` 함수에 액세스하고 호출합니다.

```c
// plugin/main.c
// (코드 예제는 원본 참조)
```

``uv_dlopen``은 공유 라이브러리에 대한 경로를 예상하고 불투명한 ``uv_lib_t`` 포인터를
설정합니다. 성공 시 0을 반환하고 오류 시 -1을 반환합니다. 오류 메시지를 얻으려면
``uv_dlerror``를 사용하세요.

``uv_dlsym``은 세 번째 인수에 두 번째 인수의 심볼에 대한 포인터를 저장합니다.
``init_plugin_function``은 애플리케이션의 플러그인에서 찾고 있는 종류의 함수에 대한
함수 포인터입니다.

.. _shared libraries: https://en.wikipedia.org/wiki/Shared_library

## TTY

텍스트 터미널은 오랫동안 `꽤 표준화된`_ 명령 세트로 기본 포맷팅을 지원했습니다. 이 포맷팅은
종종 프로그램에서 터미널 출력의 가독성을 향상시키는 데 사용됩니다. 예를 들어 ``grep --colour``.
libuv는 ``uv_tty_t`` 추상화(스트림) 및 모든 플랫폼에서 ANSI 이스케이프 코드를 구현하는
관련 함수를 제공합니다. 이것은 libuv가 ANSI 코드를 Windows 동등물로 변환하고 터미널 정보를
얻는 함수를 제공한다는 것을 의미합니다.

.. _pretty standardised: https://en.wikipedia.org/wiki/ANSI_escape_sequences

가장 먼저 할 일은 읽기/쓰기를 수행하는 파일 디스크립터로 ``uv_tty_t``를 초기화하는 것입니다.
이것은 다음으로 달성됩니다:

```c
int uv_tty_init(uv_loop_t*, uv_tty_t*, uv_file fd, int unused)
```

``unused`` 매개변수는 이제 자동 감지되고 무시됩니다. 이전에는 스트림에서 ``uv_read_start()``를
사용하기 위해 설정해야 했습니다.

그런 다음 모드를 *normal*로 설정하여 대부분의 TTY 포맷팅, 흐름 제어 및 기타 설정을 활성화하는
``uv_tty_set_mode``를 사용하는 것이 가장 좋습니다. 다른_ 모드도 사용 가능합니다.

.. _Other: http://docs.libuv.org/en/v1.x/tty.html#c.uv_tty_mode_t

프로그램이 종료될 때 터미널 상태를 복원하기 위해 ``uv_tty_reset_mode``를 호출하는 것을
기억하세요. 단지 좋은 매너입니다. 또 다른 좋은 매너는 리디렉션을 인식하는 것입니다. 사용자가
명령의 출력을 파일로 리디렉션하면 제어 시퀀스를 작성하지 않아야 합니다. 가독성과 ``grep``을
방해하기 때문입니다. 파일 디스크립터가 실제로 TTY인지 확인하려면 파일 디스크립터와 함께
``uv_guess_handle``을 호출하고 반환 값을 ``UV_TTY``와 비교하세요.

다음은 빨간 배경에 흰색 텍스트를 인쇄하는 간단한 예입니다:

```c
// tty/main.c
// (코드 예제는 원본 참조)
```

최종 TTY 헬퍼는 ``uv_tty_get_winsize()``이며 터미널의 너비와 높이를 얻는 데 사용되며
성공 시 ``0``을 반환합니다. 다음은 이 함수와 문자 위치 이스케이프 코드를 사용하여 일부
애니메이션을 수행하는 작은 프로그램입니다.

```c
// tty-gravity/main.c
// (코드 예제는 원본 참조)
```

이스케이프 코드는:

======  =======================
Code    Meaning
======  =======================
*2* J    화면의 일부 지우기, 2는 전체 화면
H        커서를 특정 위치로 이동, 기본값은 왼쪽 위
*n* B    커서를 n줄 아래로 이동
*n* C    커서를 n열 오른쪽으로 이동
m        표시 설정 문자열을 따름, 이 경우 녹색 배경(40+2), 흰색 텍스트(30+7)
======  =======================

보시다시피 이것은 잘 포맷된 출력을 생성하거나 심지어 콘솔 기반 아케이드 게임을 만드는 데
매우 유용합니다. 더 멋진 제어를 위해 `ncurses`_를 시도할 수 있습니다.

.. _ncurses: https://invisible-island.net/ncurses/announce.html

.. versionchanged:: 1.23.1: `readable` 매개변수는 이제 사용되지 않으며 무시됩니다.
                    적절한 값은 이제 커널에서 자동 감지됩니다.

----

.. [#] 이 컨텍스트에서 배턴이라는 용어를 처음 접한 것은 Konstantin Käfer의 node.js
       바인딩 작성에 대한 훌륭한 슬라이드에서였습니다 --
       https://kkaefer.com/node-cpp-modules/#baton
.. [#] mfp는 My Fancy Plugin입니다

.. _libev man page: http://pod.tst.eu/http://cvs.schmorp.de/libev/ev.pod#COMMON_OR_USEFUL_IDIOMS_OR_BOTH

