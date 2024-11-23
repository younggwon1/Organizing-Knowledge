# **Connection reset by peer**

## 0. Connection reset by peer 란? 

Client Service -> Server Service 로 요청을 보냈지만 Server 쪽에서 연결이 닫혀 다시 연결하라는 RST (Reset) 패킷을 보내는 경우에 Connection reset by peer 에러가 발생한다.

즉, TCP RST가 수신되었으며 이제 연결이 닫혔다는 것을 의미한다. 이는 연결 끝에서 패킷이 전송되었지만 다른 끝에서 연결을 인식하지 못하는 경우에 발생한다. 연결을 강제로 종료하기 위해 RST 비트가 설정된 패킷을 다시 보낸다.

Client Service -> Server Service 연결에서 한 쪽만 연결되어 있는 좀비 커넥션이 생겼을 때 발생한다.

- 클라이언트 -> 서버로 `SYN` 패킷을 전송하였다.
- 서버 -> 클라이언트로 `SYN,ACK` 으로 응답하였다.
- 클라이언트 -> 서버로 `ACK`로 응답 후, 3-way Handshake가 완료되었다.
- 서버 -> 클라이언트로 `RST` 패킷을 전송한다.

정상적인 TCP 3-way Handshake가 진행된 후, 서버에서 RST 패킷을 전송하여 연결을 강제로 종료하였다. RST 패킷은 연결의 문제나 예외적인 상황에서 전송되므로 해당 연결에 문제가 발생했음을 알 수 있다.



## 1. 상황

~~~
recvAddress(..) failed: Connection reset by peer; nested exception is io.netty.channel.unix.Errors$NativeIoException: recvAddress(..) failed: Connection reset by peer
~~~

A Service -> B Service 시, Client 단(A Service) 에서 **Connection reset by peer** 에러가 발생.

- Client 쪽에서 `Connection reset by peer` 예외가 발생했지만, Server 단에서는 어떤 예외도 잡아낼 수 없었다.



## 2. 해결

webclient idle timeout 설정 등의 조정이 필요.

- Target Service 의 idle time, 초당 요청 수 등을 확인하여 webclient idle timeout 설정 등의 조정이 필요하다.

[#1774 Connection reset by peer exception](https://github.com/reactor/reactor-netty/issues/1774)



### 참고 문서

- <https://yangbongsoo.tistory.com/30>
- <https://lasel.kr/archives/740>
- <https://alden-kang.tistory.com/48>
- [Connection Reset by Peer 문제 해결](https://velog.io/@youngerjesus/Connection-Reset-by-Peer-%EB%AC%B8%EC%A0%9C-%ED%95%B4%EA%B2%B0)
- [[ 네트워크 쉽게 이해하기 22편 ] TCP 3 Way-Handshake & 4 Way-Handshake](https://mindnet.tistory.com/entry/%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-%EC%89%BD%EA%B2%8C-%EC%9D%B4%ED%95%B4%ED%95%98%EA%B8%B0-22%ED%8E%B8-TCP-3-WayHandshake-4-WayHandshake)
- https://skstp35.tistory.com/250#google_vignette