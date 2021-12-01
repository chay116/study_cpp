# 17장 select보다 나은 epoll

## 17-1 epoll의 이해와 활용

#### **select 기반의 IO 멀티 플렉싱이 느린 이유**

select 함수는 소켓 관리하는 fd\_set을 직접 관리해야 하기에 fd\_set을 전달할 때 원본을 직접 전달하지 않고 복사본을 전달해야 하는 번거로움이 존재한다. select가 한번 일어날 때 마다 fd\_set을 모두 점검하면서 변화가 일어났는지 체크해야 하는데, 운영체제에 의해서 완성되는 기능이 아니고 함수에 의해 완성되기 때문에 번거롭고 성능이 떨어질 수 있다.

> 따라서, 운영체제에게 관찰대상에 대한 정보를 한번만 알려주고, 내용에 변경이 있을 때 변경 사항만 알려주자

라는 방식을 통해 이 번거러움을 해결해야한다. 이는 epoll을 통해 해소될 수 있다.

#### **select를 사용할 상황**

* 서버의 접속자가 많지 않음
* 다양한 운영체제에서 사용할 수 있어야 함 → epoll이나 iocp는 운영체제에 종속적임

#### **epoll 방식의 장점**

* select처럼 모든 소켓에 대해 반복문을 수행하면서 점검할 필요가 없음 → 변화가 일어난 소켓의 정보만 전달해서 주기 때문
* select함수에 대응하는 epoll\_wait 시 소켓의 정보를 매번 전달할 필요가 없음

#### **epoll에 필요한 함수와 구조체**

select 방식에서는 관찰대상인 fd의 저장을 위해 fd\_set형 변수를 직접 선언했지만, epoll 방식에서는 고나찰대상인 fd의 저장을 운영체제가 담당한다.

* epoll\_event, epoll\_data 구조체 : 소켓 디스크립터 등록, 이벤트 발생의 확인에 사용되는 구조체

```cpp
typedef union epoll_data
{
        void *ptr;
        int fd;                    // 이벤트가 일어난(or 일어날) 파일 디스크립터
        __unit32_t u32;
        __unit64_t u64;
} epoll_data_t;

struct epoll_event
{
        __unit32_t events;        // 관찰할 이벤트의 종류
        epoll_data_t data;
}
```

* `epoll_create` : epoll 파일 디스크립터 저장소 생성 → epoll의 시작을 운영체제에 알려주어 OS가 fd를 관리할 저장소를 만들게 함

```cpp
#include <sys/epoll.h>

int epoll_create(int size);
// 성공 시 epoll 파일 디스크립터, 실패 시 -1 반환
// size : epoll 인스턴스의 크기정보.
```

* `epoll_ctl` : 저장소에 파일 디스크립터를 등록, 삭제

```cpp
#include <sys/epoll.h>

int epoll_ctl(int epfd, int op, int fd, struct epoll_event* event);
// 성공 시 0, 실패 시 -1 반환
// epfd : epoll_create로 생성한 epoll 파일 디스크립터
// op : 관찰 대상의 추가, 삭제, 변경 여부
// fd : 등록할 파일디스크립터
```

`epoll_ctl(A, EPOLL_CTL_ADD, B, C);` epoll 인스턴스 A에 파일 디스크립터 B를 등록하되, C를 통해 전달된 이벤트의 관찰을 목적으로 등록을 진행한다.

`epoll_ctl(A, EPOLL_CTL_DEL, B, NULL);` epoll 인스턴스 A에 파일 디스크립터 B를 삭제한다.

```cpp
// epoll.h

#define EPOLL_CTL_ADD 1  // 파일 디스크립터를 epoll 인스턴스에 등록한다.
#define EPOLL_CTL_DEL 2  // 파일 디스크립터를 epoll 인스턴스에서 삭제한다.
#define EPOLL_CTL_MOD 3  // 등록된 파일 디스크립터의 이벤트 발생상황을 변경한다.
```

