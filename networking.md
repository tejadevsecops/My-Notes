# B2 — Networking Recap

Same training server as B1, but now our banking platform is running on it
(a Kubernetes cluster with ~70 pods — module B3 explains how). That gives us
something real to point curl and dig at. All outputs below are genuine.

The portal answers on port 80 of the server:

```
$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost/
200
```

---

## 1. OSI — the three layers you actually use

Forget memorising all seven. Day to day you think in three:

- **L7 (application)** — HTTP requests, status codes, headers, URLs. "The
  request failed."
- **L4 (transport)** — TCP connections and ports. "The connection failed."
- **L3 (network)** — IP routing. "The packet never arrived."

Most of your troubleshooting is deciding which of those three sentences is
true. The tools split the same way: `curl` for L7, `ss`/`telnet` for L4,
`ping`/`mtr` for L3.

---

## 2. HTTP status codes

### 2.1 The table

| Range | Meaning              | Common ones you'll meet                          |
|-------|----------------------|--------------------------------------------------|
| 2xx   | success              | 200 OK, 201 Created, 204 No Content              |
| 3xx   | redirection          | 301/302 redirect, 304 Not Modified (cache)       |
| 4xx   | **client** was wrong | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests |
| 5xx   | **server** was wrong | 500 Internal Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout |

Live from our portal — three requests, three different classes:

```
$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost/           # normal page
200
$ curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost/api/upi/pay -d '{}'
401                                                                    # not logged in — OUR fault
$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost/api/does-not-exist
404                                                                    # wrong URL — OUR fault
```

### 2.2 502 vs 503 vs 504 — learn this cold

All three come from something standing **in front** of the real server (load
balancer, reverse proxy, API gateway). They tell you *how* the thing behind
it failed:

- **502 Bad Gateway** — proxy reached the backend, got garbage or a slammed
  connection. Backend is up-ish but broken (crashing mid-request, wrong port).
- **503 Service Unavailable** — nothing healthy to send to. Every backend is
  down or failing health checks. Also what apps return when overloaded.
- **504 Gateway Timeout** — backend accepted the request and then never
  answered within the proxy's timeout. Backend is alive but too slow
  (usually stuck on a database or another service).

Rough translation: 502 = broken backend, 503 = no backend, 504 = slow
backend.

### 2.3 4xx vs 5xx from an alerting point of view

This distinction runs the whole course, so settle it now:

- **5xx means we are broken.** Page someone.
- **4xx means the caller did something odd.** A burst of 404s is usually a
  bot scanning URLs; a burst of 401s might be a client with an expired token.
  You *watch* 4xx rates (a sudden jump in 400s after a deploy = you broke an
  API contract), but you do not page at 3 am for them.

Alert on the 5xx **rate**, observe the 4xx **trend**.

---

## 3. Life of one request

When you type a URL, five things happen in order, and each one can be slow:

1. **DNS** — name → IP
2. **TCP handshake** — SYN / SYN-ACK / ACK (one round trip)
3. **TLS handshake** — certificates and keys (one or two more round trips, https only)
4. **Server processing** — the app does its work ← usually the biggest chunk
5. **Transfer** — response bytes travel back

curl can time each phase. This is the single most useful command in this
module — save it somewhere:

```
$ curl -s -o /dev/null -w "dns=%{time_namelookup}s tcp=%{time_connect}s tls=%{time_appconnect}s ttfb=%{time_starttransfer}s total=%{time_total}s\n" http://localhost/
dns=0.000314s  tcp=0.000431s  tls=0.000000s  ttfb=0.013698s  total=0.014549s
```

Reading it: DNS and TCP are effectively free (we are on the same box), tls is
0 because this is plain http, and almost the whole 14 ms is **ttfb** (time to
first byte) — i.e. the server thinking. On a real internet request you'd see
tens of ms in dns/tcp/tls too. When a user says "the site is slow", this
command tells you *which phase* is slow, and each phase points at a different
team.

