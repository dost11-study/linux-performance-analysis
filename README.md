# 리눅스 성능 분석 시작하기

인프런의 [리눅스 성능 분석 시작하기](https://www.inflearn.com/course/%EB%A6%AC%EB%88%85%EC%8A%A4-%EC%84%B1%EB%8A%A5-%EB%B6%84%EC%84%9D-%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0?cid=331322)를 정리한 내용입니다.

### uptime

시스템의 가동 시간, Load Average(서버의 CPU들이 받고 있는 부하 평균)

- R(CPU Bound)과 D(I/O Bound) 상태의 프로세스
- R이 높다면 CPU 및 스레드 개수 증가 필요, D가 높다면 IOPS가 높은 디바이스(스토리지)로 변경 필요
- vmstat을 통해서도 r, d 확인하는 것이 가능하며 Grafana로 그래프화하는 것이 좋다

### dmseg

커널 메시지 확인(OOME 발생 여부, SYN Flooding 여부) 

- 메모리가 꽉차면 커널이 OOM Killer 동작시킴
- oom_score가 높은 순서로 프로세스 종료 시킴

proc 파일 확인 후 OOME가 발생한 경우 AWS Serial Console을 통해 직접 하드웨어 시리얼 포트를 통해 kill 혹은 SysRq f가능

SYN 요청 시 SYN backlog에 쌓이고, ACT 요청 시 Listen Backlog에 쌓임

SYN Flooding을 방지하기 위한 SYN Cookie를 통해 SYN Backlog에 안쌓아두도록 설정 가능(SYN Backlog 꽉차면 실행되며 자원 고갈 현상 방지 가능)

하지만 TCP Option 헤더를 무시하므로, Windows scaling 등 성능 향상을 위한 옵션이 동작하지 않음(리눅스 커널 3.x 이후부터는 이 문제를 부분적으로 완화)

### free

메모리 사용 현황 확인

- free(어느 누구도 사용하고 있지 않은 메모리)
- available(애플리케이션에 실질적으로 할당 가능한 메모리)
    - available = free + buff/cache
- buff(블록 디바이스가 가지고 있는 블록 자체에 대한 캐시)/cache(I/O 성능 향상을 위해 사용하는 페이지 캐시)

buff/cache가 높다면 I/O가 많이 일어나는 환경

swap 영역은 메모리가 부족한 상황에서 사용되는 가상 메모리 공간 (주로 블록 디바이스의 일부 영역 사용)

최근 트렌드는 swap 영역 비활성화

- 컨테이너는 빠르게 다시 띄울 수 있기 때문에 쿠버네티스 환경에서는 대부분 swap 영역을 비활성화
- DB서버처럼 미션 크리티컬한 서버는 왜 swap 메모리 활성화가 필요한지(죽는 것보다 느린 게 낫다)

swap 개념

- swap in(swap 영역에 있던 페이지를 프로세스가 다시 접근할 때 디스크에서 RAM으로 불러오는 것)
- swap out(메모리가 부족할 때 커널이 오랫동안 안 쓴 페이지를 RAM에서 디스크(swap 영역)로 밀어내는 것)
- swap cached(한번 swap out됐다가 다시 swap in된 페이지가 있다고 가정할 때 이 페이지는 현재 RAM에 있지만, 디스크의 swap 영역에 있던 사본을 아직 지우지 않은 상태를 의미)

### df

디스크(파티션) 여유 공간 및 inode 공간 확인

파일 이름은 디렉토리에 저장하는데, inode는 실제 데이터 블록 포인터를 보관하고 있어서 파일의 실제 정보를 가지고 있음. 즉 inode는 파일 또는 디렉토리에 대한 메타데이터를 저장하는 구조체이며 파일과 디렉토리의 개수이다.

inode 사용률은 df -i를 통해서 확인 가능하다.

파일 핸들에 의해 파일을 지워도 사용량은 그대로인 경우가 있음 (프로세스가 파일 핸들 점유할 경우 lsof로 확인 가능) 프로세스가 종료되면 파일 핸들 점유가 끝나기 때문에 삭제됨

fs.file-max(file discriptor)는 동시에 파일을 몇 개까지 열 수 있는가를 정의하는 값이다.
리눅스에서는 모든 것이 파일.

No space left시 tmp 파일 생성도 안되고, ssh 접속도 안되는데 어떻게 해결할지

- 이미 열려있는 SSH 세션 활용
- 리눅스 ext4 파일시스템은 기본적으로 디스크의 5%를 root 전용으로 예약하니 루트 유저로 예약된 파일 디스크립터 활용
- SSH 완전 불가 시 콘솔 접근
    - AWS EC2 → Session Manager, GCP → Serial Console, Azure → Serial Console
    - 온프레미스는 IPMI/iDRAC/iLO 등 원격 콘솔
    - 물리 접근(서버실 직접 접근)

### top

프로세스들의 CPU 사용량 확인 및 프로세스 상태 확인

hotkey로는 1(각cpu)-d(실시간)-1(리프레시 간격)로 보는게 좋으며

us(cpu), wa(I/O)를 의미한다.

![1000024686.jpg](%EB%A6%AC%EB%88%85%EC%8A%A4%20%EC%84%B1%EB%8A%A5%20%EB%B6%84%EC%84%9D%20%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0/1000024686.jpg)

D는 vmstat의 D상태랑 동일하며 생명주기 D상태에서 다시 Waiting으로 돌아감

![1000024690.jpg](%EB%A6%AC%EB%88%85%EC%8A%A4%20%EC%84%B1%EB%8A%A5%20%EB%B6%84%EC%84%9D%20%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0/1000024690.jpg)

sshd, httpd 같은 데몬 프로세스는 사용자 제어없이 백그라운드에서 동작하는 프로세스

### netstat

소켓은 시스템 콜이며, socket()으로 소켓 생성하고 bind()로 ip 포트 할당하며 listen()으로 외부 접속 감지. 연결되면 서버, 클라이언트 모두 ESTABLISHED 상태가 된다.

![1000024700.jpg](%EB%A6%AC%EB%88%85%EC%8A%A4%20%EC%84%B1%EB%8A%A5%20%EB%B6%84%EC%84%9D%20%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0/1000024700.jpg)

TCP 패킷에 재전송 관련된 부분은 없으며 시퀀스 번호로 명시적으로 관리함. 대신 커널 영역에서 재전송 큐를 만들어서 관리함.

TCP 연결 해제 시에는 4단계로 구성되며, (커널) CLOSE_WAIT에서 (애플리케이션) LAST_ACK으로 명시적으로 리소스 정리해줘야함 (try catch finally). 이때 정리 안해주면 file descriptor 고갈 가능

![1000024701.jpg](%EB%A6%AC%EB%88%85%EC%8A%A4%20%EC%84%B1%EB%8A%A5%20%EB%B6%84%EC%84%9D%20%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0/1000024701.jpg)

CLOSE_WAIT가 있다는건 문제가 있을 수 있는데 tcpdump를 통해서 패킷 수집 및 분석해야함

### CORS, CSRF

CORS는 다른 요청에서 오는 요청을 허용할지 말지 검증 하는 단계이고(브라우저가 제어), CSRF는 요청이 정당한 사용자의 요청인지 검증하는 과정이다. 이때 OPTIONS 메서드인 CORS Preflight가 이 CORS가 허용되어 있는지 여부를 검사하는 과정이다.

### Nginx miss configuration

Nginx에는 worker_processes와 worker_connections 옵션이 존재하며, CPU 코어에 일치시키지 않으면 코어 수가 많아도 실제로 요청을 처리하는 워커가 1개뿐이라 CPU를 놀리는 상황이 발생함

### 간헐적인 네트워크 응답 지연, Timeout

1. 타임아웃 → 현재 상태가 정상이라고 판단할때까지 얼마나 기다릴 것인가?
언어, 라이브러리 마다 다르지만 크게 두 가지 타임아웃을 주로 가져감
    1. Connection Timeout (종단 간 연결을 처음 맺을 때)
    2. Read Timeout (종단 간 연결을 맺은 후 데이터를 주고 받을 때)
2. Rount Trip Time(RTT)
    1. 패킷이 종단 간 이동할 때 걸린 시간(물리적 거리에 따른 시간)
3. Timeout 설정 시 응답에 소요되는 시간과 RTT를 고려 해야 한다
    1. RTT를 모르더라도 일단 커널의 패킷 초기 타임아웃 값인 InitRTO(1초)를 기다림
    따라서 Connection Timeout 설정을 할 때는 3-Way handshake 과정 중 최소한 한 번의 패킷 유실 정도는 방어하도록 3초(InitRTO 1초 + RTT 고려) 정도로 설정하는 것이 좋음
    2. 패킷에 대한 응답이 Retransmission Timeout(RTO, 재전송) 이내에 도착하지 않으면 유실로 간주함
    RTO는 200ms가 최소값이며 굳이 수정하진 않음.
    3. Read Timeout도 Connection Timeout처럼 최소한 한 번의 패킷 유실을 방어하도록 설정해야하며 1초 (프로세싱 시간 + RTO 고려) 정도로 설정하는 것이 좋으며, 프로세싱 시간은 모두 다르다.

### 간헐적인 커넥션 종료 에러, Keepalive Timeout

간헐적으로 API 호출 시 커넥션 에러 발생

- Connection permaturely closed BEFORE response
- 간헐적으로 먼저 서버측에서 FIN 패킷을 보내서 연결이 끊어져 버리는 현상이 발생한다면 tcpdump를 통해서 분석할 수 있고, 문제는 커넥션 재활용을 위한 Keepalive 설정 때문이다.

해결방법으로는

1. 요청할 때 Connection: close 헤더를 포함시켜서 HTTP KeepAlive를 사용하지 않는다 → 성능저하 생김
2. 상대방 서버의 HTTP keepalive-timeout 늘려달라고 한다 → 에러 발생 빈도는 줄겠지만 근본적인 해결 방법 아님
3. 상대방 서버보다 Idle timeout을 짧게 가져간다(호출하는 쪽 수정) → 다음 요청을 전달하기까지 기다리는 시간을 줄인다

### 간헐적인 타임아웃, JVM Full-GC로 인한 장애

간헐적으로 API 호출 시 타임아웃 에러가 발생

- [WARN] - from application in [application-akka.actor](http://application-akka.actor).default-dispatcher-3 XXX is too slow [ElpasedTime: 10864 ms]

먼저 서버에 부하가 극심해서 패킷 생성이 늦어질 수도 있으므로 Load Average 및 CPU 사용량을 체크하고 애플리케이션에 영향을 줄만한 로직이 있는지 체크한다. → 없음에도 응답이 10초가 걸린다거나 하면 Full-GC가 발생해 GC 쓰레드를 제외한 다른 쓰레드들이 멈추는 현상이 발생할 수 있다.

### EC2 CPU 이상 동작, 물리서버의 CPU 불량으로 인한 장애

간헐적으로 CPU Usage가 높다는 알람이 발생 (모든 서버에서 발생하는 것이 아닌 일부 서버에서 발생)

첫 번째 조치

1. top을 통해 어떤 프로세스가 CPU를 사용하는지 확인
2. JAVA 프로세스가 100%를 사용하는 현상 발생
3. 구글링을 통해 Java, 특히 tomcat이 CPU를 점유하는 현상 검색
4. /dev/random 블로킹 이슈(난수 생성) 확인

하지만, 위의 경우가 원인이었다면 모든 서버에서 동일하게 발생해야함

두 번째 조치

이슈가 발생하는 서버만 서비스에서 제외(ASG 즉 오토스케일링 그룹에서 제외, 로드밸런서 제외 등) 및 삭제 방지 기능 활성화(안하면 인스턴스 죽이고 다시 실행)

![image.png](%EB%A6%AC%EB%88%85%EC%8A%A4%20%EC%84%B1%EB%8A%A5%20%EB%B6%84%EC%84%9D%20%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0/image.png)

모니터링 에이전트 CPU가 131%이며 top 명령어가 100%를 치게 만드는 이상한 현상 발생

→ 서버 자체가 이상하다

/proc/cpuinfo

![image.png](%EB%A6%AC%EB%88%85%EC%8A%A4%20%EC%84%B1%EB%8A%A5%20%EB%B6%84%EC%84%9D%20%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0/image%201.png)

cpu 클럭이 비정상적인 수치였음.

두 기능 C-State(전력 소모량 줄이기 위해 일부 CPU 코어를 비활성화), P-State(작업 부하에 따라서 CPU의 전압과 MHz를 조절하는 기능)에 의해 영향 받을 수 있음 → turbostat 명령어를 통해 강제로 CPU MHz를 고정시킬 수 있음

![image.png](%EB%A6%AC%EB%88%85%EC%8A%A4%20%EC%84%B1%EB%8A%A5%20%EB%B6%84%EC%84%9D%20%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0/image%202.png)

하지만 그래도 해결되지 않는다면, CPU 하드웨어 자체의 고장일 수 있음.

### 근본적인 타임아웃 문제를 해결하는 분석 방법

netstat을 통해 네트워크 연결은 잘 되어 있는지 확인하고 → tcpdump를 통해 패킷들의 흐름을 수집 후 분석한다

![image.png](%EB%A6%AC%EB%88%85%EC%8A%A4%20%EC%84%B1%EB%8A%A5%20%EB%B6%84%EC%84%9D%20%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0/image%203.png)

가장 많이 쓰는 명령어 uptime(로드 에버리지), dmesg(커널 메시지), free(메모리 사용량, available), df(디스크 사용량), top(cpu 잘쓰고있는지), netstat(네트워크 정보), iostat(디스크i/o 통계), mpstat(CPU 사용률)