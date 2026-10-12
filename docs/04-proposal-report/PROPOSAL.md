# Edge Telemetry Gateway: Project Proposal

**Author:** Justin Mascarenhas  **Status:** proposal, pre-implementation  **Hardware:** BeagleBone Black, MPU6050, Jetson Orin Nano Super, laptop

## Abstract
I will build a small edge-telemetry system on embedded Linux. An MPU6050 motion sensor is wired over I2C to a BeagleBone Black (ARMv7). A C++ gateway daemon reads the sensor, packs each sample into a length-prefixed, CRC-32 checked binary frame, and streams it over TCP/IP from a single non-blocking `epoll` loop to several clients at once: a logger on a Jetson (wired Ethernet) and a live and load-test client on a laptop (Wi-Fi). The sensor is read first from userspace through `/dev/i2c-2`, and finally through my own out-of-tree Linux kernel character driver (C), bound by a device-tree overlay. The software ships as multi-architecture Docker images (amd64, arm64, arm/v7) through a GitHub Actions CI/CD pipeline that tests, publishes and deploys on every push.

## Motivation
Embedded roles ask for the same cluster of skills: C and C++ on Linux, kernel drivers and device trees, networking, containers and CI/CD. This project exercises each in one system small enough to understand end to end and explain without notes. Every claim in the final report will come from something I built and measured.

## Objectives
1. A framing protocol that survives TCP's byte-stream behavior: length prefix, CRC-32, sequence numbers, heartbeats.
2. An `epoll` server that serves many clients from one thread, with clients that reconnect with backoff and jitter.
3. Real sensor data on the BBB over I2C, first from userspace and then through my own kernel driver.
4. Reproducible multi-architecture builds and push-to-deploy delivery to the BBB.

## Methodology
Work proceeds in ordered steps, each tested on its own before the next begins:

| Stage | Work | Done when |
|---|---|---|
| Protocol | C++ frame encode/decode and stream reassembly, with GoogleTest | Tests pass under ASan and UBSan |
| Networking | Gateway with a simulated sensor; clients with RTT measurement, CSV logging and a load mode | 10,000 frames, zero CRC failures |
| Docker | Multi-stage image, Compose, `buildx` for three architectures | ARM images run under QEMU |
| BBB | MPU6050 via `/dev/i2c-2`, systemd service, Jetson and laptop clients, Wireshark verification | Live values track physical motion on both clients |
| CI/CD | GitHub Actions: build, test, publish, deploy to the BBB | A push alone updates the board |
| Driver | Kernel module in C plus device-tree overlay, exposing `/dev/mpu6050` | A cold boot streams live data |

The kernel driver is C because Linux has no C++ module support; the gateway and clients are C++. Data is streamed over TCP on a home LAN. It is not broadcast and not Internet-facing.

## Environment
The BBB and Jetson connect by Ethernet through a bridged switch; the laptop uses Wi-Fi; all share one `192.168.1.0/24` network. Development and tests run on the laptop first, then on the target board. A serial console through a Raspberry Pi Pico gives boot-time visibility on the headless BBB.

## Anticipated results
All numbers are filled in only after measurement.

| Metric | Result |
|---|---|
| Throughput at the sensor's maximum rate | ⟨X⟩ frames/s; limit: ⟨I2C read time / CPU / link⟩ |
| Round-trip time, wired vs Wi-Fi (p50 / p99) | ⟨X⟩ / ⟨X⟩ ms vs ⟨X⟩ / ⟨X⟩ ms; `TCP_NODELAY` on vs off |
| Concurrent clients on one epoll thread | ⟨N⟩ with zero drops |
| Integrity | Zero CRC failures across ⟨N⟩ frames |
| Cable-pull recovery | ⟨X⟩ s to first valid frame |
| Service restart (`kill -9`) | ⟨X⟩ s |
| Push to running on the BBB | ⟨X⟩ min |
| Image size per architecture | ⟨X⟩ / ⟨X⟩ / ⟨X⟩ MB |

## Risks and mitigations
- **Limited BBB capacity** (one 1 GHz core, 512 MB RAM): the load test finds the ceiling and records what limits it.
- **I2C read time** may cap the sample rate: measure it, report it, and do not claim more.
- **Wi-Fi jitter** will widen RTT: report wired and Wi-Fi separately.
- **Kernel module mismatch:** build against headers that match the running kernel.
- **Power stability:** the board runs on USB power from the Jetson; note and report any resets under load.

## Out of scope
An Ethernet MAC driver, LwIP, TLS, cloud services, multiple sensors, Yocto and CAN (reserved for later projects).

## Deliverables
Protocol library and tests, gateway and client, multi-arch images and CI/CD pipeline, kernel driver and overlay, and a final report with measurements and lessons learned.
