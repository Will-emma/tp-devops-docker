FROM python:3.12-slim
# Client SSH pour se connecter au serveur
RUN apt-get update && apt-get install -y --no-install-recommends openssh-client \
    && rm -rf /var/lib/apt/lists/*
# Ansible complet (avec la collection community.docker)
RUN pip install --no-cache-dir ansible
WORKDIR /ansible
ENTRYPOINT ["ansible-playbook"]