* EPOLLIN : 수신할 데이터가 존재하는 이벤트
* EPOLLOUT : 즉시 데이터를 전송할 수 있도록 출력 버퍼가 비워진 이벤트
* EPOLLPRI : OOB(Out-Of-Band) 데이터가 수신된 이벤트
* EPOLLRDHUP : 연결이 종료된 이벤트 (half-close 포함)
* EPOLLET : Edge-trigger 방식으로 이벤트 감지. 이 경우 | 연산자를 이용해 이벤트 종류도 함께 명시
* EPOLLONESHOT : 최초의 이벤트만 감지하고 이후에는 return하지 않는 방식. | 연산자를 이용해 이벤트 종류를 함께 명시
* EPOLLERR : 에러가 발생한 이벤트

```cpp
enum EPOLL_EVENTS
  {
    EPOLLIN = 0x001,
#define EPOLLIN EPOLLIN
    EPOLLPRI = 0x002,
#define EPOLLPRI EPOLLPRI
    EPOLLOUT = 0x004,
#define EPOLLOUT EPOLLOUT
    EPOLLRDNORM = 0x040,
#define EPOLLRDNORM EPOLLRDNORM
    EPOLLRDBAND = 0x080,
#define EPOLLRDBAND EPOLLRDBAND
    EPOLLWRNORM = 0x100,
#define EPOLLWRNORM EPOLLWRNORM
    EPOLLWRBAND = 0x200,
#define EPOLLWRBAND EPOLLWRBAND
    EPOLLMSG = 0x400,
#define EPOLLMSG EPOLLMSG
    EPOLLERR = 0x008,
#define EPOLLERR EPOLLERR
    EPOLLHUP = 0x010,
#define EPOLLHUP EPOLLHUP
    EPOLLRDHUP = 0x2000,
#define EPOLLRDHUP EPOLLRDHUP
    EPOLLONESHOT = (1 << 30),
#define EPOLLONESHOT EPOLLONESHOT
    EPOLLET = (1 << 31)
#define EPOLLET EPOLLET
  };
```

* epoll\_wait : select 함수처럼 파일의 디스크립터 변화를 대기

```cpp
#include <sys/epoll.h>

int epoll_wait(int epfd, struct epoll_event* events, int maxevents, int timeout);
// 성공 시 이벤트가 발생한 파일 디스크립터의 수, 실패 시 -1 반환
// epfd : epoll 파일 디스크립터
// events : 이벤트가 발생한 파일 디스크립터가 채워질 버퍼의 주소 값
// maxevent : 최대 이벤트 수
// timeout : 대기 시간, -1 전달 시 이벤트 발생 시까지 무한 대기
```

```cpp
	epfd=epoll_create(EPOLL_SIZE);
	ep_events=malloc(sizeof(struct epoll_event)*EPOLL_SIZE);

	event.events=EPOLLIN;
	event.data.fd=serv_sock;
	epoll_ctl(epfd, EPOLL_CTL_ADD, serv_sock, &event);

	while(1)
	{
		event_cnt=epoll_wait(epfd, ep_events, EPOLL_SIZE, -1);
        ...;
  	}
```

#### **epoll 기반의 에코 서버**

```cpp
int main(int argc, char *argv[])
{
	int serv_sock, clnt_sock;
	struct sockaddr_in serv_adr, clnt_adr;
	socklen_t adr_sz;
	int str_len, i;
	char buf[BUF_SIZE];

	struct epoll_event *ep_events;
	struct epoll_event event;
	int epfd, event_cnt;
	...;

	serv_sock=socket(PF_INET, SOCK_STREAM, 0);
	...;

	if(bind(serv_sock, (struct sockaddr*) &serv_adr, sizeof(serv_adr))==-1)
		printf("bind() error");
	if(listen(serv_sock, 5)==-1)
		printf("listen() error");

	epfd=epoll_create(EPOLL_SIZE);		// epoll 저장소 생성
	ep_events=malloc(sizeof(struct epoll_event)*EPOLL_SIZE);	// 이벤트 저장할 공간 생성

	event.events=EPOLLIN;
	event.data.fd=serv_sock;
	epoll_ctl(epfd, EPOLL_CTL_ADD, serv_sock, &event);		// 서버 소켓의 이벤트 등록

	while(1)
	{
		event_cnt=epoll_wait(epfd, ep_events, EPOLL_SIZE, -1);		// 이벤트 발생 대기
		if(event_cnt==-1)
		{
			puts("epoll_wait() error");
			break;
		}

		for(i=0; i<event_cnt; i++)
		{
			if(ep_events[i].data.fd==serv_sock)
			{
				adr_sz=sizeof(clnt_adr);
				clnt_sock = accept(serv_sock, (struct sockaddr*)&clnt_adr, &adr_sz);
				event.events=EPOLLIN;
				event.data.fd=clnt_sock;
				epoll_ctl(epfd, EPOLL_CTL_ADD, clnt_sock, &event);
                // 새로운 소켓도 등록
				printf("connected client: %d \n", clnt_sock);
			} // 발생한 소켓이 서버 소켓이면
			else
			{
					str_len=read(ep_events[i].data.fd, buf, BUF_SIZE);
					if(str_len==0)    // close request!
					{
						epoll_ctl(epfd, EPOLL_CTL_DEL, ep_events[i].data.fd, NULL);
                        // 소켓을 epoll 관리 대상에서 삭제
						close(ep_events[i].data.fd);		// 소켓 삭제
						printf("closed client: %d \n", ep_events[i].data.fd);
					}
					else
					{
						write(ep_events[i].data.fd, buf, str_len);    // echo!
					}

			}
		}
	}
	close(serv_sock);		// 서버 소켓 소멸
	close(epfd);			// epoll 인스턴스 소멸
	return 0;
}
```

