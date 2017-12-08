# Basic and Inexpensive Network Diode (BIND)

## Basic principle

Transmit data one way.

Challenges:

  * Impossible to detect loss of transmission
  * Very few protocol and technologies dedicated to this use-case

Thought it is possible to do the same with a software or hardware firewall, the idea here is to create a electronic isolation that is more difficult to bypass and less subject to human error.

<p align="center"><img src="images/01_generalview.svg" width="80%"></p>

## Some ideas

### The physical break

Fast Ethernet networks use two copper pair to communicate : one pair for each way. It is easy to trick the network by disconnecting the receiving copper pair on one side so the transmission can only occur in one way. The receiver gateway is not aware that the data it send is lost.

There is still a problem : the emitter gateway never receives data so it's Ethernet port remains inactive thinking no cable is actually plugged. In order to force the activation of the port it is possible to "plug it on itself" by wiring the emitting pair on the receiving copper pair. That will create an echo and the port will go up because it will sense it's own NLP signals.

Note this will not work on gigabit networks.

### USB and Ethernet

Two Ethernet ports would be needed on the emitter and the receiver gateway to relay the data. Sadly, most SoC only have one on-board.

Ethernet over USB is a good alternative. Performance is not as great than on a dedicated Ethernet port especially regarding latency. In this use-case, latency is not an issue and neither is bandwidth because the diode is only 100 Mbps.

USB also has an interesting feature by providing the board with both data and electric power. This setup does not require an extra power supply.

### Pseudo synchronous transfer

Similar projects use a store and forward approach were files are recursively store and transmitted from one link to the next in the transmission chain. This approach implies the transfered files been fully stored on both gateways at some point. It means the storage has to be sized appropriately and always available for writing.

Another approach is to transmit data as a continuous stream. This way the files can be transmitted to the next hop without been fully loaded in the first place. The major drawback is induce by the synchronous nature of this transmission. The stream has to be stable and carefully adjusted.

## Prototype

Two [Orange PI zero](http://www.orangepi.org/orangepizero/) development boards with the following benefits:

  * Low footprint
  * Low cost (around 15€ for the 512MB of RAM at the end of 2017)
  * Low power consumption allowing USB only powering
  * Ethernet port (managed by the SoC)

### Overview

<p align="center"><img src="images/10_prototype.svg" width="80%"></p>

![General view](images/11_prototype.jpg)

## Software

Serveral tools are used:

  * BASH...
  * [mbuffer](http://www.maier-komor.de/mbuffer.html): buffers I/O operations and displays the throughput rate. It is multi-threaded, supports network connections, and offers more options than the standard buffer. This is used on the emitting and the receiving gateway to improve the data flow.
  * [UDPcast](http://www.udpcast.linux.lu/): a file transfer tool that can send data simultaneously to many destinations on a LAN using broadcast and multicast in UDP. This is the real thing. It can transmit data over unidirectionnal link like a satellite transmission. It does error correction and badwidth throttling.
  * [socat](http://www.dest-unreach.org/socat/doc/socat.html): a command line based utility that establishes two bidirectional byte streams and transfers data between them. Is a general purpose relay used here to send and receive data on TCP.

<p align="center"><img src="images/01_generalview.svg" width="80%"></p>

### On the client

```bash
while true; do
	mbuffer -i testfile00.raw -t -m 32M -4 -O 10.42.1.1:2222
	sleep 16
done
```

### One the emitting gateway

```bash
while true; do
	export TMPDIR=/tmp; export HOME=/root; mbuffer -q -t -m 32M -4 -I 2222 | udp-sender --no-progress --interface eth0 --rexmit-hello-interval 100 --portbase 9000 --async --fec 8x16/64 --max-bitrate 50m --autostart 2 --nokbd
	sleep 2
done
```

### On the receiving gateway

```bash
while true; do
	export TMPDIR=/tmp; export HOME=/root; udp-receiver --no-progress --interface eth0 --portbase 9000 | mbuffer -t -m 32M | socat - TCP4:10.42.0.115:2223,forever,keepalive,keepidle=1,keepintvl=1
	sleep 1
done
```

### On the client

```bash
socat -u TCP4-LISTEN:2223,bind=10.42.0.115,fork,reuseaddr SYSTEM:"md5sum"
```
