# 14장 멀티캐스트 & 브로드캐스트

## 14-1 멀티캐스트(Multicast)

#### **멀티캐스트**

먼티 캐스트 방식의 데이터 전송은 UDP를 기반으로 하는데, 하나의 목적지가 아닌 특정 그룹에 해당하는 다수의 호스트에게 데이터를 전송한다. 한번만 전송하더라도, 라우터들에 의해 복사되어 모두에게 전송된다.

![https://blog.kakaocdn.net/dn/ber3f5/btqZ9Bza6P2/0wHeAEKen1h4s1hKNMmm60/img.png](https://blog.kakaocdn.net/dn/ber3f5/btqZ9Bza6P2/0wHeAEKen1h4s1hKNMmm60/img.png)

#### 라우팅과 TTL, 그룹 가입 방법

**TTL**

**TTL**은 \*\*`'패킷을 얼마나 멀리 전달할 것인가'`\*\*를 결정하는 주 요소가 된다. 라우터를 거칠때마다 1씩 감소되어 0이되면 해당 패킷이 소멸되며, 너무 크게 설정하면 네트워크 트래픽에 좋지 않은 영향을 줄 수 있다. TTL을 설정하는 법은 setsockopt 함수를 이용하여 구성할 수 있다.

```cpp
int send_sock;
int time_to_live = 64;

send_sock=socket(PF_INET, SOCK_DGRAM< 0);		// UDP 소켓
setsockpet(send_sock, IPPROTO_IP, IP_MULTICAST_TTL, (void*)&time_to_live, sizeof(time_to_live);
```

그룹 가입과 관련된 프로토콜의 레벨은 IPPROTO\_IP이고, 옵션의 이름은 IP\_ADD\_MEMBERSHIP이다. 구현은 다음과 같다.

```cpp
int main(int argc, char *argv[])
{
	int recv_sock;
	struct sockaddr_in adr;
	struct ip_mreq join_adr;

	recv_sock=socket(PF_INET, SOCK_DGRAM, 0);
	...;
	join_adr.imr_multiaddr.s_addr=inet_addr(argv[1]);		// 가입할 그룹 주소
	join_adr.imr_interface.s_addr=htonl(INADDR_ANY);		// 본인의 주소

	setsockopt(recv_sock, IPPROTO_IP, IP_ADD_MEMBERSHIP, (void*)&join_adr, sizeof(join_adr));

	while(1)
	{
		str_len=recvfrom(recv_sock, buf, BUF_SIZE-1, 0, NULL, 0);
		if(str_len<0) {	break; }
		buf[str_len]=0;
		fputs(buf, stdout);
	}
	close(recv_sock);
	return 0;
}
```

#### 멀티캐스트 Sender와 Receiver의 구현

Sender은 파일에 저장된 뉴스 정보를 AAA 그룹으로 broadcasting하고, receiver은 이 정보를 수신하는 것을 바탕으로 구현하였다.

`news_sender.c`

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define TTL 64
#define BUF_SIZE 30
void error_handling(char *message);

int main(int argc, char *argv[])
{
	int send_sock;
	struct sockaddr_in mul_adr;
	int time_live=TTL;
	FILE *fp;
	char buf[BUF_SIZE];

	if(argc!=3){
		printf("Usage : %s <GroupIP> <PORT>\n", argv[0]);
		exit(1);
	}
  	
	send_sock=socket(PF_INET, SOCK_DGRAM, 0);
	memset(&mul_adr, 0, sizeof(mul_adr));
	mul_adr.sin_family=AF_INET;
	mul_adr.sin_addr.s_addr=inet_addr(argv[1]);  // Multicast IP
	mul_adr.sin_port=htons(atoi(argv[2]));       // Multicast Port
	
	setsockopt(send_sock, IPPROTO_IP, 
		IP_MULTICAST_TTL, (void*)&time_live, sizeof(time_live));
	
	if((fp=fopen("news.txt", "r"))==NULL)
		error_handling("fopen() error");

	while(!feof(fp))   /* Broadcasting */
	{
		fgets(buf, BUF_SIZE, fp);
		sendto(send_sock, buf, strlen(buf), 
			0, (struct sockaddr*)&mul_adr, sizeof(mul_adr));
		sleep(2);
	}
	fclose(fp);
	close(send_sock);
	return 0;
}

void error_handling(char *message)
{
	fputs(message, stderr);
	fputc('\n', stderr);
	exit(1);
}
```

> Sender는 일반 UDP 소켓 프로그램과 차이가 크지 않다. UDP 소켓을 생성하고, IP주소를 멀티캐스트 주소로 설정한 뒤 TTL 정보를 지정하여 sendto 함수를 통해 전송한다.

`news_receiver.c`

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 30
void error_handling(char *message);

int main(int argc, char *argv[])
{
	int recv_sock;
	int str_len;
	char buf[BUF_SIZE];
	struct sockaddr_in adr;
	struct ip_mreq join_adr;
	
	if(argc!=3) {
		printf("Usage : %s <GroupIP> <PORT>\n", argv[0]);
		exit(1);
	 }
  
	recv_sock=socket(PF_INET, SOCK_DGRAM, 0);
 	memset(&adr, 0, sizeof(adr));
	adr.sin_family=AF_INET;
	adr.sin_addr.s_addr=htonl(INADDR_ANY);	
	adr.sin_port=htons(atoi(argv[2]));
	
	if(bind(recv_sock, (struct sockaddr*) &adr, sizeof(adr))==-1)
		error_handling("bind() error");
	
	join_adr.imr_multiaddr.s_addr=inet_addr(argv[1]);
	join_adr.imr_interface.s_addr=htonl(INADDR_ANY);
  	
	setsockopt(recv_sock, IPPROTO_IP, 
		IP_ADD_MEMBERSHIP, (void*)&join_adr, sizeof(join_adr));
  
	while(1)
	{
		str_len=recvfrom(recv_sock, buf, BUF_SIZE-1, 0, NULL, 0);
		if(str_len<0) 
			break;
		buf[str_len]=0;
		fputs(buf, stdout);
	}
	close(recv_sock);
	return 0;
}

void error_handling(char *message)
{
	fputs(message, stderr);
	fputc('\n', stderr);
	exit(1);
}
```

> ip\_mreg형 변수을 머티캐스트 그룹 IP주소와 그룹에 가입할 호스트의 IP주소를 초기화한 후, IP\_ADD\_MEMBERSHIP을 이용해서 멀티캐스트 그룹에 가입한다. 그 후 recvfrom 함수호출을 통해 멀티캐스트 데이터를 수신한다.

***

## 14-2 브로드캐스트(Broadcast)

### **브로드캐스트**

브로드캐스트는 `동일한 네트워크 내에 존재하는 호스트에게 데이터를 전송하기 위한 방법` 이다. 데이터를 전송하는 대상이 호스트가 아닌 네트워크 이기 때문에 네트워크에 연결된 호스트에 모두 전달된다. 데이터 전송 시 사용되는 IP주소의 형태에 따라서 다음과 같이 두 가지 형태로 구분이 된다.

* Directed 브로드캐스트
  * 특정 지역의 네트워크에 연결된 모든 호스트에게 데이터를 전송하기 위한 수단.
  * 192.34.24인 네트워크의 모든 호스트에게 데이터를 전송하려면 192.12.34.255로 전송하면 됨.
* Local 브로드캐스트
  * Local 브로드캐스트를 위해서는 255.255.255.255라는 IP주소가 특별히 예약되어 있다.
  * 네트워크 주소가 192.32.24인 네트워크에 연결되어 있는 호스트가 IP주소 255.255.255.255로 전달하면 192.32.24로 시작하는 IP주소의 모든 호스트에게 데이터 전달

```cpp
int send_sock;
int bcast = 1;		// 브로드캐스트 ON

send_sock=socket(PF_INET, SOCK_DGRAM, 0);
setsockopt(send_sock, SOL_SOCKET, SO_BROADCAST, (void*)&bcast, sizeof(bcast));
```

#### 브로드캐스트 기반의 Sender와 Receiver의 구현

`news_sender_brd.c`

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 30
void error_handling(char *message);

int main(int argc, char *argv[])
{
	int send_sock;
	struct sockaddr_in broad_adr;
	FILE *fp;
	char buf[BUF_SIZE];
	int so_brd=1;
	
	if(argc!=3) {
		printf("Usage : %s <Boradcast IP> <PORT>\n", argv[0]);
		exit(1);
	}
  
	send_sock=socket(PF_INET, SOCK_DGRAM, 0);	
	memset(&broad_adr, 0, sizeof(broad_adr));
	broad_adr.sin_family=AF_INET;
	broad_adr.sin_addr.s_addr=inet_addr(argv[1]);
	broad_adr.sin_port=htons(atoi(argv[2]));
	
	setsockopt(send_sock, SOL_SOCKET, 
		SO_BROADCAST, (void*)&so_brd, sizeof(so_brd));
     // UDP 소켓을 브로드캐스트 기반 데이터 전송을 하도록 옵션 정보 변경
	if((fp=fopen("news.txt", "r"))==NULL)
		error_handling("fopen() error");

	while(!feof(fp))
	{
		fgets(buf, BUF_SIZE, fp);
		sendto(send_sock, buf, strlen(buf), 
			0, (struct sockaddr*)&broad_adr, sizeof(broad_adr));
		sleep(2);
	}

	close(send_sock);
	return 0;
}

void error_handling(char *message)
{
	fputs(message, stderr);
	fputc('\n', stderr);
	exit(1);
}
```

`news_receiver_brd.c`

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 30
void error_handling(char *message);

int main(int argc, char *argv[])
{
	int recv_sock;
	struct sockaddr_in adr;
	int str_len;
	char buf[BUF_SIZE];
	
	if(argc!=2) {
		printf("Usage : %s  <PORT>\n", argv[0]);
		exit(1);
	 }
  
	recv_sock=socket(PF_INET, SOCK_DGRAM, 0);
	
	memset(&adr, 0, sizeof(adr));
	adr.sin_family=AF_INET;
	adr.sin_addr.s_addr=htonl(INADDR_ANY);	
	adr.sin_port=htons(atoi(argv[1]));
	
	if(bind(recv_sock, (struct sockaddr*)&adr, sizeof(adr))==-1)
		error_handling("bind() error");
  
	while(1)
	{
		str_len=recvfrom(recv_sock, buf, BUF_SIZE-1, 0, NULL, 0);
		if(str_len<0) 
			break;
		buf[str_len]=0;
		fputs(buf, stdout);
	}
	
	close(recv_sock);
	return 0;
}

void error_handling(char *message)
{
	fputs(message, stderr);
	fputc('\n', stderr);
	exit(1);
}
```
