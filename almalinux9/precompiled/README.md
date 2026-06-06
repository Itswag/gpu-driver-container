# AlmaLinux 9.0 预编译 NVIDIA GPU 驱动容器镜像

基于 [yum-packaging-precompiled-kmod](https://github.com/NVIDIA/yum-packaging-precompiled-kmod) 项目构建预编译内核模块 RPM，生成可直接加载驱动的容器镜像。

支持在**离线环境**下通过 NVIDIA GPU Operator 自动安装 GPU 驱动。

## 前置条件

* 一台用于构建的 x86_64 Linux 机器，已安装 Docker（或 Podman）
* 可联网下载 NVIDIA 驱动和 AlmaLinux 软件包（构建时需要，运行时不需要）

## 构建步骤

### 1. 确定目标内核版本

在目标 AlmaLinux 9.0 节点上执行：

```bash
uname -r
# 输出示例: 5.14.0-70.13.1.el9_0.x86_64
```

### 2. 设置环境变量

```bash
export KERNEL_VERSION=5.14.0-70.13.1.el9_0.x86_64
export DRIVER_VERSION=580.159.04
export ALMALINUX_VERSION=9.0
```

### 3. 构建镜像

```bash
make image \
  KERNEL_VERSION=${KERNEL_VERSION} \
  DRIVER_VERSION=${DRIVER_VERSION} \
  ALMALINUX_VERSION=${ALMALINUX_VERSION}
```

镜像 tag 格式为：`${IMAGE_REGISTRY}/${IMAGE_NAME}:${DRIVER_VERSION}-${KERNEL_VERSION_TAG}-almalinux9.0`

### 4. 推送到私有仓库

```bash
# 重新 tag 到你的私有仓库
docker tag nvcr.io/ea-cnt/nv_only/driver:${DRIVER_VERSION}-${KERNEL_VERSION_TAG}-almalinux9.0 \
  your-registry.example.com/nvidia/driver:${DRIVER_VERSION}-${KERNEL_VERSION_TAG}-almalinux9.0

docker push your-registry.example.com/nvidia/driver:${DRIVER_VERSION}-${KERNEL_VERSION_TAG}-almalinux9.0
```

## 可选配置

### 使用自定义签名密钥

默认构建过程会生成自签名密钥。如需使用自定义签名密钥（例如 Secure Boot 场景），将密钥文件放在当前目录下：

- `private_key.priv`：私钥
- `public_key.der`：DER 格式公钥证书

### 自定义构建参数

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `ALMALINUX_VERSION` | `9.0` | AlmaLinux 版本 |
| `DRIVER_VERSION` | （必填） | NVIDIA 驱动版本 |
| `KERNEL_VERSION` | （必填） | 目标内核版本 |
| `BUILD_ARCH` | `x86_64` | 构建架构 |
| `DRIVER_OPEN` | `false` | 是否使用开源内核模块 |
| `DRIVER_TYPE` | `passthrough` | 驱动类型（`passthrough` 或 `vgpu`） |
| `IMAGE_REGISTRY` | `nvcr.io/ea-cnt/nv_only` | 镜像仓库地址 |
| `IMAGE_NAME` | `driver` | 镜像名称 |
| `CONTAINER_TOOL` | `docker` | 容器工具（`docker` 或 `podman`） |

详见 [Makefile](Makefile) 中的完整变量定义。

## GPU Operator 离线部署

### 配置 NVIDIADriver 自定义资源

```json
{
  "apiVersion": "nvidia.com/v1alpha1",
  "kind": "NVIDIADriver",
  "metadata": {
    "name": "gpu-driver"
  },
  "spec": {
    "driverType": "gpu",
    "usePrecompiled": true,
    "repository": "your-registry.example.com/nvidia",
    "image": "driver",
    "version": "580.159.04",
    "imagePullPolicy": "IfNotPresent"
  }
}
```

### 配置 ClusterPolicy

```json
{
  "spec": {
    "driver": {
      "enabled": true,
      "useNvidiaDriverCRD": true
    },
    "validator": {
      "driver": {
        "env": [
          {
            "name": "DISABLE_DEV_CHAR_SYMLINK_CREATION",
            "value": "true"
          }
        ]
      }
    }
  }
}
```

完整示例见 [nvidiadriver.json](nvidiadriver.json) 和 [clusterpolicy.json](clusterpolicy.json)。

> **注意**：预编译镜像必须与目标节点的**精确内核版本**匹配。如果节点通过 `dnf update` 升级了内核，需要重新构建对应版本的预编译镜像。

## 更多信息

- [Precompiled Driver Containers 文档](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/precompiled-drivers.html)
- [yum-packaging-precompiled-kmod](https://github.com/NVIDIA/yum-packaging-precompiled-kmod)