***

## 17-2 레벨 트리거와 엣지 트리거

#### **레벨 트리거와 엣지 트리거의 차이**

* 레벨 트리거 : 소켓 버퍼에 데이터가 남아있는 동안에 계속 이벤트 발생
* 엣지 트리거 : 소켓 버퍼에 데이터가 들어오는 순간에만 이벤트를 발생

서버쪽에서 컨트롤 요소가 많고 데이터 송수신이 빈번한 경우는 엣지 트리거가 유리하고 단순하고 데이터 송수신 상황이 다양하지 않으면 레벨 트리거 방식이 유리하다. 기본적으론 레벨 트리거로 설정이 되어 있기에 epoll을 앞선 예제 처럼 활용한다면 레베트리거 기반의 서버가 구현된다.

#### **엣지 트리거 기반의 서버 구현**

* 변수 errno을 이용한 오류의 원인 확인 → 리턴 값이 아닌 오류 발생 시 추가적인 정보 제공
* 넌 블로킹 IO로 소켓을 변경 → 엣지 트리거는 데이터 수신 시 한번만 이벤트가 발생되기 때문에 충분한 양의 버퍼를 마련한 다음 데이터를 읽어 들여야 하기에 데이터의 양에 따라 블로킹이 발생할 수 있기 때문에 넌 블로킹 IO로 변경한다. 그리고 errno 변수를 참조해서 EAGAIN 값이면 버퍼가 빈 상태인 것을 확인

```cpp
	int flag=fcntl(fd, F_GETFL, 0);
	fcntl(fd, F_SETFL, flag|O_NONBLOCK);
```

* BUF\_SIZE를 작게 해서 입력 버퍼에 데이터가 남아 있게 설정

