# Seamless Streaming: EC2 Deployment Guide

This document provides detailed instructions for deploying the [Seamless Streaming](https://huggingface.co/spaces/facebook/seamless-streaming) application on an AWS EC2 instance using GPU acceleration. Seamless Streaming is a multilingual, real-time translation system developed by Meta, capable of speech-to-speech translation using a combination of models including SeamlessM4T and fairseq2. The app includes a web frontend and a backend server running on Uvicorn with SSL support.

---

## 1. Overview of Deployment

- **Model & System**: Seamless Streaming uses a backend implemented with PyTorch and Fairseq2 for inference, a WebSocket server using FastAPI/Uvicorn for communication, and a React-based frontend.
- **Deployment**: The app is containerized with Docker and supports GPU acceleration via the NVIDIA Container Toolkit. It can be hosted on an EC2 instance for scalable deployment.
- **Frontend Port**: By default, the application listens on port `7860`.
- **SSL**: Optional self-signed SSL certificates can be configured.

---

## 2. EC2 Setup

### A. Launch an EC2 Instance

1. Go to the AWS EC2 dashboard and click **Launch Instance**.
2. Select **Ubuntu 22.04** as the AMI.
3. Choose an instance type with GPU support. Recommended: `g4dn.large` (includes a T4 GPU).
4. Configure **Security Group**:
   - Allow **port 22** (SSH) and **port 7860** (app).
5. Generate a new key pair or use an existing one.
6. Launch the instance and SSH into it:

```bash
ssh -i /path/to/key.pem ubuntu@<EC2-Public-IP>
```

---

## 3. Install System Dependencies

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common git
```

### Optional: Install Git LFS
```bash
sudo apt-get install -y git-lfs
```

---

## 4. Install Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker ubuntu
newgrp docker
```

---

## 5. Install NVIDIA Container Toolkit

```bash
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/libnvidia-container/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list   | sudo tee /etc/apt/sources.list.d/libnvidia-container.list

sudo apt-get update
sudo apt-get install -y nvidia-docker2
sudo systemctl restart docker
```

### Verify GPU Access
```bash
docker run --rm --gpus all nvidia/cuda:11.8.0-cudnn8-devel-ubuntu22.04 nvidia-smi
```

---

## 6. Clone the Repository

```bash
git clone https://huggingface.co/spaces/facebook/seamless-streaming.git
cd seamless-streaming
```

If private:
```bash
git lfs install
huggingface-cli login
```

---

## 7. Set Up SSL Certificates (Optional)

```bash
mkdir -p seamless_server/certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048   -keyout seamless_server/certs/key.pem   -out seamless_server/certs/cert.pem   -subj "/CN=seamless-streaming"
```

Update `seamless_server/run_docker.sh`:
```bash
uvicorn app_pubsub:app   --host 0.0.0.0   --port 7860   --ssl-keyfile certs/key.pem   --ssl-certfile certs/cert.pem
```

---

## 8. Build and Run Docker Image

### Build:
```bash
docker build -t seamless-streaming .
```

### Run:
```bash
docker run -it --gpus all -p 7860:7860 --network host seamless-streaming
```

### With HF_TOKEN for Private Repos:
```bash
docker build --secret id=HF_TOKEN,src=/path/to/tokenfile -t seamless-streaming .
docker run -it --gpus all -p 7860:7860 seamless-streaming
```

---

## 9. Access the Application

- Open your browser:
  - Without SSL: `http://<EC2-Public-IP>:7860`
  - With SSL: `https://<EC2-Public-IP>:7860`

You should see the Seamless Streaming frontend interface.

---

## 10. Maintenance & Best Practices

- **Security**:
  - Use secure key pairs
  - Restrict security group IP access
  - Replace self-signed SSL with a valid certificate

- **Logging**:
  ```bash
  docker logs <container_id>
  ```

- **Persistence**:
  Use Docker volumes for persistent storage:
  ```bash
  docker run -v /host/path:/container/path ...
  ```

- **Monitoring**:
  Use CloudWatch or another monitoring stack for container health.

- **Scaling**:
  Consider running behind an AWS Load Balancer for multi-instance deployments.
