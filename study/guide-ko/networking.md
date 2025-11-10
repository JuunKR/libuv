# Networking

libuv의 네트워킹은 BSD 소켓 인터페이스를 직접 사용하는 것과 크게 다르지 않습니다.
일부는 더 쉽고, 모두 논블로킹이지만 개념은 동일합니다. 또한 libuv는 BSD 소켓 구조를
사용한 소켓 설정, DNS 조회, 다양한 소켓 매개변수 조정과 같은 성가시고 반복적이며
저수준 작업을 추상화하는 유틸리티 함수를 제공합니다.

``uv_tcp_t`` 및 ``uv_udp_t`` 구조는 네트워크 I/O에 사용됩니다.

.. NOTE::

  이 장의 코드 샘플은 특정 libuv API를 보여주기 위해 존재합니다. 좋은 품질의 코드 예제는
  아닙니다. 메모리를 누수시키고 항상 연결을 제대로 닫지 않습니다.

## TCP

TCP는 연결 지향적이고 스트림 프로토콜이므로 libuv 스트림 인프라를 기반으로 합니다.

### Server

서버 소켓은 다음 순서로 진행됩니다:

1. TCP 핸들을 ``uv_tcp_init``합니다.
2. ``uv_tcp_bind``로 바인딩합니다.
3. 클라이언트에 의해 새 연결이 설정될 때마다 콜백이 호출되도록 핸들에서 ``uv_listen``을
   호출합니다.
4. ``uv_accept``를 사용하여 연결을 수락합니다.
5. :ref:`스트림 작업 <buffers-and-streams>`을 사용하여 클라이언트와 통신합니다.

다음은 간단한 echo 서버입니다.

```c
// tcp-echo-server/main.c - The listen socket
// (코드 예제는 원본 참조)
```

사람이 읽을 수 있는 IP 주소, 포트 쌍에서 BSD 소켓 API에 필요한 sockaddr_in 구조로
변환하는 데 유틸리티 함수 ``uv_ip4_addr``가 사용되는 것을 볼 수 있습니다. 역변환은
``uv_ip4_name``을 사용하여 얻을 수 있습니다.

.. NOTE::

    ip4 함수에 대한 ``uv_ip6_*`` 유사체가 있습니다.

대부분의 설정 함수는 CPU 바인딩이므로 동기적입니다. ``uv_listen``은 libuv의 콜백
스타일로 돌아가는 곳입니다. 두 번째 인수는 백로그 큐입니다. 즉, 대기 중인 연결의 최대
길이입니다.

클라이언트에 의해 연결이 시작되면 콜백은 클라이언트 소켓에 대한 핸들을 설정하고
``uv_accept``를 사용하여 핸들을 연결해야 합니다. 이 경우 이 스트림에서 읽기에 대한
관심도 설정합니다.

나머지 함수 세트는 스트림 예제와 매우 유사하며 코드에서 찾을 수 있습니다. 소켓이 필요하지
않을 때 ``uv_close``를 호출하는 것을 기억하세요. 연결을 수락하는 데 관심이 없다면
``uv_listen`` 콜백에서도 이를 수행할 수 있습니다.

### Client

서버에서 bind/listen/accept를 수행하는 곳에서 클라이언트 측에서는 단순히
``uv_tcp_connect``를 호출하는 문제입니다. ``uv_listen``과 동일한 ``uv_connect_cb``
스타일 콜백이 ``uv_tcp_connect``에서 사용됩니다.

```c
uv_tcp_t* socket = (uv_tcp_t*)malloc(sizeof(uv_tcp_t));
uv_tcp_init(loop, socket);

uv_connect_t* connect = (uv_connect_t*)malloc(sizeof(uv_connect_t));

struct sockaddr_in dest;
uv_ip4_addr("127.0.0.1", 80, &dest);

uv_tcp_connect(connect, socket, (const struct sockaddr*)&dest, on_connect);
```

여기서 ``on_connect``는 연결이 설정된 후 호출됩니다. 콜백은 소켓을 가리키는 ``.handle``
멤버가 있는 ``uv_connect_t`` 구조를 받습니다.

## UDP

`User Datagram Protocol`_은 연결 없는 신뢰할 수 없는 네트워크 통신을 제공합니다.
따라서 libuv는 스트림을 제공하지 않습니다. 대신 libuv는 `uv_udp_t` 핸들(수신용)과
`uv_udp_send_t` 요청(송신용) 및 관련 함수를 통해 논블로킹 UDP 지원을 제공합니다.
그렇다고 해서 읽기/쓰기의 실제 API는 일반 스트림 읽기와 매우 유사합니다. UDP 사용
방법을 보기 위해 예제는 `DHCP`_ 서버에서 IP 주소를 얻는 첫 번째 단계인 DHCP Discover를
보여줍니다.

.. note::

    `udp-dhcp`를 **root**로 실행해야 합니다. 잘 알려진 포트 번호 1024 미만을 사용하기
    때문입니다.

먼저 포트 68(DHCP 클라이언트)의 모든 인터페이스에 바인딩하도록 수신 소켓을 설정하고
읽기를 시작합니다. 이것은 응답하는 모든 DHCP 서버의 응답을 읽습니다. 같은 포트에서
이 컴퓨터에서 실행 중인 다른 시스템 DHCP 클라이언트와 잘 작동하도록 UV_UDP_REUSEADDR
플래그를 사용합니다. 그런 다음 유사한 송신 소켓을 설정하고 ``uv_udp_send``를 사용하여
포트 67(DHCP 서버)에서 *브로드캐스트 메시지*를 보냅니다.

