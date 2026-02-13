---
title: "[CS230] 6. I/O, 네트워크, 동시성 - System I/O, Network, Concurrency"
date: 2022-12-15
draft: false
tags: ["CS230", "Systems Programming", "KAIST", "Unix IO", "Network", "Concurrency", "Threads"]
categories: ["Computer Science"]
summary: "Unix I/O(파일 디스크립터, RIO), 네트워크 프로그래밍(소켓, HTTP), 동시성 프로그래밍(프로세스, 스레드, 동기화)을 다룹니다."
ShowToc: true
TocOpen: true
---

> KAIST CS230 시스템프로그래밍 (2022 Fall) 수업 내용을 정리한 시리즈입니다.
> 교재: *Computer Systems: A Programmer's Perspective* (CS:APP 3rd Edition)

---

## 1. Unix I/O

### 모든 것은 파일이다

Linux에서 **모든 I/O 장치는 파일**로 표현된다:

| 파일 | 예시 |
|------|------|
| Regular file | `/home/user/hello.c` |
| Directory | `/home/user/` |
| Device | `/dev/sda2` (디스크), `/dev/tty2` (터미널) |
| Socket | 네트워크 통신용 |
| Pipe | 프로세스 간 통신용 |

커널은 이들을 통일된 인터페이스(**Unix I/O**)로 다룬다.

### File Descriptor

파일을 열면 커널이 정수값(file descriptor)을 반환한다:

| fd | 의미 |
|----|------|
| 0 | `stdin` (표준 입력) |
| 1 | `stdout` (표준 출력) |
| 2 | `stderr` (표준 에러) |
| 3+ | `open()`으로 열린 파일 |

### 핵심 시스템 콜

```c
#include <fcntl.h>
#include <unistd.h>

int fd = open("/path/to/file", O_RDONLY);  // 파일 열기
// flags: O_RDONLY, O_WRONLY, O_RDWR, O_CREAT, O_TRUNC, O_APPEND

ssize_t n = read(fd, buf, sizeof(buf));    // 읽기
// 최대 sizeof(buf) bytes 읽음, 실제 읽은 바이트 수 반환
// 0 반환 → EOF, -1 반환 → 에러

ssize_t n = write(fd, buf, len);           // 쓰기

off_t pos = lseek(fd, offset, SEEK_SET);   // 파일 위치 이동

int ret = close(fd);                       // 파일 닫기
```

### Short Count

`read()`와 `write()`는 요청한 것보다 **적은 바이트**를 전송할 수 있다 (short count):

- EOF 도달
- 터미널에서 텍스트 줄 읽기
- 네트워크 소켓에서 읽기/쓰기

→ 안전한 I/O를 위해 **RIO 패키지** 사용

### RIO (Robust I/O)

Short count를 자동으로 처리하는 래퍼 함수:

```c
// Unbuffered (바이너리 데이터용)
ssize_t rio_readn(int fd, void *buf, size_t n);   // 정확히 n bytes 읽기
ssize_t rio_writen(int fd, void *buf, size_t n);  // 정확히 n bytes 쓰기

// Buffered (텍스트 줄 읽기용)
void rio_readinitb(rio_t *rp, int fd);
ssize_t rio_readlineb(rio_t *rp, void *buf, size_t maxlen);
ssize_t rio_readnb(rio_t *rp, void *buf, size_t n);
```

### I/O Redirection

Shell에서의 입출력 방향 전환:

| 문법 | 의미 |
|------|------|
| `cmd > file` | stdout을 파일로 (덮어쓰기) |
| `cmd >> file` | stdout을 파일로 (추가) |
| `cmd < file` | stdin을 파일에서 |
| `cmd 2> file` | stderr를 파일로 |
| `cmd1 \| cmd2` | cmd1의 stdout을 cmd2의 stdin으로 (파이프) |

내부적으로 `dup2()` 시스템 콜을 사용:

```c
dup2(newfd, oldfd);  // oldfd가 newfd와 같은 파일을 가리키게 함
```

### File Types

```c
struct stat statbuf;
stat("/path/to/file", &statbuf);
```

- `S_ISREG()`: Regular file
- `S_ISDIR()`: Directory
- `S_ISSOCK()`: Socket

### Standard I/O vs Unix I/O

| | Standard I/O | Unix I/O |
|---|---|---|
| 헤더 | `<stdio.h>` | `<unistd.h>`, `<fcntl.h>` |
| 핸들 | `FILE *` | `int fd` |
| 버퍼링 | 있음 (자동) | 없음 |
| 함수 | `fopen`, `fread`, `fprintf` | `open`, `read`, `write` |
| 용도 | 일반적인 파일 I/O | 저수준 제어, 네트워크 |

> 네트워크 소켓에는 Standard I/O 사용을 피하라 — 버퍼링 문제 발생 가능.
> 네트워크 I/O에는 RIO 패키지를 사용하는 것이 권장된다.

---

## 2. 네트워크 프로그래밍

### Client-Server 모델

대부분의 네트워크 애플리케이션은 client-server 모델을 따른다:

```
Client ──request──→ Server
Client ←─response── Server
```

- **Server**: 자원을 관리하고, 클라이언트 요청에 대해 서비스 제공
- **Client**: 서버에 요청을 보내고 응답을 처리
- Client와 Server는 같은 호스트에서 실행될 수도 있다

### 네트워크 계층

```
Application (HTTP, FTP, SMTP)
    ↕
Transport (TCP, UDP)
    ↕
Network (IP)
    ↕
Link (Ethernet)
    ↕
Physical
```

### IP 주소

- IPv4: 32-bit 주소 (예: `128.2.194.242`)
- Network byte order: **Big Endian**으로 저장

```c
#include <arpa/inet.h>

// host to network (변환)
uint32_t htonl(uint32_t hostlong);    // 32-bit
uint16_t htons(uint16_t hostshort);   // 16-bit

// network to host (역변환)
uint32_t ntohl(uint32_t netlong);
uint16_t ntohs(uint16_t netshort);
```

### DNS (Domain Name System)

호스트 이름 → IP 주소 변환:

```c
#include <netdb.h>

struct addrinfo *result;
getaddrinfo("www.example.com", "80", &hints, &result);
// 호스트 이름과 서비스를 주소 정보로 변환
freeaddrinfo(result);
```

### 소켓 (Socket)

네트워크 통신의 endpoint. 커널 입장에서는 **파일**과 동일하게 취급.

소켓 주소 = **IP 주소 + 포트 번호**

```c
struct sockaddr_in {
    sa_family_t    sin_family;   // AF_INET
    in_port_t      sin_port;     // 포트 번호 (network byte order)
    struct in_addr sin_addr;     // IP 주소
};
```

### TCP 연결 과정

**Server 측:**
```c
int listenfd = socket(AF_INET, SOCK_STREAM, 0);  // 1. 소켓 생성
bind(listenfd, &addr, sizeof(addr));               // 2. 주소 바인딩
listen(listenfd, BACKLOG);                         // 3. 연결 대기
int connfd = accept(listenfd, &clientaddr, &len);  // 4. 연결 수락
// read/write로 통신
close(connfd);                                     // 5. 연결 종료
```

**Client 측:**
```c
int clientfd = socket(AF_INET, SOCK_STREAM, 0);   // 1. 소켓 생성
connect(clientfd, &serveraddr, sizeof(serveraddr)); // 2. 서버에 연결
// read/write로 통신
close(clientfd);                                    // 3. 연결 종료
```

### HTTP (HyperText Transfer Protocol)

웹 서버와 클라이언트 간 통신 프로토콜:

**요청 (Request):**
```
GET /index.html HTTP/1.1
Host: www.example.com
```

**응답 (Response):**
```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1234

<html>...</html>
```

주요 상태 코드:

