# Linux에서의 Network 명령어

> Linux에서 많이 사용하는 Network 명령어를 정리해본다.

### 1. ping

> ping은 Packet Internet Groper 의 약자로 네트워크를 통해 지정된 호스트 이름 또는 IP 주소로 접근할 수 있는지를 확인하는 명령어이다. (네트워크 연결에 대한 테스트이다.)

```
$ ping google.com
```

### 2. nslookup

> nslookup은 DNS 서버에 질의하여, 도메인의 정보(IP)를 조회 하는 명령어이다.

```
$ nslookup google.com
```

### 3. ifconfig (or ip addr)

> ifconfig는 네트워크 인터페이스의 세부 정보를 확인하기 위해 사용하는 명령어이다. IP주소, 서브넷마스크, MAC주소, 네트워크 상태 등을 확인, 설정할 수 있다.

```
$ ifconfig
```

### 4. netstat

> netstat는 network statistics의 약자로 네트워크 접속, 라우팅 테이블, 네트워크 인터페이스의 통계 정보를 보여주는 명령어이다.

```
$ netstat -a
$ netstat -ltpn
```

### 5. traceroute

> traceroute는 목적지에 패킷이 도달하는데 걸리는 경로를 추적하여 관련된 네트워크 홉을 확인해주는 명령어이다.

```
$ traceroute google.com
```

### 6. ip

> ip는 ip 주소 할당, 경로 설정, 방화벽 규칙 조작 등의 네트워크 구성 관리를 위해 사용하는 명령어이다.

```
$ ip route add default via 192.168.1.1
```

### 7. tcpdump

> tcpdump는 지정된 인터페이스에서 네트워크 트래픽을 캡쳐하여 개별 패킷을 검사하고 네트워크 프로토콜을 분석할 수 있는 명령어이다.

```
$ tcpdump -i eth0
```

### 8. ssh

> ssh는 Secure Shell의 약자로 원격 호스트에 접속하기 위해 사용되는 보안 프로토콜입니다. 이를 통해 다른 Linux 시스템에 대한 보안 연결을 설정하고 로컬로 로그인한 것처럼 할 수 있는 명령어이다.

```
$ ssh user@192.168.xxx.xxx
```

### 9. nmap

> nmap은 Port Scanning 툴로 호스트나 네트워크를 스캐닝할 때 사용하는 명령어이다.

```
$ nmap google.com
```

### 10. iptables / nftables

> iptables는 리눅스상에서 방화벽을 설정하는 명령어이다.

```
$ iptables -t nat -A PREROUTING -s 210.0.0.0/24 -d 210.0.0.2/24 -p tcp --dport 80 -j DNAT --to 192.168.0.1
```

### 11. route

> route는 커널의 라우팅 테이블을 출력하거나 수정할 수 있는 명령어이다.

```
$ route
```

### 12. telnet

> telnet은 네트워크 상에서 다른 컴퓨터에 원격 로그인하거나 원격으로 명령을 실행하기 위한 프로토콜이다.

```
$ telnet example.com 23
```