브로드캐스트 플래그를 설정하는 것이 **필수**입니다. 그렇지 않으면 ``EACCES`` 오류가
발생합니다. 전송되는 정확한 메시지는 이 책과 관련이 없으며 관심이 있으면 코드를 연구할
수 있습니다. 평소처럼 읽기 및 쓰기 콜백은 문제가 발생한 경우 < 0의 상태 코드를 받습니다.

UDP 소켓은 특정 피어에 연결되어 있지 않으므로 읽기 콜백은 패킷의 발신자에 대한 추가
매개변수를 받습니다.

``nread``는 읽을 데이터가 더 이상 없으면 0일 수 있습니다. ``addr``이 NULL이면 읽을
것이 없음을 나타냅니다(콜백은 아무것도 하지 않아야 함). NULL이 아니면 ``addr``의 호스트에서
빈 데이터그램을 받았음을 나타냅니다. ``flags`` 매개변수는 할당자가 제공한 버퍼가 데이터를
보유하기에 충분히 크지 않은 경우 ``UV_UDP_PARTIAL``일 수 있습니다. *이 경우 OS는 맞지
않는 데이터를 버립니다*(그것이 UDP입니다!).

### UDP Options

#### Time-to-live

소켓에서 전송된 패킷의 TTL은 ``uv_udp_set_ttl``을 사용하여 변경할 수 있습니다.

#### IPv6 stack only

IPv6 소켓은 IPv4 및 IPv6 통신 모두에 사용할 수 있습니다. 소켓을 IPv6로만 제한하려면
``uv_udp_bind``에 ``UV_UDP_IPV6ONLY`` 플래그를 전달하세요.

#### Multicast

소켓은 다음을 사용하여 멀티캐스트 그룹을 (구독 해제)할 수 있습니다:

```c
int uv_udp_set_membership(uv_udp_t* handle, const char* multicast_addr, const char* interface_addr, uv_membership membership);
```

여기서 ``membership``은 ``UV_JOIN_GROUP`` 또는 ``UV_LEAVE_GROUP``입니다.

멀티캐스팅의 개념은 `이 가이드`_에서 잘 설명되어 있습니다.

.. _this guide: https://www.tldp.org/HOWTO/Multicast-HOWTO-2.html

멀티캐스트 패킷의 로컬 루프백은 기본적으로 활성화되어 있습니다. ``uv_udp_set_multicast_loop``를
사용하여 끄세요.

멀티캐스트 패킷의 패킷 time-to-live는 ``uv_udp_set_multicast_ttl``을 사용하여
변경할 수 있습니다.

## Querying DNS

libuv는 비동기 DNS 해석을 제공합니다. 이를 위해 자체 ``getaddrinfo`` 대체품을 제공합니다.
콜백에서 검색된 주소에 대해 일반 소켓 작업을 수행할 수 있습니다. DNS 해석의 예를 보기 위해
Libera.chat에 연결해봅시다.

```c
// dns/main.c
// (코드 예제는 원본 참조)
```

``uv_getaddrinfo``가 0이 아닌 값을 반환하면 설정에 문제가 발생한 것이며 콜백이 전혀
호출되지 않습니다. ``uv_getaddrinfo``가 반환된 직후 모든 인수를 자유롭게 해제할 수
있습니다. `hostname`, `servname` 및 `hints` 구조는 `getaddrinfo man page <getaddrinfo_>`_에
문서화되어 있습니다. 콜백은 ``NULL``일 수 있으며, 이 경우 함수가 동기적으로 실행됩니다.

해석기 콜백에서 ``struct addrinfo(s)``의 연결된 목록에서 모든 IP를 선택할 수 있습니다.
이것은 또한 ``uv_tcp_connect``를 보여줍니다. 콜백에서 ``uv_freeaddrinfo``를 호출하는
것이 필요합니다.

libuv는 역함수 `uv_getnameinfo`_도 제공합니다.

.. _uv_getnameinfo: http://docs.libuv.org/en/v1.x/dns.html#c.uv_getnameinfo

## Network interfaces

시스템의 네트워크 인터페이스에 대한 정보는 ``uv_interface_addresses``를 사용하여
libuv를 통해 얻을 수 있습니다. 이 간단한 프로그램은 사용 가능한 필드에 대한 아이디어를
얻을 수 있도록 모든 인터페이스 세부 정보를 인쇄합니다. 이것은 서비스가 시작될 때 IP 주소에
바인딩하도록 허용하는 데 유용합니다.

```c
// interfaces/main.c
// (코드 예제는 원본 참조)
```

``is_internal``은 루프백 인터페이스에 대해 true입니다. 물리적 인터페이스에 여러 IPv4/IPv6
주소가 있는 경우 이름이 여러 번 보고되며 각 주소가 한 번씩 보고됩니다.

.. _User Datagram Protocol: https://en.wikipedia.org/wiki/User_Datagram_Protocol
.. _DHCP: https://tools.ietf.org/html/rfc2131

