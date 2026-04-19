FROM jenkins/jenkins:lts-jdk21

USER root

# ----------------------------
# 1. Install tools + Docker CLI + Maven
# ----------------------------
RUN apt-get update && \
    apt-get install -y ca-certificates curl gnupg lsb-release maven wget tar && \
    install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/debian/gpg \
      | gpg --dearmor -o /etc/apt/keyrings/docker.gpg && \
    chmod a+r /etc/apt/keyrings/docker.gpg && \
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
      https://download.docker.com/linux/debian $(lsb_release -cs) stable" \
      > /etc/apt/sources.list.d/docker.list && \
    apt-get update && \
    apt-get install -y docker-ce-cli && \
    rm -rf /var/lib/apt/lists/*

# ----------------------------
# 2. Install JDK 25.0.2 (Temurin) — amd64 + arm64
# ----------------------------
RUN ARCH=$(dpkg --print-architecture) && \
    if [ "$ARCH" = "amd64" ]; then \
      JDK_URL="https://github.com/adoptium/temurin25-binaries/releases/download/jdk-25.0.2%2B10/OpenJDK25U-jdk_x64_linux_hotspot_25.0.2_10.tar.gz"; \
    elif [ "$ARCH" = "arm64" ]; then \
      JDK_URL="https://github.com/adoptium/temurin25-binaries/releases/download/jdk-25.0.2%2B10/OpenJDK25U-jdk_aarch64_linux_hotspot_25.0.2_10.tar.gz"; \
    fi && \
    wget -q "$JDK_URL" -O /tmp/jdk25.tar.gz && \
    mkdir -p /usr/lib/jvm && \
    tar -xzf /tmp/jdk25.tar.gz -C /usr/lib/jvm && \
    mv /usr/lib/jvm/jdk-25.0.2+10 /usr/lib/jvm/jdk-25 && \
    rm /tmp/jdk25.tar.gz

ENV JAVA_HOME=/usr/lib/jvm/jdk-25
ENV PATH="$JAVA_HOME/bin:$PATH"

# ----------------------------
# 3. Install kubectl
# ----------------------------
RUN curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/$(dpkg --print-architecture)/kubectl" && \
    install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl && \
    rm kubectl

# ----------------------------
# 4. Install Helm
# ----------------------------
RUN curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# ----------------------------
# 5. Fix permissions
# ----------------------------
RUN groupadd -f docker && \
    usermod -aG docker jenkins

USER jenkins