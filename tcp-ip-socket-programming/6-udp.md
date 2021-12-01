# 6장 UDP 기반 서버 클라이언트

## 06-1 UDP에 대한 이해

### UDP 소켓의 특성

UDP 소켓은 신뢰할 수 없는 전송방법을 제공하지만 TCP와 달리 ACK, SEQ와 같은 일들을 하지 않고 간결한 구조로 설계되어 있기 때문에 상황에 따라서 TCP 보다 훨씬 좋은 성능을 발휘한다.TCP와는 달리 IP를 기반으로 신뢰성 있는 데이터의 송수신을 위한 흐름제어가 존재하지 않는다. 즉 연결의 설정 및 해제 과정과 같이 연결 제어가 존재하지 않는다.

### UDP의 내부 동작원리

![https://media.vlpt.us/images/hustle-dev/post/21e1bebd-0588-4f41-bcf9-5d0957723fdb/image.png](https://media.vlpt.us/images/hustle-dev/post/21e1bebd-0588-4f41-bcf9-5d0957723fdb/image.png)

UDP의 역할 중 가장 중요한 것은 호스트로 수신된 패킷을 PORT정보를 탐조하여 최종 목적지인 UDP소켓에 전달하는 것이다.

### UDP의 효율적 사용

주로 멀티미디어 데이터의 경우 UDP를 사용하여 송수신 하는 것이 훨씬 효율적이고 빠르게 동작한다.

***

## 06-2 UDP 기반 서버/클라이언트의 구현

#### UDP에서의 서버와 클라이언트는 연결되어 있지 않다.

TCP 서버 구현과정에서 거쳤던 listen 함수와 accept함수의 호출은 불필요하다.

#### UDP에서는 서버건 클라이언트건 하나의 소켓만 있으면 된다.

![https://media.vlpt.us/images/hustle-dev/post/30d962a6-c256-4f63-a423-72c39cc71278/image.png](https://media.vlpt.us/images/hustle-dev/post/30d962a6-c256-4f63-a423-72c39cc71278/image.png)

우체통과 같이 어디든 보낼 소켓이 하나만 존재하면 되기에 UDP 소켓은 하나만 있으면 둘 이상의 호스트와의 통신이 가능하다.

### UDP 기반의 데이터 입출력 함수

TCP 소켓은 목적지에 해당하는 소켓과 연결된 사앹이기 때문에 주소정보를 미리 알고 있지만, UDP 소켓은 연결상태를 유지하지 않으므로 데이터를 전송할 때마다 반드시 목적지의 주소 정보를 별도로 추가해야한다.

```cpp
#include <sys/socket.h>
ssize_t sendto(int sock, void* buff, size_t nbytes, int flags,
	struct sockaddr *to, socklen_t addrlen); // 전송 함수
// 성공 시 전송된 바이트 수, 실패시 -1 반환
// to : 목적지 주소를 담고 있는 구조체 변수

ssize_t recvfrom(int sock, void *buff, size_t nbytes, int flags,
	struct sockaddr* from, socklen_t * addrlen); // 수신 함수
// 성공 시 전송된 바이트 수, 실패시 -1 반환
// from : 발신지 주소를 담고 있는 구조체 변수
```

#### UDP 기반의 에코 서버와 에코 클라이언트

`uecho_server.c`

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#define BUF_SIZE 30
void error_handling(char * message);

int main(int argc, char* argv[])
{
    int serv_sock;
    char message[BUF_SIZE];
    int str_len;
    socklen_t clnt_adr_sz;

    struct sockaddr_in serv_adr, clnt_adr;
    if(argc!=2)
    {
        printf("Usage: %s <port> \n", argv[0]);
        exit(1);
    }

    serv_sock = socket(PF_INET, SOCK_DGRAM, 0);
    if(serv_sock == -1)
        error_handling("UDB socket creation error");

    memset(&serv_adr, 0 ,sizeof(serv_adr));
    serv_adr.sin_family = AF_INET;
    serv_adr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_adr.sin_port = htons(atoi(argv[1]));

    if(bind(serv_sock, (struct sockaddr*)&serv_adr, sizeof(serv_adr)) == -1)
        error_handling("bind() error");

    while(1)
    {
        clnt_adr_sz = sizeof(clnt_adr);
        str_len = recvfrom(serv_sock, message, BUF_SIZE, 0,
        (struct sockaddr*)&clnt_adr, &clnt_adr_sz);
        sendto(serv_sock, message, str_len, 0,
        (struct sockaddr*)&clnt_adr, clnt_adr_sz);
    }
    close(serv_sock);
    return ;
}

void error_handling(char * message)
{
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}
```

`uecho_client.c`

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#define BUF_SIZE 30
void error_handling(char*message);

int main(int argc, char* argv[])
{
    int sock;
    char message[BUF_SIZE];
    int str_len;
    socklen_t adr_sz;

    struct sockaddr_in serv_adr, from_adr;
    if(argc!=3)
    {
        printf("Usage: %s <IP> <port> \n", argv[0]);
        exit(1);
    }

    sock = socket(PF_INET, SOCK_DGRAM, 0);
    if(sock == -1)
        error_handling("socket() error");

    memset(&serv_adr, 0, sizeof(serv_adr));
    serv_adr.sin_family = AF_INET;
    serv_adr.sin_addr.s_addr = inet_addr(argv[1]);
    serv_adr.sin_port = htons(atoi(argv[2]));

    while(1)
    {
        fputs("Inset message(q to quit): ", stdout);
        fgets(message, sizeof(message), stdin);
        if(!strcmp(message, "q\n") || !strcmp(message, "Q\n"))
            break;

        sendto(sock, message, strlen(message), 0,
        (struct sockaddr*)&serv_adr, sizeof(serv_adr));
        adr_sz = sizeof(from_adr);
        str_len = recvfrom(sock, message, BUF_SIZE, 0,
        (struct sockaddr*)&from_adr, &adr_sz);
        message[str_len] = 0;
        printf("Message from server: %s", message);
    }
    close(sock);
    return 0;
}

void error_handling(char * message)
{
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}
```

### UDP 클라이언트 소켓의 주소정보 할당

UDP 에선 sendto 함수호출 이전에 해당 소켓에 주소 정보가 할당되어 있어야 한다. 따라서 sendto 함수 호출 이전에 bind 함수를 호추래서 주소 정보를 할당해야 한다. 그러나 만약에 sendto 함수호출 시까지 주소정보가 할당되지 않았다면, sendto 함수가 처음 호출되는 시점에 해당 소켓에 IP와 PORT번호가 자동으로 할당된다.

***

## 06-3 UDP의 데이터 송수신 특성과 UDP에서의 connect 함수호출

### 데이터의 경계가 존재하는 UDP소켓

TCP 기반에서 송수신하는 데이터에는 경계가 존재하지 않는다고 하였는데 이는 다음의 의미와 같다."데이터 송수신 과정에서 호출하는 입출력함수의 호출횟수는 큰 의미를 지니지 않는다."반대로 UDP는 데이터의 경계가 존재하는 프로토콜이므로 데이터 송수신 과정에서 호출하는 입출력함수의 호출 횟수가 큰 의미를 지닌다.

`bound_host1.c`

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#define BUF_SIZE 30
void error_handling(char * message);

int main(int argc, char* argv[])
{
    int sock;
    char message[BUF_SIZE];
    struct sockaddr_in my_adr, your_adr;
    socklen_t adr_sz;
    int str_len, i;

    if(argc!=2)
    {
        printf("Usage: %s <port> \n", argv[0]);
        exit(1);
    }

    sock = socket(PF_INET, SOCK_DGRAM, 0);
    if(sock == -1)
        error_handling("socket() error");

    memset(&my_adr, 0, sizeof(my_adr));
    my_adr.sin_family = AF_INET;
    my_adr.sin_addr.s_addr = htonl(INADDR_ANY);
    my_adr.sin_port = htons(atoi(argv[1]));

    if(bind(sock, (struct sockaddr*)&my_adr, sizeof(my_adr)) == -1)
        error_handling("bind() error");

    for(i = 0; i<3; i++)
    {
        sleep(5);
        adr_sz = sizeof(your_adr);
        str_len = recvfrom(sock, message, BUF_SIZE, 0,
        (struct sockaddr*)&your_adr, &adr_sz);

        printf("Message %d: %s \n", i+1, message);
    }
    close(sock);
    return 0;
}

void error_handling(char* message)
{
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}
```

`bound_host2.c`

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#define BUF_SIZE 30
void error_handling(char* message);

int main(int argc, char* argv[])
{
    int sock;
    char msg1[] = "Hi!";
    char msg2[] = "I'm another UDP host!";
    char msg3[] = "Nice to meet you";

    struct sockaddr_in your_adr;
    socklen_t your_adr_sz;
    if(argc!=3)
    {
        printf("Usage: %s <IP> <port> \n", argv[0]);
        exit(1);
    }

    sock = socket(PF_INET, SOCK_DGRAM, 0);
    if(sock == -1)
        error_handling("socket() error");

    memset(&your_adr, 0, sizeof(your_adr));
    your_adr.sin_family = AF_INET;
    your_adr.sin_addr.s_addr = inet_addr(argv[1]);
    your_adr.sin_port = htons(atoi(argv[2]));

    sendto(sock, msg1, sizeof(msg1), 0,
    (struct sockaddr*)&your_adr, sizeof(your_adr));
    sendto(sock, msg2, sizeof(msg2), 0,
    (struct sockaddr*)&your_adr, sizeof(your_adr));
    sendto(sock, msg3, sizeof(msg3), 0,
    (struct sockaddr*)&your_adr, sizeof(your_adr));
    close(sock);
    return 0;
}

void error_handling(char *message)
{
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}
```

> bound\_host2는 총 3회의 sendto 함수호출을 통해서 데이터를 전송하고, bound\_host1.c는 총 3회의 recvfrom 함수호출을 통해서 데이터를 수신한다. 그런데 recvfrom 함수호출에는 5초간의 지연시간이 존재하기 때문에, recvfrom 함수가 호출되기 이전에 3회의 sendto함수호출이 진행되어서 데이터는 이미 bound\_hos1.c에 전송된 상태에 놓기된다. TCP라면 이 상황에서 단 한번의 입력함수 호출을 통해서 모든 데이터를 읽어들일 수 있다. 그러나 UDP는 3회 호출이 요구된다.

### connected UDP 소켓, unconnected UDP 소켓

sendto 함수호출을 통한 데이터의 전송과정은 다음과 같이 크게 세 단계로 나눌 수 있다.

* 1단계 UDP 소켓에 목적지의 IP와 PORT번호 등록
* 2단계 데이터 전송
* 3단계 UDP 소켓에 등록된 목적지 정보 삭제

즉 이렇게 목적지 정보가 등록되어 있지 않은 소켓을 'unconnected 소켓'이라 하고, 목적지 정보가 등록되어 있는 소켓을 'connected 소켓'이라고 한다.이 경우 데이터 전송을 할 때에 위의 과정을 3회 반복해야되는데 이것은 비효율적이며 connected소켓을 만들어 효율적으로 작업할 수 있다.

### connected UDP 소켓 생성

UDP 소켓을 대상으로 connect 함수만 호출해주면 된다.`uecho_con_client.c`

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <sys/socket.h>
#define BUF_SIZE 30
void error_handling(char* message);

int main(int argc, char* argv[])
{
    int sock;
    char message[BUF_SIZE];
    int str_len;
    socklen_t adr_sz; // connect에선 불필요!

    struct sockaddr_in serv_adr, from_adr; // connect에선 from_adr 불필요!
    if(argc!=3)
    {
        printf("Usage: %s <IP> <port> \n", argv[0]);
        exit(1);
    }

    sock = socket(PF_INET, SOCK_DGRAM, 0);
    if(sock == -1)
        error_handling("socket() error");

    memset(&serv_adr, 0 ,sizeof(serv_adr));
    serv_adr.sin_family = AF_INET;
    serv_adr.sin_addr.s_addr = inet_addr(argv[1]);
    serv_adr.sin_port = htons(atoi(argv[2]));

    connect(sock, (struct sockaddr*)&serv_adr, sizeof(serv_adr));

    while(1)
    {
        fputs("Insert message(q to quit): ", stdout);
        fgets(message, sizeof(message), stdin);
        if(!strcmp(message, "q\n") || !strcmp(message, "Q\n"))
            break;

        /*
        sendto(sock, message, strlen(message), 0,
        (struct sockaddr*)&serv_adr, sizeof(serv_adr));
        */
       write(sock, message, strlen(message));

       /*
       adr_sz = sizeof(from_adr);
       str_len = recvfrom(sock, message, BUF_SIZE, 0,
       (struct sockaddr*)&from_adr, &adr_sz);
       */
      str_len = read(sock, message, sizeof(message)-1);

      message[str_len] = 0;
      printf("Message from server: %s", message);
    }
    close(sock);
    return 0;
}

void error_handling(char*message)
{
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}
```
