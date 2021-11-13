# L860-GL on Linux

this method works on Arch Linux (5.15.2-arch1-1).

![image](https://user-images.githubusercontent.com/964412/141615325-aa5506df-3b73-4679-86a2-0a13b955cf5b.png)
![image](https://user-images.githubusercontent.com/964412/141615334-8f961715-d1a0-4ccb-9157-dc6d88cecc55.png)

# TODO

- write automated script

- rebuild kernel modules when kernel version has changed (DKMS?)

- integrate with ModemManager

- re-connect to network when connection is unstable / wake up from suspended state

# requirements

install below packages from package manager.

- libmbim

- kernel headers

- C Compiler toolchains (gcc, make, ...)


**if kernel version is >= 5.14, marking `iosm` module as blacklist is required**

```sh
# run as root. 
$ echo 'blacklist iosm' > /etc/modprobe.d/64-disable-iosm.conf
```

# clone repositories & build kernel modules

clone below repositories and build kernel modules.

커널 모듈 빌드가 이미 끝난 경우, <모뎀 활성화> 섹션부터 진행한다.

**시스템 업데이트 등으로 커널 버전이 바뀌면 다시 빌드해야 한다.**

## this resository (unstabler/l860gl-linux)

- this repository includes modem setup script provided from lenovo.

```sh
$ git clone https://github.com/unstabler/l860gl-linux.git
```

## GitHub `xmm7360/xmm7360-pci`

- this is open-source kernel module for xmm7360(L850), but you can send AT commands with this module by replacing text on source code. (`7360` -> `7560`)


### prepare

1. clone the repository

```sh
$ git clone https://github.com/xmm7360/xmm7360-pci.git
```

2. apply the patch included in this repository

```sh
$ cd xmm7360-pci
$ patch -p1 < xmm7360-to-xmm7560.dirty.patch
```

3. build the kernel module

```sh
$ make
```

## GitHub `x42/net-wwan-iosm`

- 현재 Linux upstream에 올라간 버전과는 조금 동떨어진, upstream에 합쳐지기 전의 오래된 버전이다.
- Linux 5.14~부터 기본으로 포함된 iosm 모듈은 이 버전과 동작이 많이 달라졌다.
  - 최신 버전의 iosm 모듈에서는 `/dev/wwanctrl` 대신 `/dev/wwanXmbimY`를 통해 MBIM 메시지를 보낼 수 있다. 그러나 L860-GL에서는 timeout이 발생하였다.

### prepare

1. clone the repository

```sh
$ git clone https://github.com/x42/net-wwan-iosm.git
```

2. build the kernel module

```sh
$ make
```

# activate modem

this requires root privilege.

1. load `xmm7560` module.

```sh
$ cd xmm7360-pci
$ insmod xmm7560.ko

# 정상적으로 로드가 완료되었는지 확인한다.
$ dmesg | tail -n 30
# if it loads successfully, you can send AT commands with /dev/ttyXMM0
$ ls /dev/ttyXMM0
```

2. activate the modem

```sh
# cat을 통해 출력을 consume 하지 않으면, AT 명령이 제대로 동작하지 않을 수 있다.
$ cat /dev/xmmXMM0 & 

# 모뎀 상태를 확인한다. 결과가 4,0이 반환되면 FCC lock이 걸린 것이다.
$ echo -ne 'AT+CFUN?\r' > /dev/ttyXMM0 

# unlock FCC-lock.
$ echo -ne 'at@nvm:fix_cat_fcclock.fcclock_mode=0\r' > /dev/ttyXMM0

# turn on modem 
$ echo -ne 'AT+CFUN=1\r' > /dev/ttyXMM0

# 다시 모뎀 상태를 확인한다. 결과가 1,0이 반환되면 정상이다.
$ echo -ne 'AT+CFUN?\r' > /dev/ttyXMM0

$ pkill cat
```

3. unload `xmm7560` module
```sh
$ rmmod xmm7560
```

# connect to network

**이 과정을 진행하던 도중 오류가 발생하면, 재부팅 후 <모뎀 활성화>부터 다시 진행한다.**

1. build `iosm` module

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

2. create virtual interface 

`wwan0`에서 VLAN 태그가 설정된 가상 인터페이스를 생성한다.

```sh
$ cd l860gl-linux/lenovo/imc_ipc_1A_V/scripts/
$ ./imc_start

# 정상적으로 끝났다면, inm0@wwan0이라는 인터페이스가 추가되어 있을 것이다.
$ ip link | grep inm0
```

3. connect to LTE network using `mbimcli`

```sh
# 트랜잭션을 시작한다.
$ mbimcli -v --no-close --noop -d /dev/wwanctrl
# 셀룰러 네트워크에 등록한다.
$ mbimcli -v --no-close --no-open=2 --register-automatic -d /dev/wwanctrl
# APN에 연결한다.
$ mbimcli -v --no-close --no-open=3 --connect="apn=lte.sktelecom.com" -d /dev/wwanctrl
```

4. set IP address manually

after the connection establishes, you can get output like below

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
# manually set IP address to inm0
$ ip addr add 10.203.178.24/24 dev inm0

# add default route (gateway)
$ ip route add default via 10.203.178.1 dev inm0

# set DNS server
$ echo 'nameserver 223.62.230.7\nnameserver 113.217.240.31' > /etc/resolv.conf
```

5. 인터넷 연결 확인하기

인터넷에 접속할 수 있을 것이다.

```sh
$ ping 1.1.1.1

$ wget -O- github.com
```
