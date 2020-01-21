---
layout: post
title: Receiving multicast packets on Linux
category: Development
feature_image: https://picsum.photos/1300/400
---
리눅스 환경에서 2개 이상의 network interface 가 있을 때 특정 interface 로 멀티캐스트 패킷을 수신하도록 설정하는 방법입니다.
<!-- more -->


#### 일반 Wifi router 를 이용한 멀티캐스트 패킷 처리의 문제
Multicast 로 multimedia stream 을 수신하여 패킷을 가공한 뒤 클라이언트로 전송해주는 시스템에서, 멀티캐스트 수신이 제대로 되지 않거나 시간이 지날 수록 패킷 드랍 현상이 심해지는 문제가 발생하는 경우가 있습니다. 관련 자료를 검색해보니, 고가의 산업용 router 와 달리, 일반적인 가정용 공유기의 경우 멀티캐스트 패킷을 처리하는 데 부하가 크게 걸려 공유기의 성능에 지장을 주는 경우가 있는 듯 하여 네트워크 구성을 변경하여 해결하기로 합니다.

#### Test Environment
- Linux distro - CentOS 7
- 서버에 Network interface 를 추가하여
  - NIC 2 (_enp0s20f0u3_) 은 multicast source 와 1:1 connection 으로
  - NIC 1 (_eno1_) 는 wifi 유무선 공유기를 이용한 local private network 으로 구성

#### 설정 방법

##### NIC static configuration


아래 내용을 참고하여, NIC 2(멀티캐스트 수신용, _enp0s20f0u3_) 를 static address 로 설정해 줍니다. ifcfg-enp0s20f0u3 파일을 직접 수정하거나, 혹은 nmtui 등의 terminal UI 도구를 이용하셔도 됩니다. 여기에서는 192.168.0.10/24 를 사용했지만, 어차피 이 인터페이스는 multicast source 와 1:1 로 직접 연결되며, multicast packet 을 수신하는 용도이기 때문에 10.10.10.10/24 등 아무 주소나 넣어도 상관없습니다. multicast source 와 같은 subnet 으로 맞춰 줄 필요도 없습니다.

참고: CentOS 7 에서 NetworkManager 가 활성화되어 있는 경우에는 ifcfg 파일의 이름이 조금 다를 수 있습니다. ex. ifcfg-Wired_connection_1 등

> /etc/sysconfig/network-scripts/ifcfg-eno1 : DHCP 사용
```shell
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=dhcp
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=eno1
UUID=74222d7e-b5bb-4fe6-a602-27733e3f72b1
DEVICE=eno1
ONBOOT=yes
```
> /etc/sysconfig/network-scripts/ifcfg-Wired_connection_1 : 멀티캐스트 수신용, static address  사용
```shell
HWADDR=58:EF:68:7F:42:8A
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=enp0s20f0u3
UUID=1a57021d-ec44-31fd-8a95-a89d67f42e25
ONBOOT=yes
AUTOCONNECT_PRIORITY=-999
IPADDR=192.168.0.10
PREFIX=24
GATEWAY=192.168.0.1
```

##### Static route configuration

아래와 같이 route-enp0s20f0u3 파일을 작성해 줍니다. (route-"네트워크 인터페이스 장치명")

> /etc/sysconfig/network-scripts/route-enp0s20f0u3

```shell
224.0.0.0/4 dev enp0s20f0u3
```
224.0.0.0/4 는 [IETF](ietf.org) 에서 정의한 multicast address range 입니다. 모든 멀티캐스트 패킷은 이 인터페이스를 통해 받도록 정의했지만, 필요에 따라 239.255.0.0/16 과 같이 특정 address 대역만 이 인터페이스를 이용하도록 정의할 수도 있습니다.

이상과 같이 설정한 후 재부팅 또는 `systemctl restart network` 으로 네트워크를 재시작하고 나면 `route` 명령어를 통해 static routing 이 제대로 적용되었는지 확인할 수 있습니다. (아래 가장 아랫줄 224.0.0.0~ 라인)

```shell
[user@localhost]$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.1.1     0.0.0.0         UG    0      0        0 eno1
169.254.0.0     0.0.0.0         255.255.0.0     U     1002   0        0 eno1
169.254.0.0     0.0.0.0         255.255.0.0     U     1003   0        0 enp0s20f0u3
192.168.0.0     0.0.0.0         255.255.255.0   U     0      0        0 enp0s20f0u3
192.168.1.0     0.0.0.0         255.255.255.0   U     0      0        0 eno1
224.0.0.0       0.0.0.0         240.0.0.0       U     0      0        0 enp0s20f0u3
```

##### Kernel variable tuning

멀티캐스트 수신에 이용될 인터페이스에 대해, `rp_filter` 옵션을 비활성화해야 합니다. Reverse Path Filter 라는 것인데, 간단히 설명하자면 수신한 패킷의 source address 가 자신과 같은 subnet 이 아닐 경우 리눅스 커널이 이를 드랍합니다. 멀티캐스트의 경우 패킷의 source address 는 항상 수신하는 서버와는 다른 subnet 일 수밖에 없기 때문에 이 옵션을 비활성화해야 멀티캐스트 패킷의 수신이 가능해집니다. 기존에는 default 값이 0 (비활성화) 이었기 때문에 문제가 되지 않았으나, 리눅스 커널 2.6.31 버전 부터 기본값이 1 (활성화) 로 바뀌었기 때문에 직접 수정해 주어야 합니다.

또, 멀티캐스트 패킷을 수신하기 위해선 `icmp_echo_ignore_broadcasts` 옵션도 비활성화해야 합니다.

/etc/sysctl.d/ 에 rp_filter.conf 등 적당한 이름으로 파일을 생성하면 시스템 부팅시 자동으로 적용됩니다.

```shell
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.enp0s20f0u3.rp_filter = 0
net.ipv4.icmp_echo_ignore_broadcasts=0
```

#### TroubleShooting

재부팅 혹은 `systemctl restart network` 로 네트워크 재시작 시 route-enp0s20f0u3 파일의 내용이 제대로 반영되지 않는 경우가 있었습니다. 알고 보니 CentOS 7 부터 default network 관리자로 실행되는 NetworkManager service 가 활성화된 경우 이런 문제가 발생한다 합니다. NetworkManager 는 네트워크 인터페이스의 상태를 실시간으로 감지하여 변경 사항을 적용해 주는 서비스로, 보통의 서버 환경에서는 초기 구축을 끝내고 나면 네트워크 구성이 변경될 일은 거의 없으므로 이 서비스를 disable 하여 해결하였습니다.

```Shell
$ systemctl stop NetworkManager
$ systemctl disable NetworkManager
```
