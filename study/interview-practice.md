# DevOps Interview Questions (Base-Mid Level)

## Linux Fundamentals

1. What's the difference between a process and a thread?
2. Explain what happens when you run `chmod 755 script.sh` - what do those numbers mean?
3. What's the difference between a soft link and a hard link?
4. How would you find all files modified in the last 24 hours under `/var/log`?
5. What does `kill -9` do differently than `kill -15`? Why does it matter?
6. Explain what load average means. If a system shows load average of `4.00, 3.50, 3.00` on a 2-core machine, what does that tell you?
7. What's the difference between `/etc/passwd` and `/etc/shadow`?
8. How do you check which process is using a specific port?
9. What's a zombie process and how does it happen?
10. Explain the difference between `stdout`, `stderr`, and how you'd redirect each.

---

## Networking

1. Walk through what happens when you type `https://example.com` in a browser and hit enter.
2. What's the difference between TCP and UDP? Give a use case for each.
3. What's a CIDR block? What does `/24` mean in `10.0.0.0/24`?
4. Explain the difference between a security group and a network ACL.
5. What's NAT and why would you use it?
6. What's the difference between Layer 4 and Layer 7 load balancing?
7. How does DNS resolution work? What's the difference between an A record and a CNAME?
8. What port does HTTPS use? SSH? DNS?
9. What's a VPN and how does it differ from a VPC?
10. How would you troubleshoot if you can't reach a remote server? Walk through your steps.

---

## Docker / Containerization

1. What's the difference between a container and a VM?
2. Explain the difference between `CMD` and `ENTRYPOINT` in a Dockerfile.
3. What's a Docker layer? Why do they matter for build times and image size?
4. How do you persist data in a container?
5. What's the difference between `COPY` and `ADD`?
6. What happens to a container's filesystem when the container stops?
7. How would you reduce the size of a Docker image?
8. What's a multi-stage build and when would you use it?
9. Explain the difference between bridge, host, and none network modes.
10. What's the difference between `docker-compose up` and `docker-compose run`?

---

## AWS

1. What's the difference between an IAM role and an IAM user?
2. Explain the difference between S3 Standard, S3 Standard-IA, and S3 Glacier.
3. What's the difference between an Application Load Balancer and a Network Load Balancer?
4. How does an Auto Scaling Group decide when to scale?
5. What's the difference between a public and private subnet?
6. What's an instance profile and why do you need one?
7. Explain what a VPC endpoint is and why you'd use one.
8. What's the difference between EBS and instance store?
9. How would you give an EC2 instance access to an S3 bucket securely (no hardcoded credentials)?
10. What's the difference between horizontal and vertical scaling? Which AWS services support each?

---

## CI/CD

1. What's the difference between continuous integration, continuous delivery, and continuous deployment?
2. Explain what a build artifact is.
3. What's the purpose of running tests in a pipeline? What types of tests belong where?
4. What's a merge/pull request workflow and why use it?
5. How do you handle secrets in a CI/CD pipeline?
6. What's the difference between a rolling deployment and a blue/green deployment?
7. What's a canary deployment?
8. Why would a build be "non-deterministic" or "flaky"? How do you fix it?
9. What's infrastructure as code and why does it matter for CI/CD?
10. What happens if two developers push conflicting changes? How should the pipeline handle it?

---

## Observability

1. What's the difference between logs, metrics, and traces?
2. What's a percentile (p50, p95, p99) and why would you use one instead of an average?
3. What are the four golden signals of monitoring?
4. What's the difference between black-box and white-box monitoring?
5. How do you decide what to alert on vs. what to just log?
6. What's structured logging and why does it matter?
7. Explain what distributed tracing solves.
8. What's cardinality and why is it a concern with metrics?
9. What's the difference between pull-based and push-based metrics collection?
10. If a service is slow, what metrics would you look at first?

---

## Kubernetes (Beginner)

1. What problem does Kubernetes solve? Why not just use Docker Compose?
2. What's the difference between a Pod and a container?
3. What's a Deployment and why would you use one instead of creating Pods directly?
4. Explain what a Service is. What's the difference between ClusterIP, NodePort, and LoadBalancer?
5. What's a namespace and why would you use one?
6. What's the difference between a ConfigMap and a Secret?
7. How does Kubernetes know if your application is healthy? What are liveness and readiness probes?
8. What's a ReplicaSet and how does it relate to a Deployment?
9. What happens when a node goes down? How does Kubernetes handle it?
10. What's `kubectl apply` vs `kubectl create`?
11. How do you view logs for a Pod? What if the Pod has multiple containers?
12. What's a DaemonSet and when would you use one?
13. Explain what an Ingress is and how it differs from a Service.
14. What's the role of etcd in a Kubernetes cluster?
15. How do you pass environment variables to a container in Kubernetes?

---

## About This Document

**Purpose:** This is a study guide for preparing for base-to-mid level DevOps interviews. The goal is not to simulate an interview 1:1, but to use these questions as a foundation for learning and building flashcards.

**How to use in conversation:**
- I will pick questions from this list and attempt to answer them.
- **Do not give me the answer outright.** Instead, ask clarifying questions, point out gaps in my reasoning, or guide me toward the answer through discussion.
- If my answer is wrong or incomplete, tell me what's missing or incorrect so I can refine my understanding.
- Once I've arrived at a solid answer (or we've fully discussed it), help me distill it into a flashcard-friendly format.

**Scope:** Linux, networking, Docker, AWS, CI/CD, observability, Kubernetes. No GCP, Azure, or Cisco.

**Living document:** New questions may be added as gaps are identified.