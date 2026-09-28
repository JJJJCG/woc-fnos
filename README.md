# woc-fnos

把 [WechatOnCloud](https://github.com/Gloridust/WechatOnCloud) 的**纯微信实例**打包成飞牛 fnOS 应用中心可安装的 `.fpk`。

在 NAS 上跑一台服务端微信，浏览器多端共享同一个登录态。装完只有一个容器、一个端口。

## 特性

- **只开放 3001（HTTPS）**：不需要的 3000（HTTP）口不做映射，攻击面更小
- **不依赖 `docker.sock`**：单实例直接用 compose 跑，权限收敛到专用应用用户，不碰宿主 root 能力
- **数据可管理**：`/config`（微信本体 + 登录态 + 聊天记录）落在应用共享目录，可在「文件管理」里查看、备份、迁移
- **设备伪装可配**：容器主机名与 MAC 由安装向导传入，用于降低微信判「非真实设备／设备农场」的风控概率
- **可后期修改**：镜像版本 / 主机名 / MAC 都能在「应用设置」里改，不用重装

## 下载安装

到 [Releases](../../releases) 下载最新 `.fpk` → 飞牛「应用中心 → 手动安装」→ 按向导填写。

访问地址：`https://<NAS地址>:3001/`

> 证书是镜像内置的自签名证书，浏览器首次会提示「不安全」，继续访问即可。

## 本仓库包含什么

只有打包工程 `woc-instance/` 和 CI，**不含** WechatOnCloud 的源码或容器镜像。

容器镜像在安装时从上游 registry 拉取：

```
ghcr.nju.edu.cn/gloridust/wechat-on-cloud   # 默认
docker.io/gloridust/wechat-on-cloud         # 备选，见 compose 注释
ghcr.io/gloridust/wechat-on-cloud           # 备选
```

## 自己构建

```bash
curl -fsSL -o fnpack https://static2.fnnas.com/fnpack/fnpack-1.2.3-linux-amd64
chmod +x fnpack && sudo mv fnpack /usr/local/bin/

cd woc-instance
fnpack build        # 产出 woc-instance.fpk（用 appname 命名，不带版本号）
```

CI 见 `.github/workflows/build-fpk.yml`：推 `vX.Y.Z` tag 会自动构建并创建 Release 挂上 `.fpk`。

## 目录结构

```
.
├── .github/
│   ├── release-notes.md            # 每个 Release 共用的安装说明
│   └── workflows/build-fpk.yml     # fnpack 打包 + 发布
└── woc-instance/                   # 飞牛应用包（fnpack 的输入）
    ├── app/docker/docker-compose.yaml
    ├── cmd/main                    # 生命周期：start/stop 返回 0，status 查容器
    ├── cmd/                        # 其余 install_*/upgrade_*/uninstall_* 可选
    ├── config/privilege            # run-as=package
    ├── config/resource             # docker-project + data-share 声明
    ├── wizard/install              # 安装向导
    ├── wizard/config               # 安装后从「应用设置」改同样的项
    ├── manifest
    ├── ICON.PNG
    └── ICON_256.PNG
```

打包细节与踩坑说明见 [`woc-instance/README.md`](woc-instance/README.md)。

## 免责声明

- 本仓库只提供**打包工程**，不包含上游源码，也不重新分发容器镜像；镜像归 / 由上游
  [Gloridust/WechatOnCloud](https://github.com/Gloridust/WechatOnCloud) 提供。
- 项目本身与腾讯、飞牛 fnOS 官方均无关联。
- 服务端微信的使用请自行确认符合软件许可与服务条款。里面的微信是**已登录状态**，
  请务必只在可信网络内访问，不要裸暴露到公网。

## 未验证项

本工程依据飞牛官方开发文档与社区 compose 打包示例编写，**尚未在真实 fnOS 设备上完整验证**。
上架前请实测：向导变量在 compose 中的可见性、`initValue` 默认值预填行为、
以及应用用户是否有权执行 `docker inspect`（影响应用中心的状态显示）。
详见 `woc-instance/README.md` 末尾。