### Connection reuse — keep-alive and pooling

Handshakes cost round trips, so nobody pays them per request. HTTP/1.1 keeps
the TCP connection open and reuses it:

```
$ curl -sv -o /dev/null http://localhost/ 2>&1 | grep -iE "Connected|< HTTP|< connection"
* Connected to localhost (127.0.0.1) port 80 (#0)
< HTTP/1.1 200 OK
< Connection: keep-alive
```

Applications do the same with databases — a **connection pool** holds e.g. 10
open Oracle connections and requests borrow one. Remember the TIME_WAIT pile
from B1? An app that *doesn't* reuse connections generates one TIME_WAIT per
request. When you see tens of thousands of them, someone forgot pooling.

# Understanding Connection Pooling and TIME_WAIT

Applications often communicate with databases using TCP connections. Creating a new database connection for every request is expensive and can lead to a large number of `TIME_WAIT` connections.

---

## Without Connection Pooling

For every incoming request, the application:

1. Opens a new database connection.
2. Executes a query.
3. Closes the database connection.

### Example

```text
Request 1
  -> Open Oracle connection
  -> Run query
  -> Close connection

Request 2
  -> Open Oracle connection
  -> Run query
  -> Close connection

Request 3
  -> Open Oracle connection
  -> Run query
  -> Close connection
```

If the application receives 10,000 requests:

```text
10,000 connections opened
10,000 connections closed
```

Each closed TCP connection enters the `TIME_WAIT` state for a period of time.

As a result, you may observe:

```text
15000 TIME-WAIT
```

or

```text
50000 TIME-WAIT
```

This is called **connection churn**.

---

## With Connection Pooling

Instead of creating and destroying connections for every request, the application maintains a pool of reusable connections.

### Example Pool

```text
Connection 1
Connection 2
Connection 3
...
Connection 10
```

When a request arrives:

```text
Request 1
  -> Borrow Connection 3
  -> Run query
  -> Return Connection 3 to pool

Request 2
  -> Borrow Connection 7
  -> Run query
  -> Return Connection 7 to pool
```

Notice that connections are **reused** rather than closed.

### Result

```text
10 connections opened
10 connections remain available
Thousands of requests served
```

Very few new `TIME_WAIT` sockets are created.

---

## Visual Comparison

### Without Pooling

```text
User Request
      |
      v
Open DB Connection
      |
      v
Run Query
      |
      v
Close Connection
      |
      v
TIME_WAIT
```

For 1,000 requests:

```text
1,000 TIME_WAIT sockets
```

---

### With Pooling

```text
           +----------------+
Request -->| Existing Pool  |
           +----------------+
                   |
                   v
             Reuse Connection
                   |
                   v
              Return to Pool
```

For 1,000 requests:

```text
Pool Size: 10

Thousands of requests served
Only 10 active connections maintained
Very few TIME_WAIT sockets
```

---

## Why Does TIME_WAIT Exist?

When a TCP connection is closed, the operating system does not immediately remove it.

Instead, it places the connection into the `TIME_WAIT` state to:

- Ensure delayed packets are discarded.
- Prevent duplicate packets from affecting future connections.
- Allow TCP to shut down cleanly.

Therefore, a moderate number of `TIME_WAIT` connections is completely normal.

---

## Real-World Example

Suppose an application handles:

```text
100 requests per second
```

### Without Pooling

```text
100 DB connections/sec opened
100 DB connections/sec closed
```

After several minutes:

```text
20000+ TIME_WAIT
```

---

### With Pooling

```text
Connection Pool Size = 20
```

The same 20 connections are reused:

```text
20 ESTAB
Very few TIME_WAIT
```

This is much more efficient and reduces resource consumption.

---

## What to Look For During Troubleshooting

Check TCP connection states:

```bash
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn
```

Example output:

```text
50000 TIME-WAIT
200 ESTAB
```

This is often a red flag.

