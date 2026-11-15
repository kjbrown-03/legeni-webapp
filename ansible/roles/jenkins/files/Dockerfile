# Image Jenkins prête pour les TPs et le projet CI/CD - Baptiste Kaldjob
#   - Docker CLI + plugin compose (pilote le démon Docker de l'hôte via le socket)
#   - kubectl (déploiement Kubernetes)
#   - Ansible + collections kubernetes.core et community.docker
#   - plugins Jenkins préinstallés (plugins.txt)
# Adaptation : tentatives automatiques (apt / curl / pip) car la connexion est instable.
FROM jenkins/jenkins:lts-jdk21
LABEL maintainer="Baptiste Kaldjob"

USER root

RUN apt-get update -o Acquire::Retries=10 \
 && apt-get install -y -o Acquire::Retries=10 --no-install-recommends ca-certificates curl gnupg python3 python3-venv openssh-client \
 && install -m 0755 -d /etc/apt/keyrings \
 && curl -fsSL --retry 10 --retry-all-errors https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc \
 && echo "deb [signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
      > /etc/apt/sources.list.d/docker.list \
 && apt-get update -o Acquire::Retries=10 \
 && apt-get install -y -o Acquire::Retries=10 --no-install-recommends docker-ce-cli docker-compose-plugin docker-buildx-plugin \
 && rm -rf /var/lib/apt/lists/*

# kubectl
RUN KUBECTL_VERSION=$(curl -fsSL --retry 10 --retry-all-errors https://dl.k8s.io/release/stable.txt) \
 && curl -fsSL --retry 10 --retry-all-errors -o /usr/local/bin/kubectl "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl" \
 && chmod +x /usr/local/bin/kubectl

# Ansible dans un environnement virtuel Python
RUN python3 -m venv /opt/ansible \
 && /opt/ansible/bin/pip install --retries 10 --timeout 120 --no-cache-dir ansible-core==2.17.* kubernetes==31.* \
 && ln -s /opt/ansible/bin/ansible* /usr/local/bin/

# collections Ansible (étape séparée + 10 tentatives : Galaxy coupe souvent les connexions)
RUN for i in $(seq 1 10); do \
      ansible-galaxy collection install --timeout 120 -p /usr/share/ansible/collections kubernetes.core community.docker && exit 0; \
      echo "galaxy : essai $i rate, nouvel essai dans 15 s"; sleep 15; \
    done; exit 1

USER jenkins

COPY --chown=jenkins:jenkins plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN for i in $(seq 1 10); do \
      jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt && exit 0; \
      echo "plugins : essai $i rate, nouvel essai dans 15 s"; sleep 15; \
    done; exit 1
