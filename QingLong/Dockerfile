FROM node:20-slim AS nodebuilder

FROM python:3.11-slim-bullseye AS builder
ARG QL_MAINTAINER="whyour"
LABEL maintainer="${QL_MAINTAINER}"
ARG QL_URL=https://github.com/${QL_MAINTAINER}/qinglong.git
ARG QL_BRANCH=debian

ENV QL_DIR=/ql \
  QL_BRANCH=${QL_BRANCH}

COPY --from=nodebuilder /usr/local/bin/node /usr/local/bin/
COPY --from=nodebuilder /usr/local/lib/node_modules/. /usr/local/lib/node_modules/
RUN set -x && \
  ln -s /usr/local/lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm && \
  apt-get update && \
  apt-get install --no-install-recommends -y libatomic1 git && \
  git config --global user.email "qinglong@@users.noreply.github.com" && \
  git config --global user.name "qinglong" && \
  git config --global http.postBuffer 524288000 && \
  git clone --depth=1 -b ${QL_BRANCH} ${QL_URL} ${QL_DIR}

RUN mkdir /tmp/build && \
  cp ${QL_DIR}/package.json ${QL_DIR}/.npmrc ${QL_DIR}/pnpm-lock.yaml /tmp/build/ && \
  npm i -g pnpm@8.3.1 && \
  cd /tmp/build && \
  pnpm install --prod && \
  rm -rf /root/.pnpm-store /root/.cache /root/.npm

FROM python:3.11-slim-bullseye

ARG QL_MAINTAINER="whyour"
LABEL maintainer="${QL_MAINTAINER}"
ARG QL_URL=https://github.com/${QL_MAINTAINER}/qinglong.git
ARG QL_BRANCH=debian

ENV PNPM_HOME=/root/.local/share/pnpm \
  QL_DIR=/ql \
  QL_BRANCH=${QL_BRANCH}

ENV PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:${PNPM_HOME}:${PNPM_HOME}/global/5/node_modules:${PNPM_HOME} \
  NODE_PATH=/usr/local/bin:/usr/local/pnpm-global/5/node_modules:/usr/local/lib/node_modules:/root/.local/share/pnpm/global/5/node_modules \
  LANG=C.UTF-8 \
  SHELL=/bin/bash \
  PS1="\u@\h:\w \$ "

COPY --from=nodebuilder /usr/local/bin/node /usr/local/bin/
COPY --from=nodebuilder /usr/local/lib/node_modules/. /usr/local/lib/node_modules/

RUN set -x && \
  ln -s /usr/local/lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm && \
  ln -s /usr/local/lib/node_modules/npm/bin/npx-cli.js /usr/local/bin/npx && \
  apt-get update && \
  apt-get upgrade -y && \
  apt-get install --no-install-recommends -y \
  git curl sudo build-essential default-jdk default-jre g++ gcc \
  libzbar0 fish ffmpeg nmap ca-certificates zsh cron wget tzdata \
  perl openssl openssh-client nginx jq procps netcat sshpass rclone \
  unzip libatomic1 \
  chromium chromium-driver fonts-liberation libasound2 \
  libatk-bridge2.0-0 libgtk-3-0 libnspr4 libnss3 libx11-xcb1 \
  libxss1 libxtst6 xdg-utils && \
  apt-get clean && \
  rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* /usr/share/doc /usr/share/man && \
  ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
  echo "Asia/Shanghai" >/etc/timezone && \
  git config --global user.email "qinglong@@users.noreply.github.com" && \
  git config --global user.name "qinglong" && \
  git config --global http.postBuffer 524288000 && \
  npm install -g pnpm@8.3.1 pm2 ts-node && \
  rm -rf /root/.pnpm-store /root/.local/share/pnpm/store /root/.cache /root/.npm && \
  chmod u+s /usr/sbin/cron && \
  ulimit -c 0

RUN curl -fsSL https://code-server.dev/install.sh | sh -s -- --version=4.96.4 && \
  rm -rf /tmp/code-server-install.sh /root/.cache

RUN curl -fsSL https://ollama.com/install.sh | sh && \
  rm -rf /tmp/ollama-install.sh /root/.ollama/.install

ARG SOURCE_COMMIT
# 🎯 核心修复：移除 URL 空格 + 确保目录存在
RUN git clone --depth=1 -b ${QL_BRANCH} ${QL_URL} ${QL_DIR} && \
  cd ${QL_DIR} && \
  cp -f .env.example .env && \
  chmod 777 ${QL_DIR}/shell/*.sh ${QL_DIR}/docker/*.sh && \
  mkdir -p ${QL_DIR}/static && \
  git clone --depth=1 -b ${QL_BRANCH} https://github.com/${QL_MAINTAINER}/qinglong-static.git /tmp/static && \
  cp -rf /tmp/static/* ${QL_DIR}/static/ && \
  rm -rf /tmp/static ${QL_DIR}/.git && \
  rm -f ${QL_DIR}/docker/docker-entrypoint.sh

COPY docker-entrypoint.sh ${QL_DIR}/docker
COPY front.conf ${QL_DIR}/docker/front.conf

RUN mkdir -p /ql/data/{config,log,db,scripts,repo,raw,deps} && \
  chmod -R 777 /ql /var /usr/local /etc/nginx /run /usr /root

COPY --from=builder /tmp/build/node_modules/. /ql/node_modules/
COPY services.json /ql/node_modules/nodemailer/lib/well-known/
COPY notify.py /notify.py

RUN useradd -m -s /bin/zsh coder && \
  echo 'coder ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers && \
  mkdir -p /home/coder/.local/share/code-server/User && \
  chmod -R 777 /home/coder && \
  echo '{ "workbench.colorTheme": "Default Dark Modern", "telemetry.enableTelemetry": true, "telemetry.enableCrashReporter": true }' > /home/coder/.local/share/code-server/User/settings.json && \
  chown -R coder:coder /home/coder/.local/share/code-server

USER coder
ENV HOME=/home/coder PATH=/home/coder/.local/bin:$PATH
WORKDIR ${QL_DIR}

HEALTHCHECK --interval=5s --timeout=2s --retries=20 \
  CMD curl -sf --noproxy '*' http://127.0.0.1:5400/api/health || exit 1

ENTRYPOINT ["./docker/docker-entrypoint.sh"]
VOLUME /ql/data
EXPOSE 5700
