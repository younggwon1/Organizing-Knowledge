# DNS Query Failed 트래킹

## 개요
운영 중인 서비스의 Logs 에 `io.netty.resolver.dns.DnsResolveContext$SearchDomainUnknownHostException` 에러가 발생하여 이를 해결하기 위해 `ndots 설정을 2`로 하여 DNS Query 요청을 줄이기 위한 방안을 고려해보았습니다.

추가로 코드 단에서 조치할 수 있는 방안을 마련하여 관련 이슈를 해결해봅니다.

1. `UDP Timeout 시 TCP Fallback`
> netty 버전을 netty-4.1.105.Final 이상 으로 올려 UDP Timeout 시 TCP Fallback 처리를 할 수 있도록 해주는 방안을 고려하여 서비스 안정성을 확보합니다.

- httpClient.resolver(nameResolverSpec -> nameResolverSpec.retryTcpOnTimeout(true));

- DnsNameResolverBuilder.socketChannelFactory(...., true)
   - https://github.com/netty/netty/pull/13757/files
   - https://github.com/reactor/reactor-netty/issues/3059

2. `spring cloud gateway`
> spring cloud gateway 를 사용하고 있는 서비스에 대해 다음과 같은 조치를 하여 서비스의 안정성을 확보합니다.

- httpClient.resolver(DefaultAddressResolverGroup.INSTANCE); 를 설정하면 Netty의 비동기 DNS 해석 기능을 활용하여 비동기 방식으로 DNS를 조회하게 되기에 성능이 향상되고 또한 DNS 캐싱이 지원되기에 동일한 도메인 요청 시 DNS 조회를 줄일 수 있다고 합니다.
   - https://github.com/reactor/reactor-netty/issues/1431
   - https://stackoverflow.com/questions/67075057/unknowhhostexception-using-spring-cloud-starter-gateway-with-spring-boot-2-4-0-a


특히, 1번 방안으로 코드 단에서 조치를 하여 UDP Timeout 시 TCP Fallback을 통해 정상 처리될 수 있도록 하기 위해 테스트를 진행합니다.

## 테스트
> Chaos Engineering 도구인 [`litmus`](https://litmuschaos.github.io/litmus/experiments/categories/contents/)를 사용하여 인위적으로 DNS Query Timeout을 발생하여 TCP Fallback이 정상적으로 동작하는지 확인해봅니다.

이때 사용한 환경은 [`pod network latency`](https://litmuschaos.github.io/litmus/experiments/categories/pods/pod-network-latency/#introduction) 를 사용하였습니다.

- query '5473' via UDP timed out after `5000 milliseconds` (no stack trace available) 를 참고하여 NETWORK_LATENCY를 6s 로 설정하여 의도적으로 UDP timed out 이 발생하도록 합니다.

```yaml
spec:
  appinfo:
    appns: default
    applabel: "app.kubernetes.io/name={deployment name}"
    appkind: deployment
  chaosServiceAccount: litmus-admin
  experiments:
  - name: pod-network-latency
    spec:
      components:
        env:
        - name: NETWORK_LATENCY
          value: '6000' #in ms
        # supports comma separated destination ips
        - name: DESTINATION_IPS
          value: '172.20.0.10'
        # supports comma separated destination hosts
        - name: DESTINATION_HOSTS
          value: 'kube-dns.kube-system.svc.cluster.local'
        # supports comma separated destination ports
        - name: DESTINATION_PORTS
          value: '53'
        - name: TOTAL_CHAOS_DURATION
          value: '60'
```

```
$ tcpdump -i any port 53 (UDP → TCP 패킷 전환 확인 가능)
- TCP SYN 패킷은 "Flags [S]"로 표시
- SYN+ACK는 "Flags [S.]"로 표시
- 연결 종료(RST)는 "Flags [R]"로 표시

# UDP (kube-dns service 질의)
08:09:30.773565 eth0  Out IP {pod name}.60639 > 172.20.0.10.53: 15332+ [1au] A? {service}.{namespace}.svc.cluster.local. (67)
08:09:30.773994 eth0  In  IP 172.20.0.10.53 > {pod name}.60639: 15332*- 1/0/1 A 172.20.171.138 (121)

# TCP (coredns pod 질의)
08:09:29.775440 eth0  Out IP {pod name}.35900 > 10.119.34.189.53: Flags [S], seq 3447715082, win 62727, options [mss 8961,sackOK,TS val 3516720673 ecr 0,nop,wscale 7], length 0
08:09:29.775620 eth0  In  IP 10.119.34.189.53 > {pod name}.35900: Flags [S.], seq 2064863880, ack 3447715083, win 62643, options [mss 8961,sackOK,TS val 1406768545 ecr 3516720673,nop,wscale 7], length 0
08:09:29.816251 eth0  Out IP {pod name}.39484 > 10.119.37.157.53: Flags [S], seq 3508263417, win 62727, options [mss 8961,sackOK,TS val 4067914733 ecr 0,nop,wscale 7], length 0
08:09:29.817264 eth0  In  IP 10.119.37.157.53 > {pod name}.39484: Flags [S.], seq 4205410720, ack 3508263418, win 62643, options [mss 8961,sackOK,TS val 1079478679 ecr 4067914733,nop,wscale 7], length 0
08:09:35.775640 eth0  Out IP {pod name}.35900 > 10.119.34.189.53: Flags [R], seq 3447715083, win 0, length 0
08:09:35.817277 eth0  Out IP {pod name}.39484 > 10.119.37.157.53: Flags [R], seq 3508263418, win 0, length 0
```

UDP 패킷만 보였던 tcpdump 로그에서 TCP 패킷 (SYN, SYN+ACK, RST) 이 확인되었습니다.
UDP Timeout 시 TCP Fallback 동작이 되었다는 것을 확인하였는데 문제가 발생했습니다.

기존 발생하였던 `UDP timed out after 5000 milliseconds` 에러 외에 `Received TCP DNS response with unexpected ID` 에러가 같이 발생하여 API 응답 `500`이 발생하였습니다. 정상적으로 처리되지 않았음을 의미합니다.
```
caused by io.netty.resolver.dns.DnsNameResolverTimeoutException: [5473: /172.20.0.10:53] DefaultDnsQuestion({service}.{namespace}.svc.cluster.local. IN A) query '5473' via UDP timed out after 5000 milliseconds (no stack trace available)

Suppressed: io.netty.resolver.dns.DnsNameResolverException: [9403: 172.20.0.10/10.119.20.198:53] DefaultDnsQuestion({service}.{namespace}.svc.cluster.local. IN A) Received TCP DNS response with unexpected ID (no stack trace available)
```

`Received TCP DNS response with unexpected ID` 에러가 왜? 발생하는지 파악해봐야합니다.
> CoreDNS 응답의 ID가 요청(TCP DNS 질의)한 ID와 다름을 의미합니다. DNS 프로토콜에서는 요청(request)과 응답(response)이 매칭되기 위해 Transaction ID를 사용하는데, 응답에서 이 ID가 예상과 다르면 클라이언트는 해당 응답을 무시하거나 오류로 처리하게 됩니다.

Received TCP DNS response with unexpected ID 이슈를 분석하다 보니 다음과 같이 정리해볼 수 있을 것 같습니다.

`DNS 요청/응답 ID Mismatch`

Netty는 각 DNS 쿼리에 대해 고유한 transaction ID를 부여하는데, 응답이 들어올 때 이 ID를 비교하여 매칭 여부를 확인합니다. 하지만 다음 상황에서 Mismatch가 발생할 수 있습니다.
- 여러 쿼리를 병렬로 TCP로 전송했을 때 응답이 예상 순서와 다르게 도착
- 같은 커넥션을 공유하다가 응답 ID가 꼬임
- DNS 리졸버가 TCP 재사용을 비정상적으로 처리

`retryTcpOnTimeout(true)` 옵션은 UDP Timeout이 발생하면 TCP로 재시도(fallback) 하도록 하는 설정이기는 하나 기본 Netty 리졸버 (DefaultAddressResolverGroup)를 사용하기에 응답 처리 중 ID Mismatch 문제를 해결하기는 어렵습니다.

따라서 `retryTcpOnTimeout(true)` 만으로는 문제가 해결되지 않을 것으로 보여 Netty의 DnsNameResolver를 커스터마이징하여 응답 ID Mismatch를 방지하는 방안으로 가야합니다.

EX)
```java
@Configuration
class WebClientConfig {

    @Bean
    fun webClient(): WebClient {

        // 1. Netty EventLoopGroup 생성
        val group = NioEventLoopGroup(1)
        val dnsResolver = DnsNameResolverBuilder(group.next())
            .channelType(NioDatagramChannel::class.java) // UDP 기본
            .socketChannelFactory(ReflectiveChannelFactory(NioSocketChannel::class.java)) // TCP fallback
            .optResourceEnabled(false)
            .queryTimeoutMillis(3000)
            .build()

        // 2. 커스텀 AddressResolverGroup 생성
        val customResolverGroup = object : AddressResolverGroup<InetSocketAddress>() {
            override fun newResolver(executor: EventExecutor): NameResolver<InetSocketAddress> {
                return dnsResolver
            }
        }

        // 3. HttpClient 구성
        val client = HttpClient.create()
            .resolver(customResolverGroup) // 여기서 커스텀 리졸버 지정
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)
            .option(ChannelOption.SO_KEEPALIVE, true)
            .responseTimeout(Duration.ofMillis(3000))

        // 4. WebClient 빌더
        val connector = ReactorClientHttpConnector(client)
        return WebClient.builder()
            .clientConnector(connector)
            .defaultHeader("User-Agent", SERVICE_NAME)
            .build()
    }
}
```

하지만 WebClientConfig 를 수정하는 작업이다보니 webClient를 사용하는 전반적인 기능에 영향이 갈 수도 있을 것으로 보여 `dnsConfig ndots 2 설정` 및 `호스트 변경`을 통해 `SearchDomainUnknownHostException` 이슈가 재현되는지 확인하는 방향으로 결정하였습니다.
