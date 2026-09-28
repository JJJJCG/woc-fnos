# 云微 · 微信实例（飞牛 fpk 包）

把 [WechatOnCloud](https://github.com/Gloridust/WechatOnCloud) 的**纯微信实例**（不含面板）打成
飞牛应用中心可安装的 `.fpk`。

本包只跑**一个微信实例容器**，不依赖宿主 `/var/run/docker.sock`，权限收敛到最小。
（上游仓库里的 `fnos/woc/` 打的是**面板** `woc-panel`，面板再按需创建实例、需要 docker.sock——
那是另一种形态，与本包互斥。）

## 目录结构

```
woc-instance/
├── app/docker/docker-compose.yaml  # 应用中心直接执行的 compose
├── cmd/main                        # 生命周期：start/stop 返回 0，status 看容器 woc
├── config/privilege                # run-as=package（专用应用用户）
├── config/resource                 # docker-project + data-share 声明
├── wizard/install                  # 安装向导：镜像标签、容器主机名、容器 MAC
├── wizard/config                   # 安装后从「应用设置」改同样的三项
├── manifest                        # service_port=3001, checkport=true
├── ICON.PNG                        # 64×64
└── ICON_256.PNG                    # 256×256
```

## 设备伪装与 MAC

容器内会伪装成一台个人电脑：主机名、持久 machine-id、os-release 改成 deepin（`WOC_SPOOF_OS=1`），
MAC 由安装向导传入（`wizard_mac`，必填）。

**为什么必填、且没有默认值**：固定 MAC 能让微信端的设备指纹跨重启稳定；但 MAC 属于
局域网内二层寻址，若两台设备装上同一个值，会互抢同一 IP，网络直接不通。所以这里不给默认值，
强制每台设备现场生成一个。安装前在飞牛 SSH 里跑：

```bash
openssl rand -hex 6 | sed 's/\(..\)/\1:/g;s/:$//'
```

也可以用这台 NAS 自己网卡的 MAC（`ip link show`）。装错了随时到「应用设置」里改，改完重启应用即可
（`/config` 数据卷不动，微信登录态保留）。

## 只开放 3001

KasmVNC 同时提供两个口：`3000` = HTTP、`3001` = HTTPS（同一个 web 客户端的两个监听）。
本包 compose 里只有：

```yaml
ports:
  - "3001:3001"
```

`3000` 仍在容器内监听，但不做映射，所以宿主上不会暴露。访问地址：`https://<NAS地址>:3001/`。

> 证书是镜像内置的自签名证书，浏览器首次会报「不安全」——继续访问即可。
> 若以后想同时开 HTTP，把 compose 的 `ports` 补一行 `"3000:3000"`，并在 manifest 里改
> `service_port`（或直接去掉 `checkport` 那行的固定端口语义）。

## 关键点（为什么这么写）

| 项 | 做法 | 原因 |
| --- | --- | --- |
| `/config` 落盘 | `/var/apps/woc-instance/shares/woc-instance/config:/config` | 用绝对路径。`./config` 在应用中心执行 compose 时解析不可控；声明了 `data-share` 飞牛才会建目录并给应用用户配 ACL |
| `PUID`/`PGID` | `${TRIM_UID}` / `${TRIM_GID}` | 飞牛应用用户 UID 不是 1000，写死会导致 `/config` 不可写、微信装不进去 |
| 容器名 | `woc` | 与上游仓库根 `docker-compose.*.yml` 保持一致，`cmd/main` 的 status 也查它 |
| `mac_address` | 由向导传入 `${wizard_mac}`，必填 | 固定 MAC 能让容器设备指纹跨重启稳定；但写死一个值 → 所有安装本包设备的容器 MAC 相同，同局域网必冲突。故做成必填 + 正则校验，安装时强制每台给不同的值 |
| `docker-project` | `name: woc-instance` / `path: docker` | `path` 是**相对于 `app/` 目录**的路径，写 `docker` 而非 `app/docker`；写错应用中心找不到 compose |
| 不挂 `docker.sock` | `run-as=package` 即可 | 单实例不需要动态创建容器，回避了面板包那个「package 用户无权访问 docker.sock」的坑 |

## 构建

CI 见仓库根的 `.github/workflows/build-fpk.yml`：推 `v*.*.*` tag 时自动下载官方 `fnpack`（linux-amd64）打包，
创建 GitHub Release 并挂上 `.fpk`；推 `main` 与手动触发则只构建、产物为 workflow artifact。

本地构建：

```bash
curl -fsSL -o /usr/local/bin/fnpack https://static2.fnnas.com/fnpack/fnpack-1.2.3-linux-amd64
chmod +x /usr/local/bin/fnpack

cd woc-instance
fnpack build
# 产出 woc-instance-<version>.fpk
```

然后在飞牛「应用中心 → 手动安装」上传 `.fpk`。

> **可执行位**：`cmd/main` 必须是 0755。Windows 上 `core.filemode=false`，`git add` 只会记成 100644，
> CI checkout 出来就不可执行。提交前补一条：
>
> ```bash
> git update-index --chmod=+x woc-instance/cmd/main
> ```
>
> CI 里也加了 `chmod +x woc-instance/cmd/*` 兜底，但本地提交时顺手改掉更干净。

## ⚠️ 尚未在真实 fnOS 设备上验证

依据官方文档 + 社区 compose 打包示例编写，上架前请实测：

1. **向导变量在 compose 里可见**：`wizard_version` / `wizard_hostname` / `wizard_mac` 依赖
   「向导值 → 同名环境变量」。安装后进应用设置（`wizard/config`）改值应能生效。
   注意 `wizard_mac` 是 `mac_address: ${wizard_mac}`，**没有 `:-默认值` 兜底**——
   若变量没传进来，compose 会因 MAC 非法直接起不来。这是有意为之（宁可报错也不给一个会撞车的默认 MAC），
   但意味着装完第一次启动必须确认 MAC 填对了。
2. **`initValue` 预填**：字段默认值用的是当前官方文档的 `initValue`。若你的 fnOS 版本上默认值不预填，
   把 `wizard/install` 里的 `initValue` 换成 `defaultValue`（部分示例工程用的是这个旧键名）。
3. **`cmd/main` 能调 `docker`**：status 依赖 `docker inspect`。若应用用户不在 `docker` 组且无权访问
   `docker.sock`，status 会恒返回「未运行」——不影响容器实际运行，但应用中心状态显示会不准。
   必要时在 `config/privilege` 加 `"join-groups": ["docker"]`。
4. **3001 端口冲突**：`checkport=true` 会让应用中心在启动前检查 3001 是否被占用。
