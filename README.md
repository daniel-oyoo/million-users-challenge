# The Million Users Challenge — High-Concurrency Java 21 / Spring Boot 3.2 Submission

> **Merged into [`arjaythedev/million-users-challenge`](https://github.com/arjaythedev/million-users-challenge) ([PR #86](https://github.com/arjaythedev/million-users-challenge/pull/86)) — Ranked  on the [Global Leaderboard](https://benchmarks.arjaythedev.com/million-users/leaderboard/java-springboot-danieloyoo)**

This fork contains my submission (`submissions/java-springboot-danieloyoo`) to **The Million Users Challenge**—a systems engineering benchmark testing how many concurrent active users a single **16-core, 32 GB RAM** server can sustain for 5 minutes with **zero caching**, **direct internet exposure** (no Nginx/reverse proxy), **strict per-request JWT verification**, and **synchronous PostgreSQL 18 commits**.

This entry sustained **390,000  users** with **0% errors** and **14,635 req/s** under strict tail-latency constraints (`p95 < 500 ms`, `p99 < 1 s`).

---

## Official Benchmark Results

All runs were executed on isolated bare-metal/VPS hardware (`Ubuntu 24.04 x86_64`, separate PostgreSQL 18 database host, distributed load generators) against a fresh copy of the seed database (`50k users`, `500k posts`, `2M likes`) and verified against all **42 automated compliance tests**.

| CPU Allocation | Concurrent Users Held (5 min) | Throughput | p95 Latency | p99 Latency | Error Rate | App CPU | Postgres CPU |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **16 Cores** | **340,000** | 12,972 req/s | 71 ms | 780 ms | **0.00%** | 42% | 28% |
| **8 Cores** | **390,000** | 14,635 req/s | 170 ms | 366 ms | **0.00%** | 77% | 33% |
| **4 Cores** | **200,000** | 8,045 req/s | 34 ms | 169 ms | **0.00%** | 76% | 16% |

> **Leaderboard Ranking: https://benchmarks.arjaythedev.com/million-users/leaderboard/java-springboot-danieloyoo**

---

## Engineering & Performance Architecture

To sustain hundreds of thousands of simultaneous keep-alive connections across two listening ports (`80` and `81`) without caching or reverse proxies, the service was engineered around four core optimizations:

1. **Java 21 Virtual Threads (`Project Loom`)**
   * Enabled `spring.threads.virtual.enabled=true` with a shared virtual-thread executor across both embedded Tomcat connectors, eliminating OS platform-thread exhaustion across hundreds of thousands of long-lived idle keep-alive sockets.
2. **Dual-Port Embedded Tomcat Socket Tuning**
   * Removed default connector caps on both the primary port (`80`) and secondary `EXTRA_PORT` (`81`), configuring unified `max-connections`, `accept-count`, and `90s` keep-alive timeouts so burst traffic and idle load-generator pools never hit silent socket queuing or timeouts.
   * Diagnosed and eliminated an initial 17k-user connection bottleneck caused by asymmetric connector defaults on port `81` (raising capacity **22.9×** from 17,000 to 390,000 users).
3. **Zero-Overhead Data Access & Single-Query Semantics**
   * Bypassed ORM/Hibernate overhead in favor of raw parameterized SQL via Spring `JdbcTemplate` mapped directly to the specification queries.
   * Leveraged PostgreSQL constraint semantics (`INSERT ... ON CONFLICT DO NOTHING` + `SQLSTATE 23503` foreign-key exception mapping) to handle idempotent likes and `404` validation in a single database round-trip.
   * Tuned HikariCP connection pooling (`128` connections) within the 512-connection ceiling to balance throughput and tail latency without exhausting database slots.
4. **Strict Spec Compliance & Container-Aware JVM Sizing**
   * Enforced per-request HS256 JWT signature and mandatory `exp` claim verification (`jjwt`), exact status-code precedence (`400` / `401` / `404` / `200` / `201`), and strict JSON type validation.
   * Configured `-XX:MaxRAMPercentage=70` to maximize heap headroom for socket buffers and connection state inside the 32 GB memory envelope without OOM risk.

---

## Repository Structure

| Path | Description |
| :--- | :--- |
| [`submissions/java-springboot-danieloyoo/`](submissions/java-springboot-danieloyoo) | **My merged submission**: Spring Boot 3.2.3 / Java 21 source, `install.sh`, `build.sh`, `start.sh`, and technical write-up |
| [`SPEC.md`](SPEC.md) | API contract, 5 endpoints, exact JSON payloads, error precedence, and allowed SQL queries |
| [`schema.sql`](schema.sql) | Fixed PostgreSQL 18 database schema |
| [`seed/make-seed.sh`](seed/make-seed.sh) | Seed generator (`50k` users, `500k` posts, `2M` likes, and `seed/tokens.json`) |
| [`test/run.sh`](test/run.sh) | End-to-end test runner executing all 42 compliance checks against a fresh database |
| [`bench/`](bench) | Distributed load generator and benchmarking harness |

---

## Running My Submission Locally

```bash
# 1. Generate the seed database (requires Node >= 18 and PostgreSQL >= 16)
bash seed/make-seed.sh

# 2. Build, launch on a fresh database copy, and run all 42 compliance checks
bash test/run.sh submissions/java-springboot-danieloyoo
