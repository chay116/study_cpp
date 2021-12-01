# 4장 TCP 기반 서버/클라이언트 1

## 04-1 TCP와 UDP에 대한 이해

### TCP/IP 프로토콜 스택

![TCP/IP Protocol Stack](<../.gitbook/assets/TCPIP protocol stack.png>)

위와 같이 TCP/IP 스택이 총 4개의 계층으로 나눠지는 것을 통해, 데이터 송수신의 과정을 4개의 영역으로 계층화하여 세분화될 수 있다는 것을 알 수 있다. 각각의 계층을 담당하는 것은 운영체제와 같은 소프트웨어이기도 하고 NIC와 같은 물리적인 장치이기도 함.

### TCP/IP 프로토콜의 탄생배경

'인터넷을 통한 효율적인 데이터의 송수신'을 위해 문제를 영역별로 나누어서 해결하다보니 프로토콜이 여러개가 만들어졌다. 이들을 계층 구조를 통해 상호간에 관계를 맺게 되었으며 이렇게 계층화해서 얻게 되는 장점은 **표준화 작업을 통한 개방형 시스템의 설계**이다.

#### LINK 계층

LINK 계층은 **물리적인 영역의 표준화**에 대한 결과이며 이는 가장 기본이 되는 영역으로 LAN, WAN, MAN과 같은 네트워크 표준과 관련된 프로토콜을 정의하는 영역이다.

#### IP 계층

물리적 연결을 기반으로 **목적지로 데이터를 전송하기 위해서 중간에 어떠한 경로를 거칠지와 관련한 문제를 해결하는 계층**에 해당한다. 이 계층에서 사용하는 프로토콜이 IP(Internet Protocol)이다. 이 IP 자체는 비 연결지향적이며 신뢰할 수 없는 프로토콜이라 오류발생에 대한 대비가 되어있지 않다고 말한다.

#### TCP/UDP 계층

IP 계층에서 알려준 경로정보를 바탕으로 **데이터의 실제 송수신을 담당**한다. 여기서 TCP의 역할에 대해서 좀 더 설명하면 데이터를 주고받는 과정에서 서로 데이터의 주고 받음을 확인하고, 분실된 데이터에 대해서 재전송해주어 데이터 전송을 신뢰하도록 만들어준다.

#### APPLICATION 계층

위에서 설명한 내용은 소켓을 만들면 데이터 송수신과정에서 자동으로 처리되는 것이다. 최종적으로 소켓이라는 도구가 프로그래머에게 주어졌으며, 이 도구를 통해 무엇인가를 만드는 과정에서 프로그램의 성격에 따라 클라이언트와 서버간의 데이터 송수신에 대한 약속들이 정해지는데, 이를 APPLICATION 프로토콜이라고 한다.

***

## 04-2 TCP기반 서버, 클라이언트 구현

### TCP 서버에서의 기본적인 함수호출 순서

![](<../.gitbook/assets/function call order in TCP server.png>)

#### 연결요청 대기 상태로의 진입

listen 함수 호출을 통해 연결 요청 대기 상태로 진입한다.

```cpp
#include <sys/socket.h>
int listen(int sockfd, int backlog);
// 성공시 0, 실패 시 -1 반환
// sockfd : 소켓의 fd 전달
// backlog : 연결 요청 대기열의 크기 설정, 연결 요청 자체를 대기시킬 수 있는 상태에 있다는 의미.
```

#### 클라이언트의 연결요청 수락

들어온 순서대로 연결요청을 수락해야 한다. accept 함수는 연결요청 대기 큐에서 대기중인 클라이언트의 연결요청을 수락하는 기능의 함수이다. 함수 호출 성공 시 내부적으로 데이터 입출력에 사용할 소켓을 생성하고, 그 소켓의 파일 디스크립터를 반환한다.

```cpp
#include <sys/socket.h>
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
// 성공시 파일 디스크립터, 실패 시 -1 반환
// sockfd : 생성한 소켓의 fd 전달
// addr : 연결 요청한 클라이언트의 주소 정보
// addrlen : addr의 크기
```

`hello_server.c`

> 위에서 하나하나 살펴보면 우선 소켓을 생성한뒤 구조체 변수를 초기화한 후 bind함수를 호출한다. 그리고 연결 요청 대기상태로 들어가기 위해 listen함수를 호출하고, 연결요청 대기 큐의 크기를 5로 설정하고있다. 그 이후에 accept함수가 호출된다. 마지막으로 write함수 호출을 통해서 클라이언트에게 데이터를 전송하고 마지막으로 close함수호출을 통해 연결을 끊고 있다.

### TCP 클라이언트의 기본적인 함수호출 순서

![](../.gitbook/assets/socket\_order.png)

클라이언트가 서버에 연결요청을 하기 위해서는 connect 함수가 필요하다. 그 후, 서버에 의해 연결요청이 접수되거나, 네트워크 단절 등 오류상황이 발생해서 연결요청이 중단되는 경우 함수가 반환된다.

