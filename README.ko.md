# Fibocom L860-GL을 Linux에서 사용하기

오랜 시간 삽질 끝에 Fibocom L860-GL LTE 모뎀을 Linux에서 동작시키는 것에 성공하였으므로 여기에 기록을 남긴다.

이 방법은 Arch Linux (5.15.2-arch1-1) 에서 동작함을 확인하였다.

![image](https://user-images.githubusercontent.com/964412/141615325-aa5506df-3b73-4679-86a2-0a13b955cf5b.png)
![image](https://user-images.githubusercontent.com/964412/141615334-8f961715-d1a0-4ccb-9157-dc6d88cecc55.png)

# TODO

- 자동화

- 커널 버전이 바뀔 때에 대한 대응 (DKMS 등을 사용하나?)

- ModemManager하고 연동하기

- 네트워크 재접속 및 suspend 상태에서 깨어났을 때 대응

# 전제 사항

다음 소프트웨어 패키지가 설치되어 있어야 한다.

- libmbim

- Linux 커널 헤더

- C 컴파일러 툴체인 (gcc, make, ...)


**커널 버전이 5.14 이상인 경우, 커널에서 기본으로 제공하는 iosm 모듈을 반드시 비활성화 한다.**

```sh
# root 권한으로 실행한다.
$ echo 'blacklist iosm' > /etc/modprobe.d/64-disable-iosm.conf
```

# 레포지토리 클론하기 & 커널 모듈 빌드하기

2개의 레포지토리를 클론하여 커널 모듈을 빌드한다.

커널 모듈 빌드가 이미 끝난 경우, <모뎀 활성화> 섹션부터 진행한다.

**시스템 업데이트 등으로 커널 버전이 바뀌면 다시 빌드해야 한다.**

## 이 레포지토리 (unstabler/l860gl-linux)

- Lenovo에서 제공하는 VLAN 설정 스크립트를 포함하고 있다.

```sh
$ git clone https://github.com/unstabler/l860gl-linux.git
```

## GitHub `xmm7360/xmm7360-pci`

- xmm7360(L850)을 위한 오픈 소스 커널 모듈이다.
- 본래는 xmm7560(L860-GL)이 아닌 xmm7560을 위한 모듈이지만, 소스 코드 내의 모든 `7360` 텍스트를 `7560`으로 변경하는 것 만으로도 AT 명령을 내릴 수 있다.


### 준비하기

1. 상기 레포지토리를 클론한다.

```sh
$ git clone https://github.com/xmm7360/xmm7360-pci.git
```

2. 이 레포지토리에 같이 포함된 patch를 적용한다.

```sh
$ cd xmm7360-pci
$ patch -p1 < xmm7360-to-xmm7560.dirty.patch
```

3. 커널 모듈을 빌드한다.

```sh
$ make
```

## GitHub `x42/net-wwan-iosm`

- 현재 Linux upstream에 올라간 버전과는 조금 동떨어진, upstream에 합쳐지기 전의 오래된 버전이다.
- Linux 5.14~부터 기본으로 포함된 iosm 모듈은 이 버전과 동작이 많이 달라졌다.
  - 최신 버전의 iosm 모듈에서는 `/dev/wwanctrl` 대신 `/dev/wwanXmbimY`를 통해 MBIM 메시지를 보낼 수 있다. 그러나 L860-GL에서는 timeout이 발생하였다.

### 준비하기

1. 상기 레포지토리를 클론한다.

```sh
$ git clone https://github.com/x42/net-wwan-iosm.git
```

2. 커널 모듈을 빌드한다.

```sh
$ make
```

# 모뎀 활성화

아래 사항은 모두 root 권한을 필요로 한다.

1. xmm7560 모듈을 로드한다.

```sh
$ cd xmm7360-pci
$ insmod xmm7560.ko

# 정상적으로 로드가 완료되었는지 확인한다.
$ dmesg | tail -n 30
# 정상적으로 로드가 완료되면, /dev/ttyXMM0을 통해 AT 명령을 보낼 수 있다.
$ ls /dev/ttyXMM0
```

2. 아래 명령을 통해 모뎀을 활성화한다.

```sh
# cat을 통해 출력을 consume 하지 않으면, AT 명령이 제대로 동작하지 않을 수 있다.
$ cat /dev/xmmXMM0 & 

# 모뎀 상태를 확인한다. 결과가 4,0이 반환되면 FCC lock이 걸린 것이다.
$ echo -ne 'AT+CFUN?\r' > /dev/ttyXMM0 

# FCC lock을 해제한다.
$ echo -ne 'at@nvm:fix_cat_fcclock.fcclock_mode=0\r' > /dev/ttyXMM0

# 모뎀을 켠다.
$ echo -ne 'AT+CFUN=1\r' > /dev/ttyXMM0

# 다시 모뎀 상태를 확인한다. 결과가 1,0이 반환되면 정상이다.
$ echo -ne 'AT+CFUN?\r' > /dev/ttyXMM0

# cat을 종료한다.
$ pkill cat
```

3. AT 명령을 내릴 필요가 없어졌으므로, xmm7560 모듈을 언로드한다.
```sh
$ rmmod xmm7560
```

# 네트워크 접속

아래 사항은 모두 root 권한을 필요로 한다.

**이 과정을 진행하던 도중 오류가 발생하면, 재부팅 후 <모뎀 활성화>부터 다시 진행한다.**

1. 새로 빌드한 iosm 모듈을 로드한다.

```sh
$ cd net-wwan-iosm
$ insmod iosm.ko

# 정상적으로 로드가 완료되었는지 확인한다.
$ dmesg | tail -n 30
# 정상적으로 로드가 완료되면, /dev/wwanctrl을 통해 MBIM 명령을 보낼 수 있다.
$ ls /dev/wwanctrl
# 정상적으로 로드가 완료되면, wwan0 네트워크 인터페이스가 추가된다.
$ ip link | grep wwan0
```

2. 가상 인터페이스 생성하기

`wwan0`에서 VLAN 태그가 설정된 가상 인터페이스를 생성한다.

```sh
$ cd l860gl-linux/lenovo/imc_ipc_1A_V/scripts/
$ ./imc_start

# 정상적으로 끝났다면, inm0@wwan0이라는 인터페이스가 추가되어 있을 것이다.
$ ip link | grep inm0
```

3. `mbimcli`를 통해 LTE 네트워크에 연결하기

```sh
# 트랜잭션을 시작한다.
$ mbimcli -v --no-close --noop -d /dev/wwanctrl
# 셀룰러 네트워크에 등록한다.
$ mbimcli -v --no-close --no-open=2 --register-automatic -d /dev/wwanctrl
# APN에 연결한다.
$ mbimcli -v --no-close --no-open=3 --connect="apn=lte.sktelecom.com" -d /dev/wwanctrl
```

4. 수동으로 IP 주소 설정하기

APN 연결이 끝나면, mbimcli에서 아래와 같은 출력이 나올 것이다.

```
[13 Nov 2021, 19:26:06] [Debug] [/dev/wwanctrl] Received message (translated)...
>>>>>> Header:
>>>>>>   length      = 160
>>>>>>   type        = command-done (0x80000003)
>>>>>>   transaction = 4
>>>>>> Fragment header:
>>>>>>   total   = 1
>>>>>>   current = 0
>>>>>> Contents:
>>>>>>   status error = 'None' (0x00000000)
>>>>>>   service      = 'basic-connect' (a289cc33-bcbb-8b4f-b6b0-133ec2aae6df)
>>>>>>   cid          = 'ip-configuration' (0x0000000f)
>>>>>> Fields:
>>>>>>   SessionId = '0'
>>>>>>   IPv4ConfigurationAvailable = 'address, gateway, dns'
>>>>>>   IPv6ConfigurationAvailable = 'dns'
>>>>>>   IPv4AddressCount = '1'
>>>>>>   IPv4Address = '{
>>>>>>     [0] = {
>>>>>>           OnLinkPrefixLength = '24'
>>>>>>           IPv4Address = '10.203.178.24'
>>>>>>     },
>>>>>>   }'
>>>>>>   IPv6AddressCount = '0'
>>>>>>   IPv6Address = '{
>>>>>>   }'
>>>>>>   IPv4Gateway = '10.203.178.1'
>>>>>>   IPv6Gateway = ''
>>>>>>   IPv4DnsServerCount = '2'
>>>>>>   IPv4DnsServer = '223.62.230.7, 113.217.240.31'
>>>>>>   IPv6DnsServerCount = '2'
>>>>>>   IPv6DnsServer = 'fd00:e13:101:c::7, fd00:213:101:c::31'
>>>>>>   IPv4Mtu = '0'
>>>>>>   IPv6Mtu = '0'


[/dev/wwanctrl] IPv4 configuration available: 'address, gateway, dns'
     IP [0]: '10.203.178.24/24'
    Gateway: '10.203.178.1'
    DNS [0]: '223.62.230.7'
    DNS [1]: '113.217.240.31'

[/dev/wwanctrl] IPv6 configuration available: 'dns'
    DNS [0]: 'fd00:e13:101:c::7'
    DNS [1]: 'fd00:213:101:c::31'
[13 Nov 2021, 19:26:06] [Debug] [/dev/wwanctrl] channel destroyed
[13 Nov 2021, 19:26:06] [Debug] Device closed
[/dev/wwanctrl] Session not closed:
            TRID: '5'
```

위와 같은 출력 정보를 확인하면서 IP 주소를 설정한다. 

여기서는 `10.203.178.24/24` 라는 IP 주소가 할당되었고, `10.203.178.1`을 게이트웨이 (default route)로써 설정한다.
DNS 서버는 `223.62.230.7`하고 `113.217.240.31`을 사용한다.

```sh
# inm0 인터페이스에 IP 주소를 설정한다.
$ ip addr add 10.203.178.24/24 dev inm0

# 기본 라우트 (게이트웨이)를 설정한다.
$ ip route add default via 10.203.178.1 dev inm0

# DNS 서버를 설정한다.
$ echo 'nameserver 223.62.230.7\nnameserver 113.217.240.31' > /etc/resolv.conf
```

5. 인터넷 연결 확인하기

인터넷에 접속할 수 있을 것이다.

```sh
$ ping 1.1.1.1

$ wget -O- github.com
```
