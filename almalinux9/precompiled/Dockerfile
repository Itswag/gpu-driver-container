# 基于 AlmaLinux 9.0 的 GPU 预编译驱动容器镜像
# 使用 RPM kmod 模式预编译 NVIDIA 内核模块，支持离线环境 GPU Operator 部署

ARG ALMALINUX_VERSION=9.0
ARG ALMALINUX_BASE_IMAGE=almalinux:${ALMALINUX_VERSION}

# ============================================================
# 阶段一：Builder - 编译 NVIDIA 内核模块 RPM
# ============================================================
FROM ${ALMALINUX_BASE_IMAGE} AS builder

ARG BASE_URL='https://us.download.nvidia.com/tesla'

ARG KERNEL_VERSION=''
ARG ALMALINUX_VERSION
ARG RHEL_VERSION_MAJOR='9'

ARG BUILD_ARCH='x86_64'
ARG TARGET_ARCH='x86_64'

ARG DRIVER_VERSION=''
ARG DRIVER_EPOCH='1'
ARG DRIVER_OPEN='false'
ARG DRIVER_PATH='kernel'

ARG BUILDER_USER=''
ARG BUILDER_EMAIL=''
ARG DRIVER_STREAM_TYPE=''

# 配置 AlmaLinux Vault 仓库（旧版本内核包已从主仓库移除到 vault）
# 锁定 releasever 确保安装与目标系统匹配的内核包
RUN echo "${ALMALINUX_VERSION}" > /etc/dnf/vars/releasever \
    && sed -i 's|^mirrorlist=|#mirrorlist=|g' /etc/yum.repos.d/almalinux*.repo \
    && sed -i 's|^# baseurl=https://repo.almalinux.org/almalinux|baseurl=https://vault.almalinux.org|g' /etc/yum.repos.d/almalinux*.repo

# 安装编译内核模块所需的依赖，并安装目标内核模块用于复制 DRM 依赖
RUN dnf install -y \
    gcc \
    make \
    rpm-build \
    openssl \
    git \
    elfutils-libelf-devel \
    kernel-devel-${KERNEL_VERSION} \
    kernel-headers-${KERNEL_VERSION} \
    kernel-${KERNEL_VERSION} \
    && dnf clean all

RUN useradd -u 1001 -m -s /bin/bash builder

USER builder

WORKDIR /home/builder
COPY --chown=1001:0 x509-configuration.ini private_key.priv* public_key.der* /home/builder/

