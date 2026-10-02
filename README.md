# หลักสูตร Erlang ครบวงจร: จากพื้นฐานสู่ระดับโลก

> **Erlang Programming Course: From Zero to World-Class**
> ระบบ Distributed ที่เสถียรที่สุดในโลก — สร้างระบบที่ทนทานต่อความผิดพลาด 99.9999999%

---

## เกี่ยวกับหลักสูตรนี้

หลักสูตรนี้ออกแบบมาเพื่อพาคุณจากศูนย์สู่ระดับมืออาชีพในการเขียนโปรแกรมด้วย Erlang โดยครอบคลุมทุกด้าน ตั้งแต่พื้นฐานการติดตั้ง จนถึงการสร้างระบบ distributed ที่ใช้งานได้จริงในระดับ production

### ทำไมต้องเรียน Erlang?

- **99.9999999% uptime** — WhatsApp, RabbitMQ, CouchDB ใช้ Erlang
- **Concurrency ดีที่สุด** — spawning millions of lightweight processes
- **Fault-tolerance built-in** — "Let it crash" philosophy
- **Hot code swapping** — อัพเดทระบบโดยไม่ต้อง downtime
- **Distributed by design** — ออกแบบมาสำหรับ distributed systems ตั้งแต่ต้น

---

## โครงสร้างหลักสูตร (100+ Parts)

### ระดับที่ 1: พื้นฐาน (Parts 01-20)
| Part | หัวข้อ | สถานะ |
|------|--------|-------|
| [Part 01](part01/README.md) | บทนำ Erlang และการติดตั้ง | ✅ |
| [Part 02](part02/README.md) | ประเภทข้อมูลพื้นฐานและ Syntax | ✅ |
| [Part 03](part03/README.md) | Pattern Matching | ✅ |
| [Part 04](part04/README.md) | Functions และ Modules | ✅ |
| [Part 05](part05/README.md) | Lists และ Recursion | ✅ |
| [Part 06](part06/README.md) | Tuples, Maps และ Records | ✅ |
| [Part 07](part07/README.md) | Control Flow: case, if, guards | ✅ |
| [Part 08](part08/README.md) | String และ Binary | ✅ |
| [Part 09](part09/README.md) | I/O และ File Operations | ✅ |
| [Part 10](part10/README.md) | Error Handling พื้นฐาน | ✅ |

### ระดับที่ 2: Concurrency (Parts 11-30)
| Part | หัวข้อ | สถานะ |
|------|--------|-------|
| Part 11 | Processes และ spawn | 🔄 |
| Part 12 | Message Passing | 🔄 |
| Part 13 | receive และ selective receive | 🔄 |
| Part 14 | Process Links และ Monitors | 🔄 |
| Part 15 | GenServer พื้นฐาน | 🔄 |
| Part 16 | GenServer ขั้นสูง | 🔄 |
| Part 17 | Supervisor และ Supervision Trees | 🔄 |
| Part 18 | OTP Application | 🔄 |
| Part 19 | ETS: Erlang Term Storage | 🔄 |
| Part 20 | DETS: Disk-based Storage | 🔄 |

### ระดับที่ 3: OTP Framework (Parts 21-40)
| Part | หัวข้อ | สถานะ |
|------|--------|-------|
| Part 21 | gen_statem: State Machines | 🔄 |
| Part 22 | gen_event: Event Handlers | 🔄 |
| Part 23 | gen_tcp: TCP Networking | 🔄 |
| Part 24 | gen_udp: UDP Networking | 🔄 |
| Part 25 | ssl: Secure Connections | 🔄 |
| Part 26 | Mnesia Database | 🔄 |
| Part 27 | Mnesia ขั้นสูง | 🔄 |
| Part 28 | Release Management | 🔄 |
| Part 29 | Hot Code Upgrades | 🔄 |
| Part 30 | Logging และ Tracing | 🔄 |

### ระดับที่ 4: Distributed Systems (Parts 31-50)
| Part | หัวข้อ | สถานะ |
|------|--------|-------|
| Part 31 | Distributed Erlang Nodes | 🔄 |
| Part 32 | Node Discovery และ Clustering | 🔄 |
| Part 33 | Global Process Registry | 🔄 |
| Part 34 | Distributed Transactions | 🔄 |
| Part 35 | CAP Theorem ใน Erlang | 🔄 |
| Part 36 | Load Balancing | 🔄 |
| Part 37 | Consistent Hashing | 🔄 |
| Part 38 | Leader Election | 🔄 |
| Part 39 | Distributed Data Structures | 🔄 |
| Part 40 | Network Partitions Handling | 🔄 |