Questions to ask:

1. Is the application creating a new connection for every request?
2. Is database connection pooling enabled?
3. Is HTTP keep-alive disabled?
4. Are clients repeatedly reconnecting?

---

## Rule of Thumb

✅ A few hundred `TIME_WAIT` connections on a busy server is usually normal.

⚠️ Thousands to tens of thousands of `TIME_WAIT` connections may indicate excessive connection creation and teardown.

🚨 If every request creates a new database or HTTP connection, investigate connection pooling configuration.

### Key Takeaway

> Connection pooling improves performance by reusing existing connections. A large and continuously growing number of `TIME_WAIT` connections often indicates that connections are being opened and closed too frequently instead of being reused.

---

## 4. DNS

### 4.1 TTL and caching

Every DNS answer carries a TTL — "you may cache me this many seconds". Ask
twice and watch it count down (166 → 163 three seconds later):

```
$ dig amazon.com | grep -m1 "^amazon"
amazon.com.		166	IN	A	98.87.170.71
$ dig amazon.com | grep -m1 "^amazon"
amazon.com.		163	IN	A	98.82.161.185
```

TTL is why DNS changes "take time to propagate" — caches everywhere hold the
old answer until their TTL runs out.

# Understanding DNS TTL (Time To Live)

Every DNS record includes a **TTL (Time To Live)** value.

TTL tells DNS resolvers:

> "You may cache this DNS answer for this many seconds before requesting a fresh copy."

---

## Example

### First Query

```bash
dig amazon.com | grep -m1 "^amazon"
```

Output:

```text
amazon.com.    166    IN    A    98.87.170.71
```

### Second Query (3 Seconds Later)

```bash
dig amazon.com | grep -m1 "^amazon"
```

Output:

```text
amazon.com.    163    IN    A    98.82.161.185
```

---

## Breaking Down the Output

```text
amazon.com.    166    IN    A    98.87.170.71
```

| Field | Meaning |
|---------|---------|
| `amazon.com.` | Domain name |
| `166` | TTL (Time To Live) in seconds |
| `IN` | Internet record class |
| `A` | IPv4 address record |
| `98.87.170.71` | IP address returned by DNS |

The most important field here is:

```text
166
```

This means:

> The DNS response may be cached for another 166 seconds.

---

## Why Does the TTL Decrease?

TTL acts like a countdown timer.

### Initial Response

```text
TTL = 166
```

### Three Seconds Later

```text
TTL = 163
```

The value decreases because time has passed:

```text
166
165
164
163
...
0
```

When the TTL reaches:

```text
0
```

the cached DNS record expires and a new DNS lookup must be performed.

---

## Visual Example

### Time = 10:00:00

```text
amazon.com -> 98.87.170.71
TTL = 166
```

### Time = 10:00:03

```text
amazon.com -> 98.87.170.71
TTL = 163
```

The answer is still considered valid because the TTL has not yet expired.

---

## Why Did the IP Address Change?

In the example, the returned IP address changed:

```text
amazon.com. 166 IN A 98.87.170.71
```

to

```text
amazon.com. 163 IN A 98.82.161.185
```

This is normal for large websites.

Companies like Amazon use:

- Multiple servers
- Multiple IP addresses
- Load balancing
- Geographic routing
- Content delivery networks (CDNs)

For example:

```text
amazon.com
├── 98.87.170.71
├── 98.82.161.185
├── 54.x.x.x
└── many more...
```

DNS may return any of these valid addresses depending on location, load, and routing policies.

---

## Why TTL Exists

Without DNS caching:

```text
Browser
   |
   v
DNS Query
   |
   v
DNS Server
```

A DNS lookup would occur for every request.

With caching:

```text
First DNS Query
       |
       v
Cache Result
       |
       v
Reuse Cached Result Until TTL Expires
```

Benefits:

- Faster application performance
- Reduced DNS traffic
- Lower load on DNS infrastructure

---

