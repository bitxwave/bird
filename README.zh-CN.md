# bird

[BIRD](https://bird.network.cz/)（BIRD Internet Routing Daemon）的 Docker 镜像与配置。  
[English](README.md)

---

### 简介

本仓库提供 **BIRD**（BIRD Internet Routing Daemon）的轻量 Docker 镜像。BIRD 是常用的 BGP/OSPF 路由守护进程。镜像基于 Debian Trixie slim，并以非 root 用户运行。

### 特性

- **多版本**：通过构建参数指定任意发布版本（如 2.0.10、3.0.1）；源码优先取官方发布 tarball，不可达时回退到[上游 git 归档](https://gitlab.nic.cz/labs/bird)。
- **多架构**：镜像提供 `linux/amd64` 与 `linux/arm64`，拉取时会自动匹配当前架构。
- **精简**：多阶段构建，最终镜像仅包含运行时依赖。
- **非 root**：以用户 `bird` 运行；`bird` 二进制已设置文件能力（`cap_net_raw`、`cap_net_admin`），无需 root 即可跑 OSPF/BGP。
- **配置友好**：可挂载自己的 `bird.conf`；若不存在则使用示例配置，并追加 stderr 日志配置，保证 `docker logs` 能看到输出。
- **健康检查**：内置通过 `birdc show status` 的健康检查。

### 镜像

镜像发布在 GitHub Container Registry。建议使用与 BIRD 官方版本一致的标签，或使用 `latest`（最近一次按 tag 构建的版本）。

```sh
# 按版本拉取（推荐，与 BIRD 发布版本一致）
docker pull ghcr.io/picopock/bird:2.0.10
docker pull ghcr.io/picopock/bird:3.0.1

# 或拉取 latest（最近一次打 tag 触发的构建）
docker pull ghcr.io/picopock/bird:latest
```

### 快速开始

```sh
# 使用自己的配置（推荐加 --network host，便于邻居发现）
docker run -d --name bird \
  --network host \
  --cap-add=NET_ADMIN --cap-add=NET_RAW \
  -v /path/to/your/bird.conf:/etc/bird/bird.conf:ro \
  ghcr.io/picopock/bird:latest

# 仅用默认配置（示例配置；邻居发现通常仍需 --network host）
docker run -d --name bird \
  --cap-add=NET_ADMIN --cap-add=NET_RAW \
  ghcr.io/picopock/bird:latest
```

**网络与邻居发现：** 要让其它路由器发现本实例，请使用 `--network host`（BIRD 使用宿主机网卡，适合 OSPF/BGP）或与宿主机同网段的 macvlan，并对端配置为容器的 IP。使用默认 bridge 时，OSPF 组播与 BGP（179 端口）无法从物理网段访问，邻居无法建立。

**能力与权限：** 镜像已对 `bird` 二进制设置文件能力，非 root 也可创建原始套接字。运行容器时仍建议加上 `--cap-add=NET_ADMIN --cap-add=NET_RAW`，以保证在各种环境下 OSPF 邻居都能正常建立。

### 配置说明

- **配置文件**：`/etc/bird/bird.conf`
- **控制套接字**：`/var/run/bird/bird.ctl`
- **示例配置**：`/etc/bird/bird.conf.example`

挂载点：

- `/etc/bird` — 配置目录（可在此挂载自己的 `bird.conf`）。
- `/var/run/bird` — 运行时目录（控制套接字等）。

若启动时不存在 `/etc/bird/bird.conf`，入口脚本会从示例配置复制并追加 `log stderr all;`。控制套接字路径由 `-s` 参数传给 BIRD，不写在配置文件里。

### 本地构建

```sh
# 默认构建（BIRD 3.0.1）
docker build -t bird:local .

# 指定 BIRD 版本构建
docker build -t bird:2.0.10 --build-arg BIRD_VERSION=2.0.10 .
```

版本号须为上游存在的 `x.y.z` 格式发布版本（见[发布 tag 列表](https://gitlab.nic.cz/labs/bird/-/tags)）。

### 版本与 CI

- 推送 **tag**（如 `v2.0.10`、`v3.0.1`）会触发 GitHub Actions 构建。
- 工作流根据 tag 设置 `BIRD_VERSION`（如 `v2.0.10` → `2.0.10`），构建并推送到 `ghcr.io/picopock/bird`，同时打上版本标签和 `latest`。
- Tag 必须为 **semver** 格式（如 `v1.2.3`），否则工作流会报错退出。

### 许可

BIRD 使用 GNU General Public License。详见 [BIRD 项目](https://bird.network.cz/)。本 Docker 镜像与配置按原样提供。
