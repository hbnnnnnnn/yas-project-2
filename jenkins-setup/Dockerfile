FROM jenkins/jenkins:lts-jdk21

USER root

# ----------------------------
# 1. Install Docker CLI + Maven
# ----------------------------
RUN apt-get update && \
    apt-get install -y ca-certificates curl gnupg lsb-release maven && \
    install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/debian/gpg \
      | gpg --dearmor -o /etc/apt/keyrings/docker.gpg && \
    chmod a+r /etc/apt/keyrings/docker.gpg && \
    echo \
      "deb [arch=$(dpkg --print-architecture) \
      signed-by=/etc/apt/keyrings/docker.gpg] \
      https://download.docker.com/linux/debian \
      $(lsb_release -cs) stable" \
      > /etc/apt/sources.list.d/docker.list && \
    apt-get update && \
    apt-get install -y docker-ce-cli && \
    rm -rf /var/lib/apt/lists/*

# ----------------------------
# 2. Install kubectl
# ----------------------------
RUN curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && \
    install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl && \
    rm kubectl

# ----------------------------
# 3. Install Helm
# ----------------------------
RUN curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# ----------------------------
# 4. Fix permissions
# ----------------------------
RUN groupadd -f docker && \
    usermod -aG docker jenkins

USER jenkins