## Short vs Long TTL

### Short TTL

```text
60 seconds
```

Advantages:

✅ DNS changes propagate quickly

Disadvantages:

❌ More DNS queries

---

### Long TTL

```text
86400 seconds (1 day)
```

Advantages:

✅ Fewer DNS queries

✅ Better caching efficiency

Disadvantages:

❌ DNS changes take longer to propagate

---

## Key Takeaway

When you see:

```text
amazon.com.    166    IN    A    98.87.170.71
```

the value:

```text
166
```

is the **TTL (Time To Live)**.

It indicates how many seconds the DNS response may remain cached before another DNS lookup is required.

As time passes, the TTL counts down:

```text
166 → 163 → 160 → ... → 0
```

Once the TTL reaches **0**, the cached record expires and a fresh DNS query is performed.

### 4.2 Local resolver vs public resolver

Compare who answered — the `SERVER` line:

```
$ dig amazon.com | grep -E "Query time|SERVER"
;; Query time: 5 msec
;; SERVER: 172.31.0.2#53(172.31.0.2)        ← the AWS VPC resolver

$ dig @8.8.8.8 amazon.com | grep -E "Query time|SERVER"
;; Query time: 1 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)              ← Google public DNS
```

Why compare? If the local resolver returns something different (or nothing)
while 8.8.8.8 answers fine, the problem is your resolver, not the domain.
That's a 30-second check that ends many "is DNS down?" debates.

## Compare DNS Resolvers Using the `SERVER` Line

### Default Resolver

```bash
dig amazon.com | grep -E "Query time|SERVER"
```

Example:

```text
;; Query time: 5 msec
;; SERVER: 172.31.0.2#53(172.31.0.2)
```

### Google Public DNS

```bash
dig @8.8.8.8 amazon.com | grep -E "Query time|SERVER"
```

Example:

```text
;; Query time: 1 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
```

### What the `SERVER` Line Shows

The `SERVER` line identifies which DNS server answered your query.

- `172.31.0.2` → Local/AWS VPC DNS Resolver
- `8.8.8.8` → Google Public DNS

### Why Compare?

If your default resolver fails but Google DNS returns a valid answer:

```text
Local Resolver  -> Fails
Google DNS      -> Works
```

✅ The domain is fine.

❌ The problem is with your local DNS resolver.

### Quick Troubleshooting Tip

Run both commands and compare:

- Did both return an answer?
- Did both return the same IP address?
- Which DNS server responded?
- What was the query time?

### Key Takeaway

> Comparing your local DNS resolver with a known-good resolver such as `8.8.8.8` is a fast way to determine whether a DNS issue is caused by the domain itself or by your local DNS infrastructure.

### 4.3 CoreDNS in Kubernetes

Inside the cluster, pods use **CoreDNS** (note the server 10.96.0.10 — a
cluster IP, not the VPC resolver). Services get automatic names. From inside
the portal pod:

```
$ kubectl -n bankobs exec deploy/web-portal -- nslookup kafka
Server:   10.96.0.10
Name:     kafka.bankobs.svc.cluster.local
Address:  10.96.71.63
```

Two things to notice:

- We asked for just `kafka` and got `kafka.bankobs.svc.cluster.local` — the
  pod's search domain expanded it. That's why our app configs can simply say
  `kafka:9092` or `oracle:1521`.
- The full name pattern is `<service>.<namespace>.svc.cluster.local`. From
  another namespace you'd write `kafka.bankobs`.


## Understanding CoreDNS in Kubernetes

### Check DNS Resolution Inside a Pod

```bash
kubectl -n bankobs exec deploy/web-portal -- nslookup kafka
```

Example Output:

```text
Server:   10.96.0.10
Name:     kafka.bankobs.svc.cluster.local
Address:  10.96.71.63
```

### Key Points

#### 1. CoreDNS Handles Internal DNS

```text
Server: 10.96.0.10
```