```cpp
#include <sys/socket.h>
int connect(int sockfd, struct sockaddr *serv_addr, socklen_t addrlen);
// 성공 시 0, 실패 시 -1 반환
// sockfd : 생성한 소켓의 fd 전달
// serv_addr : 서버 주소 정보
// addrlen : myaddr의 크기
```

`hello_client.c`

> 클라이언트 예제에서는 socket을 생성한 후 구조체 정보를 초기화한 후에 connect함수를 호출한다. 여기서 connect는 서버의 accept함수 호출이 아닌 대기 큐에 등록된 상황을 의미하는 것이므로 connect함수가 반환했더라고 당장에 서비스가 이루어지는 것은 아니다. 또한 서버의 bind와 같이 IP와 PORT가 커널에 의해 자동적으로 할당된다. 그래서 bind함수를 명시적으로 호출할 필요가 없다. 마지막으로 데이터를 송수신 한후에 close를 통해 소켓을 닫아준다.

### TCP기반 서버, 클라이언트의 함수호출 관계

![](../.gitbook/assets/server\_client.png)

***

## 04-3 Iterative 기반의 서버, 클라이언트 구현

에코 서버와 에코 클라이언트를 여기서 구현할 것이며, 에코 서버는 클라이언트가 전송하는 문자열 데이터를 그대로 재전송하는 것이다.

`echo_server.c`

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#define BUF_SIZE 1024
void error_handling(char* message);

int main(int argc, char* argv[])
{
    int serv_sock, clnt_sock;
    char message[BUF_SIZE];
    int str_len, i;

    struct sockaddr_in serv_adr, clnt_adr;
    socklen_t clnt_adr_sz;

    if(argc!=2)
    {
        printf("Usage : %s <port>\n", argv[0]);
        exit(1);
    }

    serv_sock=socket(PF_INET, SOCK_STREAM, 0);
    if(serv_sock == -1)
    {
        error_handling("socket() error!");
    }

    memset(&serv_adr, 0, sizeof(serv_adr));
    serv_adr.sin_family = AF_INET;
    serv_adr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_adr.sin_port = htons(atoi(argv[1]));

    if(bind(serv_sock, (struct sockaddr*)&serv_adr, sizeof(serv_adr)) == -1)
        error_handling("bind() error!");

    if(listen(serv_sock, 5) == -1)
        error_handling("listen() error!");

    clnt_adr_sz = sizeof(clnt_adr);

    for(int i = 0; i<5; i++)
    {
        clnt_sock = accept(serv_sock, (struct sockaddr*)&clnt_adr, &clnt_adr_sz);
        if(clnt_sock == -1)
            error_handling("accept() error!");
        else
            printf("Connected client %d \n", i+1);

        while((str_len = read(clnt_sock, message, BUF_SIZE))!= 0)
            write(clnt_sock, message, str_len);

        close(clnt_sock);

    }
    close(serv_sock);
    return 0;
}

void error_handling(char * message)
{
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}
```

thread를 활용하지 않는 한, 하나의 client에 대해서 대처밖에 하지 못한다. 이와 같이 반복문을 통해 accept를 지속적으로 발동시키는 함수의 형태를 iterative 함수라고 한다.

`echo_client.c`

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#define BUF_SIZE 1024
void error_handling(char * message);

int main(int argc, char * argv[])
{
    int sock;
    char message[BUF_SIZE];
    int str_len;
    struct sockaddr_in serv_adr;

    if(argc!=3)
    {
        printf("Usage: %s <IP> <port> \n", argv[0]);
        exit(1);
    }

    sock = socket(PF_INET, SOCK_STREAM, 0);
    if(sock == -1)
        error_handling("socket() error!");

    memset(&serv_adr, 0, sizeof(serv_adr));
    serv_adr.sin_family = AF_INET;
    serv_adr.sin_addr.s_addr = inet_addr(argv[1]);
    serv_adr.sin_port = htons(atoi(argv[2]));

    if(connect(sock, (struct sockaddr*)&serv_adr, sizeof(serv_adr)) == -1)
        error_handling("connect() error!");
    else
        puts("Connected...........");

    while(1)
    {
        fputs("Input message(Q to quit): ", stdout);
        fgets(message, BUF_SIZE, stdin);

        if(!strcmp(message, "q\n") || !strcmp(message,"Q\n"))
            break;

        write(sock,message, strlen(message));
        str_len = read(sock, message, BUF_SIZE-1);
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

> 위의 에코 서버, 클라이언트 예제는 1가지 문제점이 있다. 그것은 바로 read, write함수가 호출될 때마다 문자열 단위로 실제 입출력이 이뤄진다라는 가정을 하고 있다. TCP클라이언트는 둘 이상의 write 함수호출로 전달된 문자열의 정보가 묶여서 한번에 서버로 전송할 수 있다. 이러한 TCP의 특성을 생각한다면 문제가 존재하지만 위 예제는 오류없이 정상적으로 작동은 한다. (But 오류 발생 가능성 존재!)