```cpp
#include BUF_SIZE 4

int main(int argc, char *argv[])
{
	int serv_sock, clnt_sock;
	struct sockaddr_in serv_adr, clnt_adr;
	socklen_t adr_sz;
	int str_len, i;
	char buf[BUF_SIZE];

	struct epoll_event *ep_events;
	struct epoll_event event;
	int epfd, event_cnt;
	...;

	serv_sock=socket(PF_INET, SOCK_STREAM, 0);
	...;

	if(bind(serv_sock, (struct sockaddr*) &serv_adr, sizeof(serv_adr))==-1)
		printf("bind() error");
	if(listen(serv_sock, 5)==-1)
		printf("listen() error");

	epfd=epoll_create(EPOLL_SIZE);		// epoll 저장소 생성
	ep_events=malloc(sizeof(struct epoll_event)*EPOLL_SIZE);	// 이벤트 저장할 공간 생성

	event.events=EPOLLIN;
	event.data.fd=serv_sock;
	epoll_ctl(epfd, EPOLL_CTL_ADD, serv_sock, &event);		// 서버 소켓의 이벤트 등록

	while(1)
	{
		event_cnt=epoll_wait(epfd, ep_events, EPOLL_SIZE, -1);		// 이벤트 발생 대기
		if(event_cnt==-1)
		{
			puts("epoll_wait() error");
			break;
		}

		for(i=0; i<event_cnt; i++)
		{
			if(ep_events[i].data.fd==serv_sock)
			{
				adr_sz=sizeof(clnt_adr);
				clnt_sock = accept(serv_sock, (struct sockaddr*)&clnt_adr, &adr_sz);
				event.events=EPOLLIN;
				event.data.fd=clnt_sock;
				epoll_ctl(epfd, EPOLL_CTL_ADD, clnt_sock, &event);
                // 새로운 소켓도 등록
				printf("connected client: %d \n", clnt_sock);
			} // 발생한 소켓이 서버 소켓이면
			else
			{
					str_len=read(ep_events[i].data.fd, buf, BUF_SIZE);
					if(str_len==0)    // close request!
					{
						epoll_ctl(epfd, EPOLL_CTL_DEL, ep_events[i].data.fd, NULL);
                        // 소켓을 epoll 관리 대상에서 삭제
						close(ep_events[i].data.fd);		// 소켓 삭제
						printf("closed client: %d \n", ep_events[i].data.fd);
					}
					else
					{
						write(ep_events[i].data.fd, buf, str_len);    // echo!
					}

			}
		}
	}
	close(serv_sock);		// 서버 소켓 소멸
	close(epfd);			// epoll 인스턴스 소멸
	return 0;
}
```

→ 레벨 트리거일 경우 이벤트가 계속 발생

→ 엣지 트리거일 경우 한번만 발생

```cpp
int main(int argc, char *argv[])
{
	...;
	epfd=epoll_create(EPOLL_SIZE);
	ep_events=malloc(sizeof(struct epoll_event)*EPOLL_SIZE);

	int flag=fcntl(fd, F_GETFL, 0);
	fcntl(fd, F_SETFL, flag|O_NONBLOCK);	// 넌 블로킹 모드

	event.events=EPOLLIN;
	event.data.fd=serv_sock;
	epoll_ctl(epfd, EPOLL_CTL_ADD, serv_sock, &event);

	while(1)
	{
		event_cnt=epoll_wait(epfd, ep_events, EPOLL_SIZE, -1);
		if(event_cnt==-1)
		{
			puts("epoll_wait() error");
			break;
		}

		puts("return epoll_wait");
		for(i=0; i<event_cnt; i++)
		{
			if(ep_events[i].data.fd==serv_sock)
			{
				adr_sz=sizeof(clnt_adr);
				clnt_sock=accept(serv_sock, (struct sockaddr*)&clnt_adr, &adr_sz);
				int flag=fcntl(clnt_sock, F_GETFL, 0);
				fcntl(clnt_sock, F_SETFL, flag|O_NONBLOCK);		// 클라이언트 소켓도 넌 블로킹
				event.events=EPOLLIN|EPOLLET;
				event.data.fd=clnt_sock;
				epoll_ctl(epfd, EPOLL_CTL_ADD, clnt_sock, &event);
				printf("connected client: %d \n", clnt_sock);
			}
			else
			{
					while(1)
					{
						str_len=read(ep_events[i].data.fd, buf, BUF_SIZE);
						if(str_len==0)    // close request!
						{
							epoll_ctl(epfd, EPOLL_CTL_DEL, ep_events[i].data.fd, NULL);
							close(ep_events[i].data.fd);
							printf("closed client: %d \n", ep_events[i].data.fd);
							break;
						}
						else if(str_len<0)
						{
							if(errno==EAGAIN)		// 버퍼가 비어있는지 계속 확인
								break;
						}
						else
						{
							write(ep_events[i].data.fd, buf, str_len);    // echo!
						}
				}
			}
		}
	}
	close(serv_sock);
	close(epfd);
	return 0;
}
```

#### 레벨 트리거와 엣지 트리거 중에 뭐가 더 좋은건가요?

엣지 트리거 방식을 사용하면 `"데이터의 수신과 데이터가 처리되는 시점을 분리할 수 있다!"`

구현 모델 특성상 엣지 트리거가 좋은 성능을 발휘할 확률이 높지만 무조건 빨라질 수는 없다.
