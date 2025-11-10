# Introduction

이 '책'은 Windows와 Unix에서 동일한 API를 제공하는 고성능 이벤트 기반 I/O 라이브러리로
libuv_를 사용하는 방법에 대한 작은 튜토리얼 모음입니다.

libuv의 주요 영역을 다루고 있지만, 모든 함수와 데이터 구조를 논의하는 포괄적인 레퍼런스는
아닙니다. 전체 세부사항은 `공식 libuv 문서`_를 참조할 수 있습니다.

.. _공식 libuv 문서: https://docs.libuv.org/en/v1.x/

이 책은 아직 진행 중이므로 섹션이 불완전할 수 있지만, 성장하는 과정을 즐기시길 바랍니다.

## 이 책의 대상 독자

이 책을 읽고 있다면 다음 중 하나입니다:

1) 시스템 프로그래머로, 데몬이나 네트워크 서비스 및 클라이언트와 같은 저수준 프로그램을
   만듭니다. 이벤트 루프 접근 방식이 애플리케이션에 잘 맞는다고 판단하여 libuv를 사용하기로
   결정했습니다.

2) node.js 모듈 작성자로, C 또는 C++로 작성된 플랫폼 API를 JavaScript에 노출되는
   (비)동기 API 세트로 래핑하고 싶습니다. 순수하게 node.js 컨텍스트에서 libuv를 사용할
   것입니다. 이를 위해서는 이 책이 v8/node.js 특정 부분을 다루지 않으므로 다른 리소스가
   필요합니다.

이 책은 C 프로그래밍 언어에 익숙하다고 가정합니다.

## 배경

node.js_ 프로젝트는 2009년 브라우저에서 분리된 JavaScript 환경으로 시작되었습니다.
Google의 V8_과 Marc Lehmann의 libev_를 사용하여, node.js는 I/O 모델(이벤트 기반)을
브라우저에 의해 형성된 프로그래밍 스일에 잘 맞는 언어와 결합했습니다. node.js가 인기를
얻으면서 Windows에서 작동하도록 만드는 것이 중요했지만, libev는 Unix에서만 실행되었습니다.
kqueue나 (e)poll과 같은 커널 이벤트 알림 메커니즘의 Windows 동등물은 IOCP입니다.
libuv는 플랫폼에 따라 libev 또는 IOCP 주변의 추상화였으며, 사용자에게 libev 기반의
API를 제공했습니다. libuv의 node-v0.9.0 버전에서 `libev가 제거되었습니다`_.

그 이후 libuv는 계속 성숙해지고 시스템 프로그래밍을 위한 고품질 독립형 라이브러리가
되었습니다. node.js 외부의 사용자로는 Mozilla의 Rust_ 프로그래밍 언어와 다양한_ 언어
바인딩이 있습니다.

이 책과 코드는 libuv 버전 `v1.42.0`_을 기반으로 합니다.

## 코드

모든 예제 코드와 책의 소스는 GitHub의 libuv_ 프로젝트의 일부로 포함되어 있습니다.
libuv_를 클론하거나 다운로드한 다음 빌드하세요::

    sh autogen.sh
    ./configure
    make

``make install``할 필요는 없습니다. 예제를 빌드하려면 ``docs/code/`` 디렉토리에서
``make``를 실행하세요.

.. _v1.42.0: https://github.com/libuv/libuv/releases/tag/v1.42.0
.. _V8: https://v8.dev
.. _libev: http://software.schmorp.de/pkg/libev.html
.. _libuv: https://github.com/libuv/libuv
.. _node.js: https://www.nodejs.org
.. _libev was removed: https://github.com/joyent/libuv/issues/485
.. _Rust: https://www.rust-lang.org
.. _variety: https://github.com/libuv/libuv/blob/v1.x/LINKS.md

