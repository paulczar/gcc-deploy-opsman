FROM ubuntu:latest

RUN apt-get update > /dev/null \
    && apt-get install -yq wget jq curl > /dev/null

RUN wget -qO /usr/local/bin/yq https://github.com/mikefarah/yq/releases/download/2.4.0/yq_linux_amd64 \
    && chmod +x /usr/local/bin/yq

RUN wget -qO /usr/local/bin/om https://github.com/pivotal-cf/om/releases/download/4.0.1/om-linux-4.0.1 \
    && chmod +x /usr/local/bin/om
