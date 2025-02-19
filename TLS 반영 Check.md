# TLS 반영 Check

TLS 버전 조치 ( 1.1 이상 ) 에 대해 잘 반영이 되었는지 확인하는 명령어

~~~
$ curl --tls-max 1.1 -v ~~~
~~~



아래와 같이 나오면 정상적으로 반영된 것.

~~~
$ curl --tls-max 1.1 -v https://{domain name}/
*   Trying {public ip}:443...
* Connected to {domain name} ({public ip}) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*  CAfile: /etc/ssl/cert.pem
*  CApath: none
* (304) (OUT), TLS handshake, Client hello (1):
* error:1404B42E:SSL routines:ST_CONNECT:tlsv1 alert protocol version
* Closing connection 0
curl: (35) error:1404B42E:SSL routines:ST_CONNECT:tlsv1 alert protocol version
~~~



참고 문서

- [TLSv1.1_2016 vs TLSv1.2_2021](https://docs.aws.amazon.com/ko_kr/AmazonCloudFront/latest/DeveloperGuide/secure-connections-supported-viewer-protocols-ciphers.html)
