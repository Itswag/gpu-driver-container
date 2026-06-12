# AlmaLinux 9.0 GPU 预编译驱动容器 - 构建配置
# 使用方法:
#   make image DRIVER_VERSION=580.159.04 KERNEL_VERSION=5.14.0-70.13.1.el9_0.x86_64

ALMALINUX_VERSION ?= 9.0
ALMALINUX_BASE_IMAGE ?= almalinux:$(ALMALINUX_VERSION)
RHEL_VERSION_MAJOR = $(shell echo "$(ALMALINUX_VERSION)" | awk -F. '{print $$1}')

KERNEL_VERSION ?=
BUILD_ARCH ?= x86_64
TARGET_ARCH ?= $(shell echo "$(BUILD_ARCH)" | sed "s/+64k//")

KERNEL_VERSION_NOARCH = $(shell echo "$(KERNEL_VERSION)" | sed "s/\.${TARGET_ARCH}//")
KERNEL_VERSION_TAG = $(shell echo "$(KERNEL_VERSION_NOARCH).$(BUILD_ARCH)" | sed "s/+/_/")
BASE_URL ?= https://us.download.nvidia.com/tesla

DRIVER_VERSION ?=
DRIVER_EPOCH ?= 1
DRIVER_TYPE ?= passthrough
DRIVER_OPEN ?= false
DRIVER_STREAM_TYPE ?=

OS_TAG ?= almalinux$(ALMALINUX_VERSION)
CONTAINER_TOOL ?= docker
DOCKERFILE ?= Dockerfile
DOCKER_BUILD_EXTRA_ARGS ?=

IMAGE_REGISTRY ?= nvcr.io/ea-cnt/nv_only
ifeq ($(DRIVER_TYPE),vgpu)
	IMAGE_NAME ?= vgpu-guest-driver
else
	IMAGE_NAME ?= driver
endif

BUILDER_USER ?= $(shell git config --get user.name)
BUILDER_EMAIL ?= $(shell git config --get user.email)

.PHONY: check-required image image-push rpm-export

check-required:
	@test -n "$(DRIVER_VERSION)" || { echo "DRIVER_VERSION is required"; exit 1; }
	@test -n "$(KERNEL_VERSION)" || { echo "KERNEL_VERSION is required"; exit 1; }

# 构建预编译驱动容器镜像
image: check-required
	@echo "!=== Building image ${IMAGE_REGISTRY}/${IMAGE_NAME}:${DRIVER_VERSION}-${KERNEL_VERSION_TAG}-${OS_TAG} ===!"
	DOCKER_BUILDKIT=1 ${CONTAINER_TOOL} build \
		${DOCKER_BUILD_EXTRA_ARGS} \
		--build-arg ALMALINUX_BASE_IMAGE=${ALMALINUX_BASE_IMAGE} \
		--build-arg ALMALINUX_VERSION=${ALMALINUX_VERSION} \
		--build-arg RHEL_VERSION_MAJOR=${RHEL_VERSION_MAJOR} \
		--build-arg BUILD_ARCH=${BUILD_ARCH} \
		--build-arg TARGET_ARCH=${TARGET_ARCH} \
		--build-arg KERNEL_VERSION=${KERNEL_VERSION} \
		--build-arg KERNEL_VERSION_NOARCH=${KERNEL_VERSION_NOARCH} \
		--build-arg DRIVER_VERSION=${DRIVER_VERSION} \
		--build-arg DRIVER_EPOCH=${DRIVER_EPOCH} \
		--build-arg BUILDER_USER="${BUILDER_USER}" \
		--build-arg BUILDER_EMAIL=${BUILDER_EMAIL} \
		--build-arg DRIVER_OPEN=${DRIVER_OPEN} \
		--build-arg DRIVER_TYPE=${DRIVER_TYPE} \
		--build-arg DRIVER_STREAM_TYPE=${DRIVER_STREAM_TYPE} \
		--build-arg BASE_URL=${BASE_URL} \
		--build-arg OS_TAG=${OS_TAG} \
		--tag ${IMAGE_REGISTRY}/${IMAGE_NAME}:${DRIVER_VERSION}-${KERNEL_VERSION_TAG}-${OS_TAG} \
		--progress=plain \
		--file ${DOCKERFILE} .

image-push:
	@echo "!=== Pushing image ===!"
	${CONTAINER_TOOL} push \
		${IMAGE_REGISTRY}/${IMAGE_NAME}:${DRIVER_VERSION}-${KERNEL_VERSION_TAG}-${OS_TAG}