# 下载 NVIDIA 驱动，编译内核模块并打包为 RPM
RUN export KVER=$(echo ${KERNEL_VERSION} | cut -d '-' -f 1) \
        KREL=$(echo ${KERNEL_VERSION} | cut -d '-' -f 2 | sed 's/\.el._.\..\+$//') \
        KDIST=$(echo ${KERNEL_VERSION} | cut -d '-' -f 2 | sed 's/^.*\(\.el._.\)\..\+$/\1/') \
        DRIVER_STREAM=$(echo ${DRIVER_VERSION} | cut -d '.' -f 1) \
    && if [ "${DRIVER_OPEN}" == "true" ] ; then export DRIVER_PATH="${DRIVER_PATH}-open" ; fi \
    && git clone --depth 1 --single-branch -b rhel${RHEL_VERSION_MAJOR} https://github.com/NVIDIA/yum-packaging-precompiled-kmod \
    && cd yum-packaging-precompiled-kmod \
    && mkdir BUILD BUILDROOT RPMS SRPMS SOURCES SPECS \
    && mkdir nvidia-kmod-${DRIVER_VERSION}-${BUILD_ARCH} \
    && curl -sLOf ${BASE_URL}/${DRIVER_VERSION}/NVIDIA-Linux-${TARGET_ARCH}-${DRIVER_VERSION}.run \
    && sh ./NVIDIA-Linux-${TARGET_ARCH}-${DRIVER_VERSION}.run --extract-only --target tmp \
    && if [ "${DRIVER_STREAM_TYPE}" == "development" ] ; then \
        export DRIVER_OPEN='true' ; \
        git clone --depth 1 --single-branch -b ${DRIVER_VERSION} https://github.com/NVIDIA/open-gpu-kernel-modules ; \
        mv open-gpu-kernel-modules/${DRIVER_PATH} nvidia-kmod-${DRIVER_VERSION}-${BUILD_ARCH}/kernel ; \
    else \
        mv tmp/${DRIVER_PATH} nvidia-kmod-${DRIVER_VERSION}-${BUILD_ARCH}/kernel ; \
    fi \
    && tar -cJf SOURCES/nvidia-kmod-${DRIVER_VERSION}-${BUILD_ARCH}.tar.xz nvidia-kmod-${DRIVER_VERSION}-${BUILD_ARCH} \
    && mv kmod-nvidia.spec SPECS/ \
    && if test -f "/home/builder/private_key.priv" -a -f "/home/builder/public_key.der" ; then \
        mv /home/builder/private_key.priv /home/builder/public_key.der SOURCES ; \
    else \
        sed -i -e "s/\$USER/${BUILDER_USER}/" -e "s/\$EMAIL/${BUILDER_EMAIL}/" ${HOME}/x509-configuration.ini ; \
        openssl req -x509 -new -nodes -utf8 -sha256 -days 36500 -batch \
            -config ${HOME}/x509-configuration.ini \
            -outform DER -out SOURCES/public_key.der \
            -keyout SOURCES/private_key.priv ; \
    fi \
    && rpmbuild \
        --define "% _arch ${BUILD_ARCH}" \
        --define "%_topdir $(pwd)" \
        --define "debug_package %{nil}" \
        --define "kernel ${KVER}" \
        --define "kernel_release ${KREL}" \
        --define "kernel_dist ${KDIST}" \
        --define "driver ${DRIVER_VERSION}" \
        --define "epoch ${DRIVER_EPOCH}" \
        --define "driver_branch ${DRIVER_STREAM}" \
        -v -bb SPECS/kmod-nvidia.spec

# ============================================================
# 阶段二：Runtime - 构建最终驱动容器镜像
# ============================================================
FROM ${ALMALINUX_BASE_IMAGE}

ARG BASE_URL='https://us.download.nvidia.com/tesla'

ARG KERNEL_VERSION
ARG KERNEL_VERSION_NOARCH
ARG ALMALINUX_VERSION
ARG OS_TAG

ARG DRIVER_STREAM_TYPE

ARG DRIVER_TYPE=passthrough
ENV DRIVER_TYPE=${DRIVER_TYPE}

ARG DRIVER_VERSION
ENV DRIVER_VERSION=${DRIVER_VERSION}

ARG BUILD_ARCH
ARG TARGET_ARCH
ENV TARGETARCH=${TARGET_ARCH}

# 避免 container-toolkit 对驱动容器的依赖
ENV NVIDIA_VISIBLE_DEVICES=void

# 默认禁用 vGPU 版本兼容性检查
ARG DISABLE_VGPU_VERSION_CHECK=true
ENV DISABLE_VGPU_VERSION_CHECK=${DISABLE_VGPU_VERSION_CHECK}

USER root

COPY --chmod=744 nvidia-driver /usr/local/bin
COPY --chmod=744 common.sh /usr/local/bin

# 从 builder 阶段复制 GPU 固件文件
COPY --from=builder --chmod=444 /home/builder/yum-packaging-precompiled-kmod/tmp/firmware/*.bin /opt/lib/firmware/nvidia/${DRIVER_VERSION}/

# 安装运行时基础依赖
RUN dnf -y install kmod binutils dnf-plugins-core \
    && dnf clean all

# 创建内核模块目录结构，防止 modprobe 报警
RUN rm -rf /lib/modules/${KERNEL_VERSION_NOARCH}.${BUILD_ARCH} \
    && mkdir -p /lib/modules/${KERNEL_VERSION_NOARCH}.${BUILD_ARCH}/proc \
    && touch /lib/modules/${KERNEL_VERSION_NOARCH}.${BUILD_ARCH}/modules.order \
    && touch /lib/modules/${KERNEL_VERSION_NOARCH}.${BUILD_ARCH}/modules.builtin \
    && depmod ${KERNEL_VERSION_NOARCH}.${BUILD_ARCH}

# 从 builder 阶段复制 DRM 内核模块依赖
COPY --from=builder /lib/modules/${KERNEL_VERSION_NOARCH}.${BUILD_ARCH}/kernel/drivers/gpu/drm/drm.ko.xz /lib/modules/${KERNEL_VERSION_NOARCH}.${BUILD_ARCH}/kernel/drivers/gpu/drm/drm.ko.xz

# 从 builder 阶段复制预编译的 NVIDIA 驱动 RPM 并安装
COPY --from=builder /home/builder/yum-packaging-precompiled-kmod/RPMS/${TARGET_ARCH}/*.rpm /rpms/
RUN dnf install -y /rpms/kmod-nvidia-*.rpm \
    && dnf clean all \
    && rm -rf /rpms

# 配置 NVIDIA CUDA 公开仓库（替代 RHSM 订阅）
RUN NVIDIA_GPGKEY_SUM=d0664fbbdb8c32356d45de36c5984617217b2d0bef41b93ccecd326ba3b80c87 \
    && curl -fsSL "https://developer.download.nvidia.com/compute/cuda/repos/rhel9/${TARGET_ARCH}/D42D0685.pub" | sed '/^Version/d' > /etc/pki/rpm-gpg/RPM-GPG-KEY-NVIDIA \
    && echo "$NVIDIA_GPGKEY_SUM  /etc/pki/rpm-gpg/RPM-GPG-KEY-NVIDIA" | sha256sum -c --strict - \
    && dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel9/${TARGET_ARCH}/cuda-rhel9.repo

# 安装 NVIDIA 用户态驱动包（通过 NVIDIA 公开仓库，无需 RHSM）
RUN export DRIVER_STREAM=$(echo ${DRIVER_VERSION} | cut -d '.' -f 1) \
    && if [ "${DRIVER_STREAM_TYPE}" == "development" ] ; then \
        curl -sLOf ${BASE_URL}/${DRIVER_VERSION}/NVIDIA-Linux-${TARGET_ARCH}-${DRIVER_VERSION}.run \
        && sh ./NVIDIA-Linux-${TARGET_ARCH}-${DRIVER_VERSION}.run \
            --silent \
            --accept-license \
            --no-kernel-modules \
            --no-nvidia-modprobe \
            --no-rebuild-initramfs \
        && rm NVIDIA-Linux-${TARGET_ARCH}-${DRIVER_VERSION}.run ; \
    else \
        dnf -y module enable nvidia-driver:${DRIVER_STREAM}/default \
        && dnf install -y \
            nvidia-driver-cuda-${DRIVER_VERSION} \
            nvidia-driver-libs-${DRIVER_VERSION} \
            libnvidia-ml-${DRIVER_VERSION} \
            nvidia-persistenced-${DRIVER_VERSION} \
        && if [ "$DRIVER_TYPE" != "vgpu" ] && [ "$TARGETARCH" != "aarch64" ]; then \
            versionArray=(${DRIVER_VERSION//./ }); \
            DRIVER_BRANCH=${versionArray[0]}; \
            if [ "$DRIVER_BRANCH" -ge "580" ]; then \
                dnf install -y nvidia-fabricmanager-${DRIVER_VERSION} libnvidia-nscq-${DRIVER_VERSION} ; \
            else \
                dnf install -y nvidia-fabric-manager-${DRIVER_VERSION} libnvidia-nscq-${DRIVER_BRANCH}-${DRIVER_VERSION} ; \
            fi ; \
        fi \
        && dnf clean all ; \
    fi

# 安装/升级 CVE 修复包
ARG CVE_UPDATES
RUN if [ -n "${CVE_UPDATES}" ]; then \
        dnf update -y ${CVE_UPDATES}; \
    fi \
    && dnf clean all

LABEL io.k8s.display-name="NVIDIA Driver Container"
LABEL name="NVIDIA Driver Container"
LABEL vendor="NVIDIA"
LABEL version="${DRIVER_VERSION}"
LABEL release="${KERNEL_VERSION}-${OS_TAG}"
LABEL summary="Provision the NVIDIA driver through containers"
LABEL description="See summary"

# 创建 license 目录
RUN mkdir -p /licenses

# 移除 CUDA 仓库配置，避免运行时 GPG 错误
RUN rm -f /etc/yum.repos.d/cuda.repo

ENTRYPOINT ["nvidia-driver", "init"]
