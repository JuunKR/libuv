# Filesystem

간단한 파일 시스템 읽기/쓰기는 ``uv_fs_*`` 함수와 ``uv_fs_t`` 구조를 사용하여
달성됩니다.

.. note::

    libuv 파일 시스템 작업은 :doc:`소켓 작업 <networking>`과 다릅니다. 소켓 작업은 운영
    체제에서 제공하는 논블로킹 작업을 사용합니다. 파일 시스템 작업은 내부적으로 블로킹
    함수를 사용하지만 이러한 함수를 `스레드 풀`_에서 호출하고 애플리케이션 상호 작용이
    필요할 때 이벤트 루프에 등록된 감시자에게 알립니다.

.. _thread pool: https://docs.libuv.org/en/v1.x/threadpool.html#thread-pool-work-scheduling

모든 파일 시스템 함수에는 두 가지 형태가 있습니다 - *동기* 및 *비동기*.

*동기* 형태는 콜백이 null이면 자동으로 호출되고(**블로킹**) 함수의 반환 값은
:ref:`libuv 오류 코드 <libuv-error-handling>`입니다. 이것은 일반적으로 동기 호출에만
유용합니다. *비동기* 형태는 콜백이 전달될 때 호출되며 반환 값은 0입니다.

## Reading/Writing files

파일 디스크립터는 다음을 사용하여 얻습니다:

```c
int uv_fs_open(uv_loop_t* loop, uv_fs_t* req, const char* path, int flags, int mode, uv_fs_cb cb)
```

``flags`` 및 ``mode``는 표준 `Unix flags <https://man7.org/linux/man-pages/man2/open.2.html>`_입니다.
libuv는 적절한 Windows 플래그로 변환하는 것을 처리합니다.

파일 디스크립터는 다음을 사용하여 닫습니다:

```c
int uv_fs_close(uv_loop_t* loop, uv_fs_t* req, uv_file file, uv_fs_cb cb)
```

파일 시스템 작업 콜백의 시그니처:

```c
void callback(uv_fs_t* req);
```

간단한 ``cat`` 구현을 봅시다. 파일이 열릴 때 콜백을 등록하는 것으로 시작합니다:

```c
// uvcat/main.c - opening a file
// (코드 예제는 원본 참조)
```

``uv_fs_t``의 ``result`` 필드는 ``uv_fs_open`` 콜백의 경우 파일 디스크립터입니다.
파일이 성공적으로 열리면 읽기를 시작합니다.

읽기 호출의 경우 읽기 콜백이 트리거되기 전에 데이터로 채워질 *초기화된* 버퍼를 전달해야
합니다. ``uv_fs_*`` 작업은 거의 직접적으로 특정 POSIX 함수에 매핑되므로 이 경우 EOF는
``result``가 0인 것으로 표시됩니다. 스트림이나 파이프의 경우 ``UV_EOF`` 상수가 상태
대신 전달되었을 것입니다.

여기서 비동기 프로그램을 작성할 때 일반적인 패턴을 볼 수 있습니다. ``uv_fs_close()``
호출이 동기적으로 수행됩니다. *일회성 작업이거나 시작 또는 종료 단계의 일부로 수행되는
작업은 일반적으로 동기적으로 수행됩니다. 프로그램이 주요 작업을 수행하고 여러 I/O 소스를
처리할 때 빠른 I/O에 관심이 있기 때문입니다*. 단독 작업의 경우 성능 차이는 일반적으로
무시할 수 있으며 더 간단한 코드로 이어질 수 있습니다.

파일 시스템 쓰기는 ``uv_fs_write()``를 사용하여 유사하게 간단합니다. *쓰기가 완료된 후
콜백이 트리거됩니다*. 우리의 경우 콜백은 단순히 다음 읽기를 구동합니다. 따라서 읽기와
쓰기는 콜백을 통해 단계적으로 진행됩니다.

.. warning::

    파일 시스템 및 디스크 드라이브가 성능을 위해 구성되는 방식으로 인해 '성공'한 쓰기가
    아직 디스크에 커밋되지 않았을 수 있습니다.

``main()``에서 도미노를 굴립니다:

```c
// uvcat/main.c
// (코드 예제는 원본 참조)
```

.. warning::

    ``uv_fs_req_cleanup()`` 함수는 libuv의 내부 메모리 할당을 해제하기 위해 파일 시스템
    요청에서 항상 호출되어야 합니다.

## Filesystem operations

