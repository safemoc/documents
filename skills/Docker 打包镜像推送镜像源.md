# 打包镜像
##  构建`Dockerfile` 文件

```sh
# syntax=docker/dockerfile:1

# 提供 Node 22 运行时
FROM node:22-bookworm-slim AS node-runtime

# Python 3.12 + uv
FROM ghcr.io/astral-sh/uv:python3.12-bookworm-slim

# 将 Node/npm 复制到 Python 镜像
COPY --from=node-runtime /usr/local/ /usr/local/

RUN apt-get update \
    && apt-get install -y --no-install-recommends supervisor \
    && rm -rf /var/lib/apt/lists/*

# ---------- 后端 ----------
WORKDIR /app/backend

ENV UV_LINK_MODE=copy \
    UV_PROJECT_ENVIRONMENT=/app/backend/.venv \
    PATH="/app/backend/.venv/bin:${PATH}" \
    PYTHONUNBUFFERED=1 \
    INSTANCE_ROLE=admin \
    MODE=prod \
    BACKEND_PORT=8010

COPY aimi-agent/pyproject.toml aimi-agent/uv.lock ./
RUN uv sync --frozen --no-dev

COPY aimi-agent/ ./

# ---------- 前端 ----------
WORKDIR /app/frontend

COPY aimi-agent-frontend/package.json \
     aimi-agent-frontend/package-lock.json ./

RUN npm ci

COPY aimi-agent-frontend/ ./

# Vite 将 Admin API 代理到同容器后端
ENV DEV_BACKEND_URL=http://127.0.0.1:8010 \
    DEV_PORT=3100

COPY docker/all/supervisord-dev.conf \
     /etc/supervisor/conf.d/aimi.conf

EXPOSE 3100

CMD ["/usr/bin/supervisord", "-n", "-c", "/etc/supervisor/supervisord.conf"]

```

```sh
docker build -t <image name> .
```

# 登陆镜像源
```sh
docker login --username=<username>@<user id> <image url>
```
```sh
输入密码:

显示 Login Succeeded 即可
```

# 给`Image`打标签
```sh 
docker tag <image name> <image url>/<image namespace>/<image>:<version>
```

# 推送到 镜像源

这个要和打的标签相同
```sh
docker push <image url>/<image namespace>/<image>:<version>
```