| 코드 | 의미 |
|------|------|
| 200 | OK |
| 301 | Moved Permanently |
| 403 | Forbidden |
| 404 | Not Found |
| 500 | Internal Server Error |

**Static vs Dynamic Content:**
- **Static**: 파일을 그대로 반환 (HTML, 이미지 등)
- **Dynamic**: 프로그램 실행 결과를 반환 (CGI 등)

---

## 3. 동시성 프로그래밍 (Concurrent Programming)

### 왜 동시성이 필요한가?

Iterative server(순차 서버)의 문제:
- 한 번에 하나의 클라이언트만 처리
- 한 클라이언트가 느리면 다른 클라이언트들이 모두 대기

→ **Concurrent server**로 여러 클라이언트를 동시에 처리

### 동시성 구현 방법

#### 1. 프로세스 기반 (fork)

```c
while (1) {
    connfd = accept(listenfd, ...);
    if (fork() == 0) {
        close(listenfd);      // child: listen socket 닫기
        handle_client(connfd); // 클라이언트 처리
        close(connfd);
        exit(0);
    }
    close(connfd);            // parent: connection socket 닫기
}
```

- 각 클라이언트를 별도 프로세스에서 처리
- **장점**: 완전한 격리 (별도 address space)
- **단점**: `fork()` 오버헤드, 프로세스 간 데이터 공유 어려움

#### 2. 스레드 기반 (pthread)

```c
while (1) {
    connfd = accept(listenfd, ...);
    pthread_create(&tid, NULL, thread_func, &connfd);
}

void *thread_func(void *arg) {
    int connfd = *((int *)arg);
    pthread_detach(pthread_self());
    handle_client(connfd);
    close(connfd);
    return NULL;
}
```

- 같은 프로세스 내에서 여러 실행 흐름
- **장점**: 가벼움, 데이터 공유 용이
- **단점**: 동기화 필요, 버그 찾기 어려움

#### 3. I/O 멀티플렉싱 (select/epoll)

```c
fd_set read_set;
while (1) {
    FD_ZERO(&read_set);
    FD_SET(listenfd, &read_set);
    // 클라이언트 fd들도 추가
    select(maxfd + 1, &read_set, NULL, NULL, NULL);
    // 준비된 fd들을 확인하고 처리
}
```

- 하나의 프로세스/스레드에서 여러 I/O를 감시
- **장점**: 오버헤드 최소, 완전한 제어
- **단점**: 프로그래밍 복잡

---

## 4. 스레드 (Threads)

### 개념

스레드는 프로세스 내의 **독립적인 실행 흐름**이다.

각 스레드가 **독립적으로** 가지는 것:
- Thread ID
- Stack (별도의 스택 공간)
- Stack pointer, PC, 범용 레지스터, condition codes

스레드들이 **공유**하는 것:
- Code, Data, Heap
- 열린 파일 (file descriptors)
- 설치된 signal handlers

### Posix Threads (Pthreads)

```c
#include <pthread.h>

// 스레드 생성
int pthread_create(pthread_t *tid, pthread_attr_t *attr,
                   void *(*start_routine)(void *), void *arg);

// 스레드 종료 대기
int pthread_join(pthread_t tid, void **retval);

// 스레드 분리 (자동 자원 회수)
int pthread_detach(pthread_t tid);

// 자신의 Thread ID
pthread_t pthread_self(void);

// 스레드 종료
void pthread_exit(void *retval);
```

### Detached vs Joinable

- **Joinable** (기본): 다른 스레드가 `pthread_join()`으로 종료를 대기하고 자원 회수
- **Detached**: 종료 시 자동으로 자원 회수, join 불가

> 서버에서는 보통 `pthread_detach()`를 사용 — join할 필요 없으므로.

---

## 5. 동기화 (Synchronization)

### 왜 동기화가 필요한가?

여러 스레드가 **공유 변수**에 동시에 접근하면 **race condition** 발생:

