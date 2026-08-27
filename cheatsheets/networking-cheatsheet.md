# Networking & TLS Cheatsheet

A quick-reference guide to network protocol layers, TLS handshake mechanics, OpenSSL commands, and load balancing.

---

## 1. The OSI & Internet Protocol Stack

```
Layer 7 (Application) : HTTP, HTTPS, gRPC, DNS, SSH, WebSocket
Layer 4 (Transport)   : TCP (Reliable, byte-stream), UDP (Fast, datagram), QUIC
Layer 3 (Network)     : IP (IPv4, IPv6), ICMP, Routing / BGP
Layer 2 (Data Link)   : Ethernet, MAC Addressing, ARP
Layer 1 (Physical)    : Fiber, Copper, Radio Waves
```

---

## 2. HTTP Protocol Evolution

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
| :--- | :--- | :--- | :--- |
| **Transport** | TCP (Port 80/443) | TCP (Port 443) | **QUIC over UDP** (Port 443) |
| **Data Format** | Text / ASCII | Binary Framing | Binary Framing |
| **Multiplexing** | No (Head-of-line blocking) | **Yes** (Over single TCP stream) | **Yes** (Independent QUIC streams) |
| **Header Compression** | None | HPACK | QPACK |
| **TCP HoL Blocking** | Yes | Yes (on packet loss) | **Completely Eliminated** |
| **Handshake Latency** | 1 RTT TCP + 1-2 RTT TLS | 1 RTT TCP + 1 RTT TLS | **0-1 RTT (Integrated Crypto)** |

---

## 3. Essential OpenSSL Commands for Backend Engineers

```bash
# 1. Test TLS Handshake and print negotiated cipher & cert chain
openssl s_client -connect api.example.com:443 -servername api.example.com

# 2. Test mTLS connection with client certificate and private key
openssl s_client -connect internal.service:8443 \
  -cert client.crt -key client.key -CAfile rootCA.crt

# 3. Check SSL certificate expiration date
openssl x509 -enddate -noout -in /etc/ssl/cert.pem

# 4. Decode full X.509 certificate details (SAN, Issuer, Subject)
openssl x509 -text -noout -in /etc/ssl/cert.pem

# 5. Verify certificate matches private key (Modulus check)
openssl x509 -noout -modulus -in cert.pem | openssl md5
openssl rsa -noout -modulus -in key.pem | openssl md5
# (Both MD5 hashes must match exactly!)
```

---

## 4. Layer 4 vs Layer 7 Load Balancing

| Feature | Layer 4 (NLB / IPVS) | Layer 7 (ALB / Envoy / NGINX) |
| :--- | :--- | :--- |
| **Inspection Level** | IP & TCP/UDP Ports | HTTP Headers, Cookies, Paths, Methods |
| **TLS Handling** | TLS Passthrough | **TLS Termination** |
| **Throughput** | Ultra High ($>1\text{M}$ req/s) | High ($50\text{k}-200\text{k}$ req/s) |
| **Routing Features** | Simple round-robin / least conn | Path routing (`/api/v1/*`), canary weighting, WAF |
