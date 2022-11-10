---
title: [UDS] Unix Domain Socket
date: 2022-11-10 21:07:25 +0900
categories: [Development, Etc]
tags: [UDS, Socket]
---
# UDS?

Unix Domain Socket, 짧게 줄여 UDS는 일반적으로 우리가 사용하는 TCP/UDP와는 조금 다른 개념이다. TCP/UDP는 네트워크 통신을 하는 반면 UDS는 파일시스템 내부의 **파일**을 이용해 통신한다.

네트워크를 사용하지 않고 어떻게 통신을 하느냐고 물을 수 있는데, 거기에 답이 있다. 다른 Host들 간 통신에는 반드시 네트워크가 필요한 반면에, Host 내부 통신이라면 네트워크가 필요없다는 것이다. 즉, UDS는 **네트워크 통신이 필요없는 Host 내부 프로세스 간 통신에 사용된다.**

UDS를 사용할 경우 기존에 사용한 TCP/UDP system call을 그대로 사용할 수 있다는 장점이 있다.

## 1. include

```
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
-----------------------------------------
#include <sys/un.h>
```

점선 위로는 기존 TCP/UDP 통신에 필요한 헤더이며, 점선 아래로는 UDS를 사용하기 위한 추가 헤더이다.

## 2. socket 생성

```
int socket(int domain, int type, int protocol);
```

기존의 함수를 그대로 사용하며, 매개변수도 동일한 유형이나 다음의 차이가 존재한다.

- domain

  기존 : PF_INET / PF_INET6(IPv6)

  변경 : PF_FILE

  ```
  int socket = socket(PF_FILE, SOCK_STREAM, 0);
  int socket = socket(PF_FILE, SOCK_DGRAM, 0);
  ```

## 3. bind

```
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

bind도 동일한 API를 사용하나 2번째 매개변수인 addr이 sockaddr_in에서 sockaddr_un으로 변경된다는 차이점이 있다.

```
struct sockaddr_un
  {
    __SOCKADDR_COMMON (sun_);
    (= sa_family_t sa_prefix;)
    char sun_path[108];        /* Path name.  */
  };

struct sockaddr_un servAddr;
memset(&servAddr, 0x00, sizeof(servAddr));

servAddr.sun_family = AF_UNIX;
snprintf(servAddr.sun_path, sizeof(servAddr.sun_path), "%s", FILE_NAME);

unsigned int servAddrLen = sizeof(servAddr);
```

- sun_family

  TCP/UDP에서 설정했던 family 종류와 동일하다. 다만, UDS이므로 **AF_UNIX 또는 AF_FILE 또는 AF_LOCAL**을 지정해야 한다.

  위의 3가지 family 변수는 모두 PF_LOCAL 변수의 alias이며 실제 값은 1이다.

  (The AF_UNIX (also known as AF_LOCAL) socket family is used to communicate between processes on the same machine efficiently.)

- sun_path

  통신에 사용될 파일시스템 내 파일 경로를 지정한다.

  snprintf를 사용해 대상 파일명을 입력하는 것이 일반적이다.

- `snprintf(servAddr.sun_path, sizeof(servAddr.sun_path), "%s", FILE_NAME);`

**주의해야 할 점은, UDS 소켓 파일은 bind 함수 실행 시점에 존재하지 않아야 한다.

그러므로 bind 실행 이전 소켓 파일의 존재 여부를 확인한 후 있을 경우 파일을 삭제해야 한다.**

```
if (access(FILE_NAME, F_OK) != 0) {
    unlink(FILE_NAME);
}

// 이후 bind 실행....
```

## 4. listen

```
int listen(int sockfd, int backlog);
```

변경사항 없이, 기존의 TCP/UDP와 동일하게 사용하면 된다.

## 5. connect

```
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

2번째 매개변수에 사용되는 구조체는 sockaddr_un 구조체이다.
일반 TCP/IP 소켓 프로그래밍처럼 형변환하여 사용한다.

```
int result = connect(serverSock, (struct sockaddr*)&servAddr, servAddrLen);
if (result == -1) {
  perror("connect() failed");
  return;
}
```

## 6. accept

```
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

2번째 매개변수에 사용되는 구조체는 sockaddr_un 구조체이다.

connect와 동일하게 형변환하여 사용하며, 필요하지 않을 경우에는 NULL을 넣어도 상관없음.

```
int clientSock = accept(servSock, (struct sockaddr*) &clientAddr, &clientAddrLen);
if (clientSock == -1) {
  perror("accept() failed");
  return;
}
```

## 7-1. send

```
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
```

## 7-2. write

```
ssize_t write(int fd, const void *buf, size_t count);
```

## 8-1. recv

```
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
```

## 8-2. read

```
ssize_t read(int fd, void *buf, size_t count);
```

위의 명시된 4개의 함수는 이미 생성된 소켓을 통해 데이터를 송/수신하는 데에 사용되므로

UDS - TCP/IP 통신의 성격에 영향을 받지 않는다.