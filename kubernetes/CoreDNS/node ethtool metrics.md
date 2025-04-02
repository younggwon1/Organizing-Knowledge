# Node ethtool Metrics

## 개요
ifconfig 명령어 수행 시 아래와 같은 정보를 얻을 수 있습니다. (특히, eth0)
```shell
eth0      Link encap:Ethernet  HWaddr 0A:1D:C5:B7:AF:E9
          inet addr:10.119.39.226  Bcast:10.119.39.255  Mask:255.255.252.0
          inet6 addr: fe80::81d:c5ff:feb7:afe9/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:9001  Metric:1
          RX packets:10600891909 errors:6470 dropped:0 overruns:0 frame:6470
          TX packets:10433661166 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:23957197270586 (21.7 TiB)  TX bytes:6009346862705 (5.4 TiB)
```

또한, ethtool -S 명령어를 사용하여 error를 자세하게 확인할 수 있습니다.
```shell
$ ethtool -S em1 | grep -i error
     rx_errors: 310
     tx_errors: 0
     rx_over_errors: 0
     rx_crc_errors: 310
     rx_frame_errors: 0
     rx_fifo_errors: 0
     rx_missed_errors: 0
     tx_aborted_errors: 0
     tx_carrier_errors: 0
     tx_fifo_errors: 0
     tx_heartbeat_errors: 0
     rx_length_errors: 0
     rx_long_length_errors: 0
     rx_short_length_errors: 0
     rx_csum_offload_errors: 1
```

이때 주요하게 볼 부분이 다음인데, 해당 packets가 의미하는 것이 무엇인지 알아봐야합니다.
- RX packets:10600891909 errors:6470 dropped:0 overruns:0 frame:6470
- TX packets:10433661166 errors:0 dropped:0 overruns:0 carrier:0


## RX / TX packet이란?
- RX packets : R (Receive) , Interface 기준으로 들어오는 패킷(Receive Packet)
- TX packets : T (Transmit) , ​Interface 기준으로 전달 혹은 나가는 패킷(Transmit Packet)

### errors
- 에러가 발생한 모든 패킷 카운트
- 너무 긴(짧은) 프레임 에러, 링-버퍼 오버플로우 에러, CRC 에러, 프레임 정렬 에러, FIFO 오버런, 패킷 분실 등 아래 3개 overruns, dropped, frame 등을 모두 포함한 에러 카운터입니다.

### overruns
- FIFO 오버런, 버퍼가 꽉차서 버린 패킷 카운트
- 이더넷이 처리할 수 없을 정도로 빠르게 자료가 오고감으로써 손실된 패킷의 갯수를 의미합니다.

### dropped
- 의도하지 않는 패킷 카운트
- linux buffers에 공간이 없어서 버려진 패킷. "no space in linux buffers" 표현, VLAN tags가 맞지 않거나 IPv6 설정이 없는데 IPv6 패킷이 들어왔을 때 버린 패킷입니다.

### frame
- 프레임 정렬 에러, 프레임 길이가 틀린 패킷 카운트
- 수신 프레임이 바이트 단위가 아닌 여분의 비트를 포함하는 패킷
- frame은 정렬되지 않은 프레임만 계산하므로 길이가 8로 나눌 수 없는 프레임을 의미
- 길이 때문에 유효한 프레임이 아니며 단순히 폐기됨

## 반영
prometheus node exporter 를 활용하여 tx, rx packet 등의 모든 NIC 메트릭을 집계하여 송 / 수신을 분석하여 에러를 트래킹합니다.

메트릭 | 의미 | 예시 | 중요도
-- | -- | -- | --
node_ethtool_tx_timeout | 패킷 전송 타임아웃 발생 수 | 네트워크 장애 발생 시 이 값이 증가 | ⭐️⭐️⭐️⭐️⭐️
node_ethtool_total_resets | 인터페이스가 리셋된 횟수 (자동/수동 포함) | 드라이버가 오류로 리셋된 흔적 | ⭐️⭐️⭐️⭐️
node_ethtool_interface_down / interface_up | 인터페이스 상태 | up=1, down=1이면 인터페이스 끊김 감지 | ⭐️⭐️⭐️⭐️
node_ethtool_wd_expired | Watchdog 타이머 만료 (심각 오류) | 드라이버가 응답하지 않아 리셋 | ⭐️⭐️⭐️⭐️
node_ethtool_bad_rx_desc_num / bad_rx_req_id / bad_tx_req_id | 비정상 RX/TX 패킷 수신/전송 | 패킷이 깨진 상태로 도착하거나 전송 실패 | ⭐️⭐️⭐️
node_ethtool_missing_intr | 인터럽트 누락 횟수 | 인터럽트 로스트로 성능 문제 야기 가능 | ⭐️⭐️⭐️
node_ethtool_device_request_reset | 디바이스에서 리셋 요청한 횟수 | ENA 드라이버 문제 가능 | ⭐️⭐️⭐️
node_ethtool_tx_desc_malformed, rx_desc_malformed | 전송/수신 패킷 디스크립터 오류 | 드라이버나 하드웨어 문제 | ⭐️⭐️⭐️


```diff
  extraArgs:
  - --collector.ethtool
- - --collector.ethtool.metrics-include=^(bw_in_allowance_exceeded|bw_out_allowance_exceeded|conntrack_allowance_exceeded|linklocal_allowance_exceeded|pps_allowance_exceeded)$
+ - --collector.ethtool.metrics-include=^(bw_in_allowance_exceeded|bw_out_allowance_exceeded|conntrack_allowance_exceeded|linklocal_allowance_exceeded|pps_allowance_exceeded|tx_timeout|total_resets|interface_(up|down)|bad_(rx_desc_num|rx_req_id|tx_req_id)|missing_intr|wd_expired|device_request_reset|tx_desc_malformed|rx_desc_malformed)$
  - --collector.ethtool.device-include=eth0
```

prometheus node exporter 로그
```shell
time=2025-03-25T05:17:58.548Z level=INFO source=ethtool_linux.go:99 msg="Parsed flag --collector.ethtool.device-include" collector=ethtool flag=eth0
time=2025-03-25T05:17:58.548Z level=INFO source=ethtool_linux.go:105 msg="Parsed flag --collector.ethtool.metrics-include" collector=ethtool flag=^(bw_in_allowance_exceeded|bw_out_allowance_exceeded|conntrack_allowance_exceeded|linklocal_allowance_exceeded|pps_allowance_exceeded|tx_timeout|total_resets|interface_(up|down)|bad_(rx_desc_num|rx_req_id|tx_req_id)|missing_intr|wd_expired|device_request_reset|tx_desc_malformed|rx_desc_malformed)$
```

## 구축
Grafana Dashboard 를 구축하여 메트릭을 확인할 수 있도록 합니다.