```c
// 두 스레드가 동시에 실행
volatile int cnt = 0;

void *thread(void *arg) {
    for (int i = 0; i < 1000000; i++)
        cnt++;    // NOT atomic! (load → add → store)
    return NULL;
}
// 결과: cnt < 2000000 (예측 불가)
```

`cnt++`는 실제로 3개의 명령어:
1. `load cnt` → register
2. `add 1` to register
3. `store register` → cnt

두 스레드가 이 사이에 끼어들면 갱신이 손실된다.

### 공유 변수 분석

| 변수 종류 | 공유? |
|-----------|-------|
| Global variable | O (모든 스레드 접근 가능) |
| Local static variable | O (하나의 인스턴스) |
| Local variable | X (각 스레드 스택에 별도 존재) |

> 정의: 변수 x가 **공유**되려면, 여러 스레드가 x의 어떤 인스턴스를 참조해야 한다.

### Mutex (상호 배제)

**Critical section**을 보호하기 위한 잠금 메커니즘:

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

pthread_mutex_lock(&mutex);
// === Critical Section ===
// 한 번에 하나의 스레드만 실행
cnt++;
// ========================
pthread_mutex_unlock(&mutex);
```

### Semaphore

Dijkstra가 제안한 동기화 도구:

```c
#include <semaphore.h>

sem_t sem;
sem_init(&sem, 0, initial_value);   // 초기값 설정

sem_wait(&sem);    // P 연산: 값이 0이면 대기, 양수면 1 감소 후 진행
// Critical Section
sem_post(&sem);    // V 연산: 값을 1 증가, 대기 중인 스레드 깨움
```

**Binary semaphore** (초기값 1) = mutex와 동일한 역할

**Counting semaphore** (초기값 n) = 동시에 n개의 스레드까지 허용

### 고전적 동시성 문제

#### Producer-Consumer

```
Producer ──→ [Buffer (크기 n)] ──→ Consumer
```

- Producer: 데이터를 생성하여 버퍼에 넣음
- Consumer: 버퍼에서 데이터를 꺼내 처리
- 동기화: 버퍼가 가득 차면 producer 대기, 비면 consumer 대기

#### Readers-Writers

- **Reader**: 공유 데이터를 읽기만 함 (동시 읽기 허용)
- **Writer**: 공유 데이터를 수정함 (배타적 접근 필요)
- 규칙: writer가 쓰는 동안에는 다른 reader/writer 접근 불가

### 주의할 점

| 문제 | 설명 |
|------|------|
| **Race condition** | 실행 순서에 따라 결과가 달라짐 |
| **Deadlock** | 서로 상대의 자원을 기다리며 영원히 대기 |
| **Livelock** | 계속 시도하지만 진전 없음 |
| **Starvation** | 특정 스레드가 영원히 실행 기회를 얻지 못함 |

**Deadlock 방지 규칙**: 모든 스레드가 같은 순서로 lock을 획득하면 deadlock 회피 가능.

---

## 정리

| 주제 | 핵심 |
|------|------|
| Unix I/O | 모든 것은 파일, fd 기반, read/write + RIO로 안전한 I/O |
| Socket | IP + Port, TCP 연결: socket → bind → listen → accept |
| HTTP | 요청/응답 프로토콜, Static/Dynamic content |
| 프로세스 기반 동시성 | fork()로 격리, 오버헤드 큼 |
| 스레드 기반 동시성 | pthread, 가벼움, 공유 메모리 → 동기화 필수 |
| Mutex/Semaphore | critical section 보호, race condition 방지 |
| Deadlock | 교착 상태 — lock 순서 통일로 방지 |

---

*이전 글: [[CS230] 5. 가상 메모리와 동적 할당](/posts/cs230-5-vm-malloc/)*

---

> 이 시리즈는 KAIST CS230 시스템프로그래밍 수업의 개인 노트를 정리한 것입니다.
> 교재와 강의 슬라이드의 내용을 바탕으로 작성되었습니다.
