# dperf [![Apache V2 License](https://img.shields.io/badge/license-Apache%20V2-blue.svg)](https://github.com/baidu/dperf/blob/main/LICENSE)

English | [中文](README-CN.md)

dperf is a DPDK based 100Gbps network performance and load testing software.

## Advantage

- High performance：
  - Based on DPDK, dperf can generate huge traffic with a single x86 server: tens of millions of HTTP CPS，hundreds of Gbps throughput and billions of concurrent connections. 
- Detailed statistics：
  - Provides detailed statistics and identify every packet loss.
- Support multiple scenarios：
  - Load testing and stability testing for Layer 4 Load Balancer and other Layer 4 gateways 
  - Network  performance testing for servers on cloud 
  - Performance test of network package processing capability for NIC and CPU
  - Can be used as a high performance HTTP server or client for load testing

## Performance
### HTTP Connections per Second
|Client Cores|Server Cores|HTTP CPS|
|------------|------------|--------|
|1|1|2,101,044|
|2|2|4,000,423|
|4|4|7,010,743|
|6|6|10,027,172|

### HTTP Throughput per Second
|Client Cores|Server Cores|RX(Gbps)|TX(Gbps))|Client CPU Usage(%)|Server CPU Usage(%)|
|------------|------------|--------|---------|-------------------|-------------------|
|1|1|18|18|60|59|
|2|2|35|35|60|59|
|4|4|46|46|43|43|

### HTTP Current Connections
|Client Cores|Server Cores|Current Connections|Client CPU Usage(%)|Server CPU Usage(%)|
|------------|------------|-------------------|-------------------|-------------------|
|1|1|100,000,000|34|39|
|2|1|200,000,000|36|39|
|4|4|400,000,000|40|41|

### Client & Server Configuration
Above performance is obtained with below configurations:

- MEM: 512GB(hugepage 100GB)
- NIC: Mellanox MT27710 25Gbps * 2
- Kernel: 4.19.90

## Statistics
dperf outputs various statistics every second：
- TPS, CPS, various PPS
- Errors of TCP/Socket/HTTP
- Packets loss/drop
- Retransmissions of TCP Flags

```
seconds 22                 cpuUsage 52
pktRx   3,001,058          pktTx    3,001,025          bitsRx   2,272,799,040      bitsTx  1,920,657,600      dropTx  0
arpRx   0                  arpTx    0                  icmpRx   0                  icmpTx  0                  otherRx 0          badRx 0
synRx   1,000,345          synTx    1,000,330          finRx    1,000,350          finTx   1,000,350          rstRx   0          rstTx 0
synRt   0                  finRt    0                  ackRt    0                  pushRt  0                  tcpDrop 0
skOpen  1,000,330          skClose  1,000,363          skCon    230                skErr   0
httpGet 1,000,345          http2XX  1,000,350          httpErr  0
ierrors 0                  oerrors  0                  imissed  0
```

## Getting Started
    #set hugepages
    #edit '/boot/grub2/grub.cfg' like this, and reboot the OS
    #linux16 /vmlinuz-... nopku transparent_hugepage=never default_hugepagesz=1G hugepagesz=1G hugepages=8
    
    #download & build dpdk
    #download and unpack dpdk
    TARGET=x86_64-native-linuxapp-gcc
    #TARGET=arm64-armv8a-linuxapp-gcc
    cd /root/dpdk/dpdk-stable-19.11.10
    make install T=$TARGET -j16
    
    #build dperf
    cd dperf
    make -j8 RTE_SDK=/root/dpdk/dpdk-stable-19.11.10 RTE_TARGET=$TARGET
    
    #bind NIC
    modprobe uio
    modprobe uio_pci_generic
    
    #skip this step if using Mellanox interface card!!
    #Suppose your PCI number is 0000:1b:00.0
    /root/dpdk/dpdk-stable-19.11.10/usertools/dpdk-devbind.py -b uio_pci_generic 0000:1b:00.0
    
    #verify if dperf is installed successfully
    #start dperf server
    #dperf server bind at 6.6.241.27:80,  gateway is 6.6.241.1
    ./build/dperf -c test/http/server-cps.conf
    
    #send request to dperf server from a client
    curl http://6.6.241.27/

## Running the tests
Below example will start a HTTP CPS stress test.    
    #run dperf server
    ./build/dperf -c test/http/server-cps.conf
    
    #from another host, run dperf client
    ./build/dperf -c test/http/client-cps.conf

## Documentation
 - [configuration](docs/configuration.md)
 - [design](docs/design.md)
 - [build](docs/build.md) 
 - [statistics](docs/statistics.md)

## Limitation
 - dperf requires that the HTTP message is in one packet, which is not suitable for the test of layer 7 load balancer;
 - dperf requires exclusive use of the network interfaces.
 - dperf uses the FDIR feature. For the network interface that do not have this feature, dperf can only run in 'single-threaded' mode.
 - dperf does not have routing capability. It is recommended to build a test environment with a switch.
 - dperf does not support bonding. Vxlan will be in the next version.

## Contributing
dperf welcomes your contribution. See the [CONTRIBUTING](CONTRIBUTING.md) file for details.

## Authors
* Jianzhang Peng - *Initial work*

## License
dperf is distributed under the [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0).