# 导出完整 NVIDIA 驱动离线安装包（kmod RPM + 用户态 RPM + 安装脚本）
# 用法: make rpm-export DRIVER_VERSION=580.159.04 KERNEL_VERSION=5.14.0-70.13.1.el9_0.x86_64
# 产出: ${RPM_OUTPUT_DIR}/ 目录，包含所有 RPM 和 install.sh
RPM_OUTPUT_DIR ?= ./rpms
BUILDER_TAG = kmod-builder-${DRIVER_VERSION}-${KERNEL_VERSION_TAG}
BUNDLER_TAG = rpm-bundler-${DRIVER_VERSION}-${KERNEL_VERSION_TAG}
DRIVER_STREAM = $(shell echo "${DRIVER_VERSION}" | cut -d. -f1)

rpm-export: check-required
	@echo "!=== Step 1/3: Building kmod-nvidia RPM ===!"
	DOCKER_BUILDKIT=1 ${CONTAINER_TOOL} build \
		${DOCKER_BUILD_EXTRA_ARGS} \
		--target builder \
		--build-arg ALMALINUX_BASE_IMAGE=${ALMALINUX_BASE_IMAGE} \
		--build-arg ALMALINUX_VERSION=${ALMALINUX_VERSION} \
		--build-arg RHEL_VERSION_MAJOR=${RHEL_VERSION_MAJOR} \
		--build-arg BUILD_ARCH=${BUILD_ARCH} \
		--build-arg TARGET_ARCH=${TARGET_ARCH} \
		--build-arg KERNEL_VERSION=${KERNEL_VERSION} \
		--build-arg DRIVER_VERSION=${DRIVER_VERSION} \
		--build-arg DRIVER_EPOCH=${DRIVER_EPOCH} \
		--build-arg BUILDER_USER="${BUILDER_USER}" \
		--build-arg BUILDER_EMAIL=${BUILDER_EMAIL} \
		--build-arg DRIVER_OPEN=${DRIVER_OPEN} \
		--build-arg DRIVER_STREAM_TYPE=${DRIVER_STREAM_TYPE} \
		--build-arg BASE_URL=${BASE_URL} \
		--tag ${BUILDER_TAG} \
		--progress=plain \
		--file ${DOCKERFILE} .
	@echo "!=== Step 2/3: Downloading userspace NVIDIA RPMs ===!"
	-${CONTAINER_TOOL} rm ${BUNDLER_TAG}
	${CONTAINER_TOOL} run --name ${BUNDLER_TAG} ${ALMALINUX_BASE_IMAGE} bash -c "\
		dnf install -y dnf-plugins-core \
		&& printf '[cuda-rhel9]\nname=NVIDIA CUDA for RHEL9\nbaseurl=https://developer.download.nvidia.com/compute/cuda/repos/rhel9/${TARGET_ARCH}/\nenabled=1\ngpgcheck=0\n' > /etc/yum.repos.d/cuda-rhel9.repo \
		&& dnf -y module enable nvidia-driver:${DRIVER_STREAM}/default \
		&& dnf download --destdir=/rpms \
			nvidia-driver-${DRIVER_VERSION} \
			nvidia-driver-cuda-${DRIVER_VERSION} \
			nvidia-driver-libs-${DRIVER_VERSION} \
			libnvidia-ml-${DRIVER_VERSION} \
			nvidia-persistenced-${DRIVER_VERSION} \
			nvidia-kmod-common-${DRIVER_VERSION} \
		&& dnf download --resolve --destdir=/rpms kmod \
		&& echo 'Download complete'"
	@echo "!=== Step 3/3: Exporting all RPMs ===!"
	-${CONTAINER_TOOL} rm ${BUILDER_TAG}-tmp
	${CONTAINER_TOOL} create --name ${BUILDER_TAG}-tmp ${BUILDER_TAG}
	${CONTAINER_TOOL} cp ${BUNDLER_TAG}:/rpms/. ${RPM_OUTPUT_DIR}/
	${CONTAINER_TOOL} cp ${BUILDER_TAG}-tmp:/home/builder/yum-packaging-precompiled-kmod/RPMS/${TARGET_ARCH}/. ${RPM_OUTPUT_DIR}/
	${CONTAINER_TOOL} rm ${BUILDER_TAG}-tmp
	${CONTAINER_TOOL} rm ${BUNDLER_TAG}
	@echo "!=== Complete! Offline bundle exported to ${RPM_OUTPUT_DIR}/ ===!"
	@echo ""
	@echo "On target machine, run:"
	@echo "  cd ${RPM_OUTPUT_DIR} && dnf install -y ./*.rpm"
	@echo "  modprobe nvidia && nvidia-smi"

