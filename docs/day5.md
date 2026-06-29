## Day 5 — Tracing Network Traffic with tcpdump

This session uses `tcpdump` to observe how HTTP traffic flows from your local machine, through the OpenShift router, and into a pod running on the cluster.

### Prerequisites

- A running OpenShift CRC cluster with the `day4-nginx` application deployed (from [Day 4](./day4.md))
- The application is accessible at `http://day4-nginx.apps-crc.testing/`
- SSH access to the Ubuntu VM hosting CRC

### Running tcpdump

Run `tcpdump` on the Ubuntu VM's primary interface (`ens18`), filtering for traffic on port 80:

```sh
sudo tcpdump -i ens18 port 80
```

### Captured Output

While `tcpdump` is listening, access the nginx application from your browser or using `curl`. The following packets were captured:

```sh
21:17:37.998585 IP 192.168.1.115.50181 > gopi-ubuntu.http: Flags [P.], seq 67400794:67401438, ack 2917029769, win 2053, options [nop,nop,TS val 3319377132 ecr 521896343], length 644: HTTP: GET / HTTP/1.1
21:17:38.001600 IP gopi-ubuntu.http > 192.168.1.115.50181: Flags [P.], seq 1:179, ack 644, win 501, options [nop,nop,TS val 521932350 ecr 3319377132], length 178: HTTP: HTTP/1.1 304 Not Modified
21:17:38.004031 IP 192.168.1.115.50181 > gopi-ubuntu.http: Flags [.], ack 179, win 2051, options [nop,nop,TS val 3319377139 ecr 521932350], length 0
21:17:57.071237 IP 192.168.1.115.50180 > gopi-ubuntu.http: Flags [.], ack 2451699598, win 2055, length 0
```

### Packet-by-Packet Analysis

| # | Direction | Summary |
|---|-----------|---------|
| 1 | `192.168.1.115:50181 → gopi-ubuntu:80` | **HTTP GET request** – Your local machine sends a request for `/`. The `[P.]` (Push) flag indicates data is being sent. |
| 2 | `gopi-ubuntu:80 → 192.168.1.115:50181` | **HTTP 304 Not Modified** – The nginx pod responds with a cached response. The router forwards this back to your machine. |
| 3 | `192.168.1.115:50181 → gopi-ubuntu:80` | **ACK** – Your machine acknowledges receiving the response. No data (`length 0`). |
| 4 | `192.168.1.115:50180 → gopi-ubuntu:80` | **ACK** – A separate connection (port `50180`) acknowledges data, likely a keep-alive signal. |

### Understanding the Traffic Flow

1. **Your local machine** (`192.168.1.115`) sends an HTTP request to the hostname `day4-nginx.apps-crc.testing`.
2. **DNS resolves** the hostname to the IP of the Ubuntu VM (`gopi-ubuntu`, which runs CRC).
3. **OpenShift router** (running on the CRC cluster) receives the request on port 80 and forwards it to the `nginx-example` service.
4. **The service** routes the traffic to the actual nginx pod.
5. The **pod's response** follows the reverse path back to your machine.

The `tcpdump` capture confirms that all traffic enters the Ubuntu VM on port 80, where the OpenShift router is listening.

### Key Takeaway

Even though the application runs inside an OpenShift pod with its own internal IP, from the network's perspective, all traffic enters through a single entry point — the OpenShift router on the host VM. This is the essence of how OpenShift routes external traffic to pods.