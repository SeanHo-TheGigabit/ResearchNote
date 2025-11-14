# Linux Kernel Networking

Welcome to the Linux Kernel Networking documentation! This guide will take you from zero to mastery in understanding how the Linux kernel handles network communication.

## 📖 Table of Contents

1. [What is Kernel Networking?](./01-what-is-kernel-networking.md)
2. [The Network Stack Architecture](./02-network-stack-architecture.md)
3. [Packet Flow Through the Kernel](./03-packet-flow.md)
4. [Key Functions and Structures](./04-key-functions.md)
5. [Network Device Drivers](./05-network-drivers.md)

## 🎯 Learning Path

### For Beginners (📚)
Start here if you're new to networking or kernel concepts:
1. Read "What is Kernel Networking?" - Simple analogies
2. Read "Network Stack Architecture" - The big picture
3. Skim "Packet Flow" - Just the overview sections

### For Students (🎓)
You have basic networking knowledge (TCP/IP, sockets):
1. Review all overview sections
2. Focus on the "Practitioner's View" in each document
3. Study the packet flow diagrams

### For Advanced Learners (🔬)
You've written network programs or studied OS internals:
1. Skip to "Deep Dive" sections
2. Study the function call chains
3. Explore the key data structures

### For Experts (🧬)
You're contributing to kernel code or doing research:
1. Jump directly to "Source Truth" sections
2. Reference the function details
3. Check the kernel version notes

## 🗺️ Quick Navigation

**Want to understand...** → **Read this**
- How packets travel from wire to application? → [Packet Flow](./03-packet-flow.md)
- What `sk_buff` is? → [Key Structures](./04-key-functions.md#sk_buff)
- How network drivers work? → [Network Drivers](./05-network-drivers.md)
- What NAPI means? → [Network Drivers](./05-network-drivers.md#napi)

## 🎓 Prerequisites by Level

### 📚 Beginner Level
- Curiosity about how internet works
- Basic understanding of computers

### 🎓 Intermediate Level
- Know what TCP/IP is
- Understand client-server concepts
- Familiar with programming concepts

### 🔬 Advanced Level
- C programming experience
- Operating systems concepts
- Network protocol knowledge

### 🧬 Expert Level
- Kernel programming experience
- Deep OS internals knowledge
- Assembly language understanding (helpful)

## 🚀 Getting Started

If you're not sure where to start, begin with [What is Kernel Networking?](./01-what-is-kernel-networking.md) and follow the links at the bottom of each page.

---

*Remember: Every expert was once a beginner. Take your time, and enjoy the journey!*
