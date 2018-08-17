# Testing SCTP protocol with docker 18.06-ce

As of Docker 18.03.0-ce (2018-03-21), SCTP has been officially integrated into the mainline of docker and swarm releases [[1](https://docs.docker.com/release-notes/docker-ce/#18030-ce-2018-03-21)]. Before that, the support was implemented mainly by user ishidawataru on github, and merged through this pull request [2](https://github.com/moby/moby/pull/33922).

I personally did the tests that are described below based on the 18.06.0-ce release where nothing related to SCTP was announced [[3](https://github.com/docker/docker-ce/releases/tag/v18.06.0-ce)]. My machine is an Ubuntu 16.04.5 LTS with a Linux 4.15.0-32-generic kernel on an amd64 architecture. 

## iperf3 with sctp support

Since version 3.1, iperf3 comes with the support of sctp. Depending on what version your package manager offers, it may or may not support it. For instance my ubuntu 16.04.5 still provides the 3.0.11-1 version that does not support sctp. Several versions of iperf exist on docker hub, and the ones I used had the same issue. I built my own in a dedicated repo [[4](https://github.com/sofianinho/iperf3-docker)], you can use it as such or rebuild another version as instructed there.

## other software to test sctp

The other software I used to test sctp, is one from the original library sctp for go that was included in the release here [[5](https://github.com/ishidawataru/sctp)]. In order to use a docker image of the echo-server example proposed in the repo, I wrote the following dockerfile of which the resulting image is `sofianinho/echo-sctp:scratch` on the docker hub.

```terminal
FROM golang:1.10-alpine

RUN apk add -U ca-certificates git \
  && git clone https://github.com/ishidawataru/sctp.git src/github.com/ishidawataru/sctp\
  && cd  src/github.com/ishidawataru/sctp/example \
  && CGO_ENABLED=0 go build -ldflags '-w -extldflags "-static"'

FROM scratch

COPY --from=0 /go/src/github.com/ishidawataru/sctp/example/example /app/sctp-echo

ENTRYPOINT ["/app/sctp-echo"]
```

In order to run it, at least for the server, you should know in advance the ip address the container will be associated with. Typically, by guessing the next one on the subnet (172.17.0.2 if it is the only container, for e.g.). I know it's not ideal, but the objective of this post is beyond that. Know that it is possible to fix ip addresses for overlay networks, but we'll come to that later.

```sh
# the server container will have the address 172.17.0.2
docker run -it -p 2000:2000/sctp --rm sofianinho/echo-sctp:scratch -server -port 2000 -ip 172.17.0.2
# the client for the echo server
docker run -it --rm sofianinho/echo-sctp:scratch -port 2000 -ip 172.17.0.2
```

## Tests

### SCTP Server inside a container and client outside

This scenario is meant to expose the SCTP port of the server, and then the client connects to it. The client itself is a binary on my machine or inside a VM (routing appropriately).

#### With iperf3
```sh
# Server in a container
docker run -it --rm -p 5201:5201/sctp sofianinho/iperf3:3.6-ubuntu18.04 -s
```
```sh
# client installed on my machine, and another one in an ubuntu VM
iperf3 -c 172.17.0.2 -p 5201 --sctp -i 1 -t 5  --bandwidth 10G --len 10k  --nstreams 4 --parallel 2
```
- So you see that both of them are reachable from both sides when the client is on my machine; Wireshark capture seems to ok too.
- For the client inside the VM, TCP and UDP ports are working but not SCTP. Wireshark show hearbeat messages then ABORT in the server interface. In the docker0 interface, INIT and INIT_ACK but no further messsage. This goes on until you stop your client.

#### With echo-sctp
```sh
docker run -it  -p 2000:2000/sctp --rm sofianinho/echo-sctp:scratch -server -port 2000 -ip 172.17.0.2
```

```sh
# client on my machine and VM
./example -port 2000 -ip 172.17.0.2
```
- Same conclusions: the pings from the client platform (metal or VM) to the server (container) work, but no SCTP message.
- Same from the wireshark capture: you see the pings, but the SCTP messages are only INIT and INIT_ACK

### Both in containers, the network is of type bridge
We create a docker network of type bridge

```sh
docker network create sctp --driver bridge
```

We run the containers attached to this network

#### With iperf3

```sh
# Server
docker run -it --rm --network sctp -p 5201:5201/sctp sofianinho/iperf3:3.6-ubuntu18.04 -s
```

```sh
# Client
docker run -it --rm --network sctp sofianinho/iperf3:3.6-ubuntu18.04 -c 172.18.0.2 -p 5201 --sctp  -i 1 -t 5 --nstreams 2
```
- Does not work and wireshark shows INIT and INIT_ACK messages but no traffic. Both TCP and UDP work fine.


#### With echo-sctp

```sh
# Server
docker run -it --network sctp -p 2000:2000/sctp --rm sofianinho/echo-sctp:scratch -server -port 2000 -ip 172.18.0.2
```

```Sh
# Client
docker run -it  --rm sofianinho/echo-sctp:scratch -port 2000 -ip 172.18.0.2
```
- The client exits with the following log:
```terminal
2018/08/17 14:24:58 Resolved address '172.18.0.2' to 172.18.0.2
2018/08/17 14:24:58 raw addr: [2 0 7 208 172 18 0 2 0 0 0 0 0 0 0 0]
2018/08/17 14:24:58 failed to dial: operation already in progress
```
The error " failed to dial: operation already in progress" is returned by function "DialSCTPExt" in the "github.com/ishidawataru/sctp" library (file sctp_linux.go). At the moment, I do not know why. The code works fine when run in the same machine (no containers) and as we see later in an `overlay` network.

### Both in containers, the network is of type overlay attachable

We create an attachable overlay network
```sh
docker network create sctp2 --driver overlay --attachable
```
#### With iperf3

```sh
# Server
docker run -it --rm --network sctp3 --ip 10.0.3.3 -p 5201:5201/sctp sofianinho/iperf3:3.6-ubuntu18.04 -s
```

```sh
# Client
docker run -it --rm --network sctp3 sofianinho/iperf3:3.6-ubuntu18.04 -c 10.0.3.3 -p 5201 --sctp --nstreams 4 --len 10k --parallel 4
```
- Does not work with SCTP as shown in the commands. Wireshark shows HEARTBEAT messages, no DATA. Both TCP and UDP work fine

#### With echo-sctp

```sh
#Server
docker run -it --network sctp2 --ip 10.0.3.3  -p 2000:2000/sctp --rm sofianinho/echo-sctp:scratch -server -port 2000 -ip 10.0.3.3
```
```sh
# Client
docker run -it --network sctp2 --ip 10.0.3.4  --rm sofianinho/echo-sctp:scratch -port 2000 -ip 10.0.3.3
```

- Works fine!


[1] Section Runtime of [changelog for docker 18.03.0](https://docs.docker.com/release-notes/docker-ce/#18030-ce-2018-03-21)
[2] Support SCTP port mapping (bump up API to v1.37) [pull 33922](https://github.com/moby/moby/pull/33922)
[3] Release notes from Docker [18.06.0-ce](https://github.com/docker/docker-ce/releases/tag/v18.06.0-ce)
[4] Docker iperf3 [repo](https://github.com/sofianinho/iperf3-docker)
[5] SCTP library for go [repo](https://github.com/ishidawataru/sctp)