This is the Kubernetes **CoreDNS** service, which resolves Kubernetes Services and Pods.

---

#### 2. Short Names Are Automatically Expanded

Requested:

```text
kafka
```

Resolved to:

```text
kafka.bankobs.svc.cluster.local
```

Kubernetes uses DNS search domains, allowing applications to use simple hostnames:

```text
kafka:9092
oracle:1521
redis:6379
```

instead of full DNS names.

---

#### 3. Kubernetes Service DNS Format

```text
<service>.<namespace>.svc.cluster.local
```

Example:

```text
kafka.bankobs.svc.cluster.local
```

Where:

```text
kafka     = Service name
bankobs   = Namespace
svc       = Service domain
cluster.local = Cluster DNS domain
```

---

### Cross-Namespace Access

From the same namespace:

```text
kafka
```

From a different namespace:

```text
kafka.bankobs
```

or

```text
kafka.bankobs.svc.cluster.local
```

---

### Key Takeaway

> CoreDNS provides automatic DNS resolution for Kubernetes Services. Pods in the same namespace can typically use short names like `kafka`, while the full DNS name follows the format `<service>.<namespace>.svc.cluster.local`.

---

## 5. Latency: percentiles, not averages

Three words that get mixed up:

- **Latency** — how long one request takes.
- **Throughput** — how many requests per second you handle.
- **Tail latency** — how long the *slowest* requests take.

We measure latency with percentiles. p95 = "95% of requests were faster than
this". Here are 20 real requests against our portal:

```
$ for i in $(seq 1 20); do curl -s -o /dev/null -w "%{time_total}\n" http://localhost/; done | sort -n > lat.txt
$ awk '{a[NR]=$1; s+=$1} END {printf "p50=%.3fs p95=%.3fs max=%.3fs avg=%.3fs\n", a[int(NR*.5)], a[int(NR*.95)], a[NR], s/NR}' lat.txt
p50=0.013s  p95=0.015s  max=0.015s  avg=0.013s
```

Healthy service: p50 and p95 close together. Now imagine 19 requests at
13 ms and one at 2 seconds. The **average** is ~112 ms — looks "a bit slow".
The p95 screams. Averages hide the users who suffered; that's why every SLO
you will ever write uses percentiles (p95, p99), never averages.

One more trap: a customer action often fans out to many services (in our app
one UPI payment touches 5+). If each has a slow p99, the chance a customer
hits at least one slow hop grows fast. Tail latency compounds.

## Understanding Latency, Throughput, and Tail Latency

### Key Terms

- **Latency**: Time taken for a single request to complete.
- **Throughput**: Number of requests processed per second.
- **Tail Latency**: How long the slowest requests take.

---

## Why Percentiles Matter

Latency is typically measured using **percentiles**:

- **p50** (Median): 50% of requests were faster than this value.
- **p95**: 95% of requests were faster than this value.
- **p99**: 99% of requests were faster than this value.

Example:

```text
p50 = 13 ms
p95 = 15 ms
max = 15 ms
avg = 13 ms
```

✅ Healthy service: p50 and p95 are very close.

---

## Why Averages Can Be Misleading

Suppose:

```text
19 requests = 13 ms
1 request  = 2000 ms
```

Average:

```text
avg ≈ 112 ms
```

This doesn't reveal that one user waited **2 seconds**.

Percentiles expose these slow requests, while averages often hide them.

---

## Healthy vs Unhealthy

### Healthy

```text
p50 = 13 ms
p95 = 15 ms
p99 = 18 ms
```

Most users experience similar response times.

### Unhealthy

```text
p50 = 13 ms
p95 = 200 ms
p99 = 2 s
```

Most users are fast, but some users experience significant delays.

---

## Tail Latency in Microservices

A single customer action may call multiple services:

```text
Portal
  ↓
Kafka
  ↓
Fraud Service
  ↓
Oracle DB
  ↓
Notification Service
```

