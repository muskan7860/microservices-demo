FROM jenkins/jenkins:lts

USER root

ENV DEBIAN_FRONTEND=noninteractive

# -----------------------------
# Install base dependencies
# -----------------------------
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    unzip \
    gnupg \
    apt-transport-https \
    git \
    docker.io

# -----------------------------
# Install kubectl (Kubernetes CLI)
# -----------------------------
RUN curl -LO "https://dl.k8s.io/release/v1.36.0/bin/linux/amd64/kubectl" \
    && chmod +x kubectl \
    && mv kubectl /usr/local/bin/

# -----------------------------
# Install Sonar Scanner
# -----------------------------
RUN wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip \
    && unzip sonar-scanner-cli-5.0.1.3006-linux.zip \
    && mv sonar-scanner-5.0.1.3006-linux /opt/sonar-scanner \
    && ln -s /opt/sonar-scanner/bin/sonar-scanner /usr/bin/sonar-scanner

# -----------------------------
# Install Trivy (SECURITY SCANNER - FIXED)
# -----------------------------
RUN apt-get update && apt-get install -y wget gnupg

RUN wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key \
    | gpg --dearmor > /usr/share/keyrings/trivy.gpg

RUN echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" \
    > /etc/apt/sources.list.d/trivy.list

RUN apt-get update && apt-get install -y trivy

# -----------------------------
# Give Jenkins permission for docker
# -----------------------------
RUN usermod -aG docker jenkins

# -----------------------------
# Switch back to Jenkins user
# -----------------------------
# -----------------------------
# Add DNS + Switch to jenkins user (with docker group)
# -----------------------------
RUN echo "nameserver 8.8.8.8" >> /etc/resolv.conf && \
    echo "nameserver 1.1.1.1" >> /etc/resolv.conf

# ✅ Run as jenkins user (who is now in docker group)
USER jenkins
