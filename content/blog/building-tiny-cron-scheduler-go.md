+++
title = "Building a Tiny Cron Scheduler With Go"
date = "2025-08-19"
tags = [
    "go",
    "fiber",
    "docker"
]
+++

If you're anything like me, you've probably got a bunch of scheduled tasks running in the cloud whether it's automation, IoT projects, data processing or just random experiments. My go-to approach has been to dockerize mini servers and deploy them on platforms like GCP or AWS. But getting something up and running on Cloud Run or Lambda isn't as lightweight as it sounds. You still need to push images to a registry, wire up an API gateway and set up a scheduler like Cloud Scheduler on GCP.

That's why I prefer hosting my applications on fly.io (and no, this isn't a sponsored post though if anyone from the Fly team is reading, I wouldn't say no 😉). Deploying apps there is ridiculously easy: just two commands with the fly CLI and you've got an app running at the edge, complete with built-in load balancing, managed Postgres, a Grafana dashboard to measure system perf and per-second billing which is perfect for workloads that don't need to run 24/7. But... I still wanted to push costs down even further.

My idea was simple: let those applications autosleep and only wake up when needed, using a cron scheduler. Sure, I could've hacked this together with something like Uptime Robot or a bash script, but I wanted a solution that was both lightweight and had a clean interface. Something where I could easily tweak cron schedules, choose between POST or GET, adjust frequencies and even define parameters like the request body.

![image info](/images/building-tiny-cron-scheduler-go/go-vs-node-perf.jpg)
_Credit: https://jaydevs.com/nodejs-vs-golang/_

My goal was to keep this super cheap while still being able to handle a large pool of requests. At first, I considered using Node.js, but even with Node Alpine in Docker, the overhead felt heavier than I wanted, especially compared to something lower-level like C or Rust. The problem was, I wasn't too keen on dealing with manual memory management or chasing down overflow bugs. That's where Go came in: lightweight, efficient, no manual garbage collection and far better performance on system resources than Node.

![image info](/images/building-tiny-cron-scheduler-go/application.jpg)
_UI built with Vue JS and BS5_

On the tech stack side, I went with Fiber since I already had experience with Express and found the syntax familiar. For the database layer, I used the PGX driver to connect to my self-hosted PostgreSQL server, and for scheduling I used the Cron v3 package to manage jobs. On the frontend, I built the UI using Vue 3 combined with Bootstrap 5, which gave me a clean interface.

After bundling everything with Docker on a Go Alpine base image and deploying to Fly (which uses Firecracker VMs), I was shocked to see the root filesystem being 663 MB 🤯 - far from the lightweight goal I had in mind. To fix this, I rebuilt my Docker image using multi-stage builds.

```dockerfile
# ---------- Build Stage ----------
FROM golang:1.24-alpine AS builder

RUN apk update && apk add --no-cache git bash upx

ENV GOPATH=/go
ENV PATH=$GOPATH/bin:$PATH

WORKDIR /app

# Install air for live reloading (optional for dev)
RUN go install github.com/air-verse/air@latest

COPY go.mod go.sum ./
RUN go mod tidy

COPY . .

# Build the binary
RUN go build -o main .

# Compress the binary with UPX
RUN upx --best --lzma ./main

FROM scratch

COPY --from=builder /app/main /main

EXPOSE 3000

# Run application
CMD ["/main"]

```

_Dockerfile with multi stage builds and UPX compression_

Multi-stage builds separate the build environment from the runtime environment, so your final image only contains what's necessary to run --- not all the tools and libraries needed to build your app. On top of that, I applied binary compression with UPX (Ultimate Packer for eXecutables). UPX is basically wizardry 🧙 and can shrink programs and DLLs by 50--70% without breaking them.

Result: A new root filesystem of 187 MB! Keep in mind, Fly deploys its own init process and SSH server, so expecting sub-100 MB images isn't realistic. Still, I was thrilled with the 77% reduction in size.

Overall, this project has been an awesome exercise in sharpening my Go skills, fine-tuning optimisations and creating a project to run tasks only when necessary saving compute costs which means a couple coffees ☕ saved each month :)

Thanks for reading!
