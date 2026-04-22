# 项目运行指南（三种方案）

你当前项目可以按三种方式运行：

- 方案 A：直接跑通单体前端项目（开发模式）
- 方案 B：前端打包后用 Nginx 托管并代理后端（生产式复现）
- 方案 C：按当前 `Dockerfile` 打镜像并容器化部署（贴近交付）

---

## 0. 前置准备

- Node.js：项目实测可用 `v12.22.12`
- npm：随 Node 安装

---

## 1. 方案 A：跑通单体项目（开发模式）

适用场景：

- 优先本地开发、看页面、调试前端代码
- 不强依赖 Nginx 运行方式

### 步骤

1. 安装依赖

```bash
npm install
```

1. 启动开发服务器

```bash
npm run serve
```

1. 在浏览器打开终端提示地址（通常是 `http://localhost:8080`）

### 说明

- 该模式由 `vue-cli-service serve` 启动，属于开发服务器。
- 若接口联调异常，需确认后端可用、接口地址配置正确。

---

## 2. 方案 B：Nginx 托管（生产式本地复现）

适用场景：

- 想尽量贴近线上运行方式
- 需要验证“静态资源 + 反向代理”这一整套链路

### 步骤

1. 打包前端

```bash
npm run build
```

打包后会生成 `dist/` 目录。

1. 安装 Nginx（Mac）

```bash
brew install nginx
```

1. 配置 Nginx（采用项目内 `nginx.conf`，端口保持 `8889`）

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

upstream webservers {
    server 127.0.0.1:8080 weight=90;
}

server {
    listen       8889;
    server_name  localhost;

    root   /Users/xct/code/my-sky-take-out/project-sky-admin-vue-ts/dist;
    index  index.html index.htm;

    location /api/ {
        proxy_pass   http://localhost:8080/admin/;
    }

    location /user/ {
        proxy_pass   http://webservers/user/;
    }

    location /ws/ {
        proxy_pass   http://webservers/ws/;
        proxy_http_version 1.1;
        proxy_read_timeout 3600s;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "$connection_upgrade";
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

1. 应用配置（推荐直接覆盖本地 Nginx 主配置）

```bash
cp ./project-sky-admin-vue-ts/nginx.conf /opt/homebrew/etc/nginx/nginx.conf
```

然后把 `server` 中两处按本机改好：

- `listen 80;` 改为 `listen 8889;`
- `root html/sky;` 改为 `root /Users/xct/code/my-sky-take-out/project-sky-admin-vue-ts/dist;`

1. 检查并重启 Nginx

```bash
nginx -t
brew services restart nginx
```

1. 打开页面

- `http://localhost:8889`

1. 确保后端服务已启动（例如 `http://127.0.0.1:8080/admin`）

---

## 3. 方案 C：Docker 部署（严格按当前项目）

适用场景：

- 需要把前端以镜像形式交付或部署到服务器
- 希望与当前仓库内 `Dockerfile` 的行为保持一致

### 3.1 先理解当前 Dockerfile 的真实行为

当前 `Dockerfile` 做的事情非常明确：

- 使用 `nginx` 基础镜像
- 将本地 `dist/` 拷贝到容器的 `/usr/share/nginx/html`
- 启动 Nginx 对外提供静态页面

这意味着：

- `Dockerfile` **不会在镜像内执行 `npm run build`**
- 构建镜像前，必须先在宿主机生成 `dist/`
- 容器内默认只提供静态文件托管，不包含 `/api` 反向代理配置

### 3.2 部署步骤（生产环境）

1. 安装依赖（首次或依赖变化时）

```bash
npm install
```

若被 Cypress 下载卡住，可改为：

```bash
CYPRESS_INSTALL_BINARY=0 npm install
```

1. 构建前端产物（生产）

```bash
npm run build
```

执行后应生成 `dist/` 目录。

1. 在项目根目录构建镜像

```bash
docker build -t sky-admin:prod .
```

1. 运行容器（将容器 80 端口映射到宿主机 8889）

```bash
docker run -d --name sky-admin -p 8889:80 --restart unless-stopped sky-admin:prod
```

1. 访问页面

- `http://localhost:8889`

1. 常用运维命令

```bash
docker ps
docker logs -f sky-admin
docker restart sky-admin
docker stop sky-admin && docker rm sky-admin
```

### 3.3 UAT 构建与部署（可选）

项目已提供 `build:uat` 脚本，可按下列方式打 UAT 版本镜像：

```bash
npm run build:uat
docker build -t sky-admin:uat .
docker run -d --name sky-admin-uat -p 8890:80 --restart unless-stopped sky-admin:uat
```

### 3.4 接口联调说明（非常重要）

你的前端环境变量里 `VUE_APP_BASE_API=/api`，因此页面会请求同域下的 `/api/*`。

- 当前仓库 `Dockerfile` 未注入自定义 Nginx `location /api` 代理规则
- 如果直接运行上述容器，`/api` 是否可用取决于外层网关/Ingress/反向代理是否已转发
- 若无外层转发，需要额外补 Nginx 代理配置（或在上游网关完成 `/api -> 后端`）

---

## 4. 三种方案怎么选

- 想快速开发和调试：选 **方案 A**
- 想模拟线上部署链路：选 **方案 B**
- 想直接产出可部署镜像：选 **方案 C**

常见做法是：

- 日常开发用方案 A
- 联调收口或部署前验证用方案 B
- 实际服务器交付用方案 C

---

## 5. 常见问题

### 5.1 页面能打开但接口报错

- 检查后端是否启动
- 检查 Nginx 的 `proxy_pass` 地址、端口、路径是否匹配后端

### 5.2 刷新子路由 404

- 确认 Nginx 配置包含：

```nginx
try_files $uri $uri/ /index.html;
```

### 5.3 `npm install` 被 Cypress 卡住

```bash
CYPRESS_INSTALL_BINARY=0 npm install
```

---

## 6. 快速命令清单

开发模式：

```bash
npm install
npm run serve
```

Nginx 模式：

```bash
npm install
npm run build
nginx -t
brew services restart nginx
```

Docker 模式（生产）：

```bash
npm install
npm run build
docker build -t sky-admin:prod .
docker run -d --name sky-admin -p 8889:80 --restart unless-stopped sky-admin:prod
```