### ระดับที่ 5: Web Development (Parts 41-60)
| Part | หัวข้อ | สถานะ |
|------|--------|-------|
| Part 41 | Cowboy Web Server | 🔄 |
| Part 42 | REST API ด้วย Cowboy | 🔄 |
| Part 43 | WebSockets | 🔄 |
| Part 44 | HTTP Client ด้วย httpc | 🔄 |
| Part 45 | JSON Processing | 🔄 |
| Part 46 | Authentication & Authorization | 🔄 |
| Part 47 | Session Management | 🔄 |
| Part 48 | Static Files & Templates | 🔄 |
| Part 49 | Middleware Patterns | 🔄 |
| Part 50 | Full-stack Erlang Web App | 🔄 |

### ระดับที่ 6: Performance & Testing (Parts 51-70)
| Part | หัวข้อ | สถานะ |
|------|--------|-------|
| Part 51 | EUnit Testing | 🔄 |
| Part 52 | Common Test Framework | 🔄 |
| Part 53 | PropEr: Property-based Testing | 🔄 |
| Part 54 | Performance Profiling | 🔄 |
| Part 55 | Memory Management | 🔄 |
| Part 56 | Scheduler Optimization | 🔄 |
| Part 57 | NIFs: Native Implemented Functions | 🔄 |
| Part 58 | Ports และ Port Drivers | 🔄 |
| Part 59 | Benchmarking | 🔄 |
| Part 60 | Production Debugging | 🔄 |

### ระดับที่ 7: Advanced Patterns (Parts 61-80)
| Part | หัวข้อ | สถานะ |
|------|--------|-------|
| Part 61 | Design Patterns ใน Erlang | 🔄 |
| Part 62 | Actor Model ขั้นสูง | 🔄 |
| Part 63 | CQRS Pattern | 🔄 |
| Part 64 | Event Sourcing | 🔄 |
| Part 65 | Saga Pattern | 🔄 |
| Part 66 | Circuit Breaker | 🔄 |
| Part 67 | Backpressure & Flow Control | 🔄 |
| Part 68 | Streaming Data Processing | 🔄 |
| Part 69 | Real-time Systems | 🔄 |
| Part 70 | Reactive Systems | 🔄 |

### ระดับที่ 8: Integration & DevOps (Parts 71-90)
| Part | หัวข้อ | สถานะ |
|------|--------|-------|
| Part 71 | Docker & Erlang | 🔄 |
| Part 72 | Kubernetes Deployment | 🔄 |
| Part 73 | CI/CD Pipeline | 🔄 |
| Part 74 | Monitoring & Metrics | 🔄 |
| Part 75 | Prometheus Integration | 🔄 |
| Part 76 | Grafana Dashboards | 🔄 |
| Part 77 | RabbitMQ Integration | 🔄 |
| Part 78 | Kafka Integration | 🔄 |
| Part 79 | PostgreSQL Integration | 🔄 |
| Part 80 | Redis Integration | 🔄 |

### ระดับที่ 9: Real-world Projects (Parts 81-100)
| Part | หัวข้อ | สถานะ |
|------|--------|-------|
| Part 81 | Chat Server ด้วย Erlang | 🔄 |
| Part 82 | Game Server | 🔄 |
| Part 83 | IoT Data Platform | 🔄 |
| Part 84 | Financial Transaction System | 🔄 |
| Part 85 | Load Balancer | 🔄 |
| Part 86 | API Gateway | 🔄 |
| Part 87 | Microservices Architecture | 🔄 |
| Part 88 | Blockchain Prototype | 🔄 |
| Part 89 | Real-time Analytics | 🔄 |
| Part 90 | Production-ready System | 🔄 |

### ระดับที่ 10: World-Class (Parts 91-100+)
| Part | หัวข้อ | สถานะ |
|------|--------|-------|
| Part 91 | Erlang VM Internals | 🔄 |
| Part 92 | Writing Erlang NIFs | 🔄 |
| Part 93 | Compiler Extensions | 🔄 |
| Part 94 | Custom OTP Behaviors | 🔄 |
| Part 95 | Distributed Consensus Algorithms | 🔄 |
| Part 96 | BEAM Bytecode | 🔄 |
| Part 97 | Contributing to OTP | 🔄 |
| Part 98 | Writing Erlang Libraries | 🔄 |
| Part 99 | Open Source Erlang Projects | 🔄 |
| Part 100 | Career & Community | 🔄 |

---

## วิธีการเรียน

1. **อ่านและทำความเข้าใจ** — อ่านทฤษฎีและตัวอย่างในแต่ละ part
2. **ลองทำตาม** — พิมพ์โค้ดเองทุกครั้ง อย่า copy-paste
3. **ทดลองดัดแปลง** — แก้ไขตัวอย่างเพื่อสร้างความเข้าใจ
4. **สร้าง Project** — ทำ mini-project ในแต่ละระดับ
5. **สอนคนอื่น** — การสอนคือการเรียนรู้ที่ดีที่สุด

## Requirements

- OS: Linux, macOS หรือ Windows (WSL แนะนำ)
- Erlang/OTP 26+ 
- rebar3 (build tool)
- ความรู้ programming พื้นฐาน (ภาษาใดก็ได้)

---

*สร้างโดย Claude Code | อัพเดทวันที่ 2026-10-02*
