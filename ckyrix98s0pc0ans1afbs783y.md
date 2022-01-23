## Soketi: 30% performance increase with ARM Docker images

Starting with [soketi 0.27.0](https://github.com/soketi/soketi/releases/tag/0.27.0), official images for the ARM architecture are available to run Soketi on.

Soketi is your simple, fast, and resilient open-source WebSockets server that is free, open-source, and can run in production at scale. 📣 [Read more about Soketi on the presentation post](https://blog.renoki.org/stop-paying-for-expensive-real-time).

In benchmarks, we tried to send 1 message per second for 500 concurrent users, half of them connecting and disconnecting, on average, about every 5-10 seconds to simulate high traffic, while the other half sits patiently on the server without interacting.
It took ~ 39ms of processing on average to deliver the message to the end-user (network overhead was not included).

In the ARM benchmarking, it took ~ 27ms of processing. Compared with the Linux x64 Docker containers, this translates to 30% better performance.

ARM-based architecture can be found in many devices, like mobile phones, IoT devices, or even [AWS's Graviton2 EC2 instances](https://www.amazonaws.cn/en/ec2/instance-types/). The ARM architecture is made for low-consumption devices, and it also comes with a performance boost compared with other architectures. (like `x64`)

You can now deploy a Pusher-compatible, fast, resilient, and free WebSockets server on edge with Raspberry Pi, in AWS's Graviton instances, or any general device that can run Docker.