``unlink``, ``rmdir``, ``stat``과 같은 모든 표준 파일 시스템 작업이 비동기적으로
지원되며 직관적인 인수 순서를 따릅니다. 읽기/쓰기/열기 호출과 동일한 패턴을 따르며
``uv_fs_t.result`` 필드에 결과를 반환합니다.

## Buffers and Streams

libuv의 기본 I/O 핸들은 스트림(``uv_stream_t``)입니다. TCP 소켓, UDP 소켓, 파일 I/O 및
IPC용 파이프는 모두 스트림 하위 클래스로 처리됩니다.

스트림은 각 하위 클래스에 대한 사용자 정의 함수를 사용하여 초기화된 다음 다음을 사용하여
작동합니다:

```c
int uv_read_start(uv_stream_t*, uv_alloc_cb alloc_cb, uv_read_cb read_cb);
int uv_read_stop(uv_stream_t*);
int uv_write(uv_write_t* req, uv_stream_t* handle,
             const uv_buf_t bufs[], unsigned int nbufs, uv_write_cb cb);
```

스트림 기반 함수는 파일 시스템 함수보다 사용하기 쉽고 libuv는 ``uv_read_start()``가
한 번 호출되면 ``uv_read_stop()``이 호출될 때까지 스트림에서 자동으로 계속 읽습니다.

데이터의 개별 단위는 버퍼입니다 -- ``uv_buf_t``. 이것은 단순히 바이트에 대한 포인터
(``uv_buf_t.base``)와 길이(``uv_buf_t.len``)의 모음입니다. ``uv_buf_t``는 가볍고
값으로 전달됩니다. 관리가 필요한 것은 실제 바이트이며, 애플리케이션에서 할당하고 해제해야
합니다.

스트림을 시연하기 위해 ``uv_pipe_t``를 사용해야 합니다. 이것은 로컬 파일을 스트리밍할
수 있게 합니다. 다음은 libuv를 사용하는 간단한 tee 유틸리티입니다. 모든 작업을 비동기적으로
수행하면 이벤트 기반 I/O의 힘을 보여줍니다. 두 쓰기는 서로를 블로킹하지 않지만 버퍼가
쓰여질 때까지 버퍼를 해제하지 않도록 버퍼 데이터를 복사하는 데 주의해야 합니다.

## File change events

모든 현대 운영 체제는 개별 파일이나 디렉토리에 감시를 설정하고 파일이 수정될 때 알림을
받을 수 있는 API를 제공합니다. libuv는 일반적인 파일 변경 알림 라이브러리를 래핑합니다.
이것은 libuv의 더 일관성 없는 부분 중 하나입니다. 파일 변경 알림 시스템 자체가 플랫폼
전반에 걸쳐 매우 다양하므로 모든 곳에서 모든 것을 작동시키는 것은 어렵습니다.

파일 변경 알림은 ``uv_fs_event_init()``을 사용하여 시작됩니다:

```c
// onchange/main.c - The setup
// (코드 예제는 원본 참조)
```

세 번째 인수는 모니터링할 실제 파일 또는 디렉토리입니다. 마지막 인수인 ``flags``는
다음과 같을 수 있습니다:

```c
enum uv_fs_event_flags {
    UV_FS_EVENT_WATCH_ENTRY = 1,
    UV_FS_EVENT_STAT = 2,
    UV_FS_EVENT_RECURSIVE = 4
};
```

``UV_FS_EVENT_WATCH_ENTRY`` 및 ``UV_FS_EVENT_STAT``는 아직 아무것도 하지 않습니다.
``UV_FS_EVENT_RECURSIVE``는 지원되는 플랫폼에서 하위 디렉토리도 감시하기 시작합니다.

콜백은 다음 인수를 받습니다:

1. ``uv_fs_event_t *handle`` - 핸들. 핸들의 ``path`` 필드는 감시가 설정된 파일입니다.
2. ``const char *filename`` - 디렉토리가 모니터링되는 경우 이것이 변경된 파일입니다.
   Linux와 Windows에서만 NULL이 아닙니다. 해당 플랫폼에서도 NULL일 수 있습니다.
3. ``int events`` - ``UV_RENAME`` 또는 ``UV_CHANGE`` 중 하나이거나 둘 다의 비트 OR.
4. ``int status`` - ``status < 0``이면 :ref:`libuv 오류 <libuv-error-handling>`가
   있습니다.

우리 예제에서는 단순히 인수를 인쇄하고 ``system()``을 사용하여 명령을 실행합니다.