Even if each service is slow only 1% of the time (`p99`), the probability of hitting at least one slow component increases as more services are involved.

This is known as:

> **Tail latency compounds.**

---

## Key Takeaway

> **Averages show overall performance, but percentiles show user experience.**
>
> That's why production SLOs and SLAs are typically based on **p95** and **p99** latency rather than average latency.

---

## 6. Network failure modes

Three ways a connection fails, and they mean different things. All three
reproduced live:

**Connection refused** — the machine answered "nothing listens on that port"
(TCP RST). Fast failure. The host is up; the service is not:

```
$ curl http://localhost:9999/
curl: (7) Failed to connect to localhost port 9999: Connection refused
```

**Connection timeout** — no answer at all. Packets go into a void: wrong IP,
firewall/security group silently dropping, or host down. Note it burns the
full timeout — refused fails in milliseconds, timeout wastes seconds:

```
$ time curl -m 3 http://10.255.255.1/
curl: (28) Connection timed out after 3001 milliseconds
real    0m3.030s
```

Refused = "up but not listening". Timeout = "unreachable or filtered". That
one distinction points you at the right team (app team vs network/firewall).

**Connection reset** — it worked, then died mid-conversation (RST during
transfer). Typical causes: the process crashed, a proxy killed an idle
connection, or a **NAT gateway dropped the mapping**. NAT devices forget
quiet connections after ~350 s; the next packet on that "connection" gets a
reset. Classic symptom: long-idle database connections dying overnight. Fix
is TCP keepalives or shorter pool idle timeouts.

---

## 7. Load balancers

**L4 LB** — balances TCP connections. Sees IPs and ports, never looks inside.
Dumb, fast. **L7 LB** — speaks HTTP: routes by path/host, retries, returns
those 502/503/504s from section 2.

Our own stack has both flavours in miniature: the server forwards port 80
into the cluster at TCP level (L4 — it neither knows nor cares that it's
HTTP), and inside, `gateway-service` routes `/api/upi/...` to upi-service,
`/api/loans/...` to loan services and so on (L7).

**Load balancer metrics as a source of truth.** The LB sees every request,
every backend answer, every timeout — from the *customer's side*. When app
dashboards and users disagree ("app looks fine" / "site is down"), LB metrics
settle the argument: 5xx count, backend response time, healthy-host count.
In big incidents, check the edge first.

---

## 8. Hands-on

Run these on the training server.

1. The curl timing template against `http://localhost/`, then against
   `https://www.google.com`. Compare where the time goes (look at dns, tls).
2. `mtr -r -c 3 -n 8.8.8.8` — real path out of AWS:

```
HOST: ip-172-31-24-168            Loss%   Snt   Last   Avg  Best  Wrst
  1.|-- 240.64.131.128             0.0%     3    1.0   1.1   0.9   1.3
  2.|-- 100.100.4.82               0.0%     3    1.1   1.4   1.1   1.7
  3.|-- 99.83.65.1                 0.0%     3    1.1   1.2   1.1   1.2
```

   mtr = traceroute that keeps measuring. Loss% at the final hop matters;
   loss at a middle hop that disappears later is just a router de-prioritising
   pings — a classic false alarm.
3. Reproduce connection refused (`curl localhost:9999`) and timeout
   (`curl -m 3 http://10.255.255.1/`). Check the exit codes (`echo $?`) —
   7 vs 28. Scripts use these.
4. `ss -tan | awk 'NR>1{print $1}' | sort | uniq -c` before and after
   hammering the portal with 50 curls. Watch TIME-WAIT grow, wait a minute,
   watch it drain.
5. DNS drill: `dig amazon.com`, note TTL and SERVER; repeat with `@8.8.8.8`;
   then the CoreDNS lookups from section 4.3.
6. Log in to the portal API and check a 401 turns into a 200:

```
curl -s -c /tmp/c.txt -X POST http://localhost/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"customer_id":"CUST-00000001","password":"Training@123"}'
```
