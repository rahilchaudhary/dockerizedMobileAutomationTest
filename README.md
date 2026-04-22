# dockerizedMobileAutomationTest
This solves the cost and accessibility problem for teams testing on fixed hardware targets
Jenkins pipeline that runs an Android test suite on a physical device inside a Docker container
# Android Real-Device Automation - Containerised CI/CD Pipeline

A Jenkins pipeline that runs an Android test suite on a physical device inside a Docker container. No device farm. No per-minute billing. Any team member registers their laptop as a Jenkins node in under a minute and runs the suite - nightly or against their own branch.

---

## The Problem

Our Android automation setup problem: only a few engineers could run owing to:
Specific environment dependencies. Local config files. Manual Appium setup. The automation exists, but it's fragile and gated.
Our context: a fixed set of Lenovo devices running Android 10, 11, and 13, deployed to schools. Known hardware. Known OS targets. No fragmentation problem to solve.

## The Solution
Containerise the test runtime. The entire execution environment lives in a Docker image. No local setup. No "works on my machine." Any team member - developer or QA - pulls the image and runs.
Distribute execution via Jenkins. Register your laptop as a node in a minute. Trigger the suite against your branch before raising a PR, or let the nightly run handle regression. Same pipeline, same device, same results.
The outcome: real-device automation that doesn't sit behind a specialist. Zero device farm cost. Developers can validate against actual hardware without asking anyone.

---
## Trade Offs

For the device problem:
- BrowserStack / LambdaTest / AWS Device Farm - cost, and they may not have your exact Lenovo hardware

For Jenkins distributed nodes:
- Could have used a single machine everyone SSHes into - single point of failure, queuing problem
- Could have used GitHub Actions / GitLab CI - but those don't give you easy access to a local machine with a physical device attached

## Architecture

https://github.com/rahilchaudhary/dockerizedMobileAutomationTest/blob/main/Dockerized_Mobile.jpg

---

## Prerequisites

- **Docker Desktop** installed and running on your local machine
- **ADB** installed on your local machine. Device connected over WiFi. Run `adb devices` to confirm the device is visible.
- **Jenkins** instance - EC2. Register your machine as a node (takes under a minute).
- **Android device** on the same WiFi network as your machine, with TCP/IP mode enabled on port 5555.

To enable TCP/IP mode on your device (one-time setup):
```bash
adb tcpip 5555
```

---

## Repo Structure

```
Dockerfile.base         # Base image: Java 17, Maven, Node.js, Appium, UIAutomator2, ADB
Dockerfile.project      # Project image: clones test repo, sets up runtime entrypoint
Jenkinsfile             # Parametrised Jenkins pipeline
Architecture diagram: Dockerized_Mobile.jpg
```

---

## Step 1 - Build the base image

```bash
docker build -f Dockerfile.base -t your-automation-base .
```

---

## Step 2 - Build the project image

```bash
docker build \
  --build-arg REPO_ACCESS_TOKEN=<your-token> \
  --build-arg GIT_BRANCH=<your-branch> \
  --no-cache \
  -f Dockerfile.project \
  -t <your-image-name> .
```

| Build arg | Description |
|---|---|
| `REPO_ACCESS_TOKEN` | Personal access token with read access to your test repository |
| `GIT_BRANCH` | Branch to clone into the image |

---

## Step 3 - Register your machine as a Jenkins node

In your Jenkins instance go to **Manage Jenkins → Nodes → New Node**. Select permanent agent. Copy the agent launch command and run it on your machine. Your machine is now a Jenkins node.

---

## Step 4 - Run the pipeline

Trigger the pipeline from Jenkins with the following parameters:

| Parameter | Description | Example |
|---|---|---|
| `DEVICE_IP` | IP address of the Android device on your network | `192.168.1.30` |
| `DOCKER_IMAGE` | Docker Hub image name | `yourusername/your-image` |
| `IMAGE_TAG` | Image tag | `latest` |
| `CONTAINER_NAME` | Name for the Docker container | `automation-container` |
| `APPLICATION` | TestNG XML suite file name | `MyApp` |
| `ENVIRONMENT` | Target environment | `prod` or `ft` |
| `SUITE` | Test suite to run | `sanity`, `smoke`, `regression` |
| `CUCUMBER_TAGS` | Cucumber tag expression | `(@TeachModule)` |
| `APPIUM_PORT` | Appium server port inside container | `4723` |
| `PLATFORM` | Execution platform | `ANDROID` |
| `BROWSER` | Execution mode | `native` |
| `REPORT_FOLDER` | Path on local machine where report will be saved | `DRIVE:\Reports` |

---

## What this does not cover

This setup is designed for a **fixed hardware target** - one device, known OS versions, same network. It is not a fragmentation testing solution and does not support parallel execution across multiple devices. The WiFi dependency means the device and the Jenkins node machine must be on the same network.

---

## How it works at runtime

1. Jenkins EC2 triggers the pipeline on the registered local node
2. Pre-flight check verifies the Android device is reachable via ADB - aborts immediately if not
3. Docker pulls the test image from Docker Hub
4. Container starts, connects to the device over ADB WiFi, and runs the Maven test suite via Appium
5. Test report is copied from the container to the local machine
6. Container is cleaned up regardless of test outcome
