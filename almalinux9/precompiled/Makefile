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

.PHONY: check-required image image-push

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
