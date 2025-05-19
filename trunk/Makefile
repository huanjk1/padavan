############################################################################
# Top-Level Makefile for Padavan-based Firmware
#
# Copyright (c) 2001-2004, SnapGear (www.snapgear.com)
# Copyright (c) 2001, Lineo
# Copyright (c) 2025, TurBoTse
############################################################################

# === Configuration Check ===
ifeq ($(wildcard .config),)
  $(error .config file not found, please run ./configure first)
endif
include .config
include versions.inc

# === Directory Layout ===
ROOTDIR         := $(CURDIR)
LINUXDIR        := $(CONFIG_LINUXDIR)
LINUXINCDIR     := $(ROOTDIR)/$(LINUXDIR)/include
SCRIPTSDIR      := $(ROOTDIR)/config/scripts
ROMFSDIR        := $(ROOTDIR)/romfs
STAGEDIR        := $(ROOTDIR)/stage
IMAGEDIR        := $(ROOTDIR)/images
MERGE_DIR       := $(ROOTDIR)/merge
MERGE_ROOTFS    := $(MERGE_DIR)/$(CONFIG_FIRMWARE_PRODUCT_ID)/rootfs
LINUX_CONFIG    := $(ROOTDIR)/$(LINUXDIR)/.config
PROJECT_CONFIG  := $(ROOTDIR)/.config
TFTPDIR         := $(or $(CONFIG_TFTPDIR),/tftpboot)

# === Host Environment ===(override with make JOBS=N)
HOST_NCPU       := $(shell nproc 2>/dev/null || echo 1)
JOBS            ?= $(HOST_NCPU)
CONFIG_SHELL    := $(shell which bash 2>/dev/null || echo /bin/sh)

# === Toolchain ===
CONFIG_CROSS_COMPILER_PATH  := $(CONFIG_CROSS_COMPILER_ROOT)/bin
CONFIG_CMAKE_TOOLCHAIN_FILE := $(ROOTDIR)/vendors/cmake/cross-mipsel-linux.cmake
KERNEL_HEADERS_PATH         := $(CONFIG_CROSS_COMPILER_ROOT)/$(CONFIG_CROSS_COMPILE)/sysroot/usr/include

# === Tools ===
TOOLS_DIR       := $(ROOTDIR)/tools
DEPMOD          := $(TOOLS_DIR)/depmod.sh
ROMFSINST       := $(TOOLS_DIR)/romfs-inst.sh
UNPACKTRX       := $(TOOLS_DIR)/unpack-trx.sh
STRIPROMFS      := $(TOOLS_DIR)/strip-romfs.sh
SSTRIP_TOOL     := $(if $(filter y,$(CONFIG_FIRMWARE_PERFORM_SSTRIP)),$(TOOLS_DIR)/sstrip/sstrip)
LZMA_TOOL       := $(TOOLS_DIR)/lzma/lzma
MKIMAGE_TOOL    := $(TOOLS_DIR)/mkimage/mkimage
MKSQUASHFS_TOOL := $(TOOLS_DIR)/mksquashfs_xz/mksquashfs
UNSQSHFS_TOOL   := $(TOOLS_DIR)/mksquashfs_xz/unsquashfs

# === Architecture Configuration ===
ifeq ($(wildcard config.arch),config.arch)
  ifeq ($(filter %_default,$(MAKECMDGOALS)),)
    include config.arch
    export ARCH_CONFIG := $(ROOTDIR)/config.arch
  endif
endif

# === Kernel Build Commands ===
KERNEL_CROSS_COMPILE ?= $(CONFIG_CROSS_COMPILE)-
ifneq ($(SUBARCH),)
  MAKEARCH        := $(MAKE) ARCH=$(SUBARCH) CROSS_COMPILE=$(CROSS_COMPILE)
  MAKEARCH_KERNEL := $(MAKE) ARCH=$(ARCH) SUBARCH=$(SUBARCH) CROSS_COMPILE=$(KERNEL_CROSS_COMPILE)
else
  MAKEARCH        := $(MAKE) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE)
  MAKEARCH_KERNEL := $(MAKE) ARCH=$(ARCH) CROSS_COMPILE=$(KERNEL_CROSS_COMPILE)
endif

# === ccache Setup ===
ifeq ($(CONFIG_CCACHE),y)
  CCACHE_WRAPPER_DIR := $(CONFIG_CROSS_COMPILER_ROOT)/ccache-wrappers
  CCACHE_BIN := $(shell command -v ccache)
  ifeq ($(CCACHE_BIN),)
    $(warning ccache not found, disabling CONFIG_CCACHE)
    CONFIG_CCACHE := n
  else
    export CCACHE_DIR := $(ROOTDIR)/.ccache
    export CCACHE_SLOPPINESS := pch_defines,time_macros
    WRAPPERS := $(addprefix $(CROSS_COMPILE), gcc g++ cc c++)
    $(shell mkdir -p $(CCACHE_WRAPPER_DIR); \
      for bin in $(WRAPPERS); do \
        [ -e "$(CCACHE_WRAPPER_DIR)/$$bin" ] || ln -sf $(CCACHE_BIN) "$(CCACHE_WRAPPER_DIR)/$$bin"; \
      done)
    PATH := $(CCACHE_WRAPPER_DIR):$(CONFIG_CROSS_COMPILER_PATH):$(PATH):$(TOOLS_DIR)
  endif
else
  PATH := $(CONFIG_CROSS_COMPILER_PATH):$(PATH):$(TOOLS_DIR)
endif
export PATH

# === Exported Variables ===
export \
  VENDOR PRODUCT CONFIG_SHELL HOST_NCPU JOBS CONFIG_WITHOUT_KERNEL \
  CONFIG_CROSS_COMPILER_ROOT CONFIG_CROSS_COMPILER_PATH CONFIG_CROSS_COMPILE CONFIG_CMAKE_TOOLCHAIN_FILE KERNEL_HEADERS_PATH \
  SSTRIP_TOOL LZMA_TOOL MKIMAGE_TOOL MKSQUASHFS_TOOL UNSQSHFS_TOOL \
  PROJECT_CONFIG LINUX_CONFIG ROOTDIR LINUXDIR ROMFSDIR SCRIPTSDIR IMAGEDIR RELFILES TFTPDIR \
  FIRMWARE_KERNEL_VER FIRMWARE_ROOTFS_VER FIRMWARE_BUILDS_VER FIRMWARE_BUILDS_REV VERSIONPKG VERSIONSTR \
  ROMFSINST UNPACKTRX

# === Build Targets ===
DIRS := libc libs user

.PHONY: all config dep tools unpack_trx linux modules modules_install modules_copy bin_copy romfs romfs.subdirs romfs.post image \
        sparse sparseall subdirs clean tools_clean kernel_clean \
        %_romfs %_only %_clean

# === Default Build Flow ===
ifeq ($(CONFIG_WITHOUT_KERNEL),y)
all: dep tools unpack_trx libc_only libs_only user_only romfs image
else
all: dep tools linux libc_only libs_only user_only romfs image
endif

# === Configuration and Dependencies ===
config:
	@echo "\033[1;34m===> 🛠️ Configuring build environment...\033[0m"
	@if [ ! -d "$(CONFIG_CROSS_COMPILER_PATH)" ] || [ ! -x "$(CONFIG_CROSS_COMPILER_PATH)/$(CC)" ]; then \
		echo "\033[1;31m===> ❌ Cross-compiler path $(CONFIG_CROSS_COMPILER_PATH) invalid or compiler not found\033[0m"; \
		exit 1; \
	fi
	@./configure || { echo "\033[1;31m===> ❌ Configuration failed\033[0m"; exit 1; }
	@echo "\033[1;33m===> CONFIG_CCACHE=$(CONFIG_CCACHE)\033[0m"
	@if [ "$(CONFIG_CCACHE)" = "y" ]; then \
		echo "\033[1;32m===> ✅ ccache enabled: Checking ccache wrappers...\033[0m"; \
		mkdir -p "$(CCACHE_WRAPPER_DIR)"; \
		for bin in $(WRAPPERS); do \
			target="$(CCACHE_WRAPPER_DIR)/$$bin"; \
			if [ -e "$$target" ]; then \
				echo "\033[1;33m→ Wrapper: $$target is correct, skipping.\033[0m"; \
			else \
				echo "\033[1;33m→ Creating wrapper: $$target → $(CCACHE_BIN)\033[0m"; \
				ln -sf "$(CCACHE_BIN)" "$$target"; \
			fi; \
		done; \
	fi

dep: config
	@echo "\033[1;34m===> 🔍 Preparing kernel configuration...\033[0m"
	@$(MAKEARCH_KERNEL) -C $(LINUXDIR) olddefconfig
	@$(MAKEARCH_KERNEL) -C $(LINUXDIR) modules_prepare

# === Tool Build ===
tools:
	@echo "\033[1;34m===> 📦 Building tools utilities...\033[0m"
	@$(MAKE) -C tools -s || { echo "\033[1;31m===> ❌ Tools build failed\033[0m"; exit 1; }
	@for tool in $(LZMA_TOOL) $(MKIMAGE_TOOL) $(MKSQUASHFS_TOOL) $(UNSQSHFS_TOOL); do \
		if [ ! -x "$$tool" ]; then \
			echo "\033[1;31m===> ❌ Tool $$tool not found or not executable\033[0m"; \
			exit 1; \
		fi; \
	done

# === Firmware Extraction ===
unpack_trx:
	@echo "\033[1;36m===> 📦 Extracting TRX firmware to merge/ directory...\033[0m"
	@$(UNPACKTRX) || { echo "\033[1;31m===> ❌ TRX unpack failed\033[0m"; exit 1; }
	@echo "\033[1;32m===> ✅ TRX firmware unpacked.\033[0m"

# === Kernel Build ===
linux:
	@echo "\033[1;30;43m BUILD \033[0m \033[1;30;44m Linux kernel...\033[0m"
	@$(MAKEARCH_KERNEL) -j$(JOBS) -C $(LINUXDIR) $(LINUXTARGET) -s 2> $(LINUXDIR)/kernel_build.log || \
	{ echo "\033[1;31m===> ❌ Kernel build failed, check $(LINUXDIR)/kernel_build.log\033[0m"; exit 1; }
	@[ -f $(LINUXDIR)/vmlinux ] && ln -f $(LINUXDIR)/vmlinux $(LINUXDIR)/linux

# === Kernel Modules ===
modules:
	@echo "\033[1;34m===> 📦 Building kernel modules...\033[0m"
	@if grep -q 'CONFIG_MODULES=y' $(LINUX_CONFIG); then \
		mkdir -p $(LINUXDIR)/modules; \
		$(MAKEARCH_KERNEL) -j$(JOBS) -C $(LINUXDIR) modules 2> $(LINUXDIR)/modules_build.log || \
		{ echo "\033[1;31m===> ❌ Module build failed, check $(LINUXDIR)/modules_build.log\033[0m"; exit 1; }; \
	else \
		echo "\033[1;33m===> ⚠️  Module support disabled\033[0m"; \
	fi

modules_install:
	@echo "\033[1;34m===> 📦 Installing kernel modules into ROMFS...\033[0m"
	@if grep -q 'CONFIG_MODULES=y' $(LINUX_CONFIG); then \
		mkdir -p $(ROMFSDIR)/lib/modules; \
		$(MAKEARCH_KERNEL) -C $(LINUXDIR) INSTALL_MOD_PATH=$(ROMFSDIR) $(DEPMOD) modules_install 2> $(LINUXDIR)/modules_install.log || \
		{ echo "\033[1;31m===> ❌ Module installation failed, check $(LINUXDIR)/modules_install.log\033[0m"; exit 1; }; \
	else \
		echo "\033[1;33m===> ⚠️  Module support disabled\033[0m"; \
	fi

modules_copy:
	@echo "\033[1;34m===> 📦 Copying merged modules...\033[0m"
	@if [ ! -d "$(MERGE_ROOTFS)/lib/modules" ]; then \
		echo "\033[1;33m===> ⚠️  No modules directory found at $(MERGE_ROOTFS)/lib/modules\033[0m"; \
	else \
		mkdir -p $(ROMFSDIR)/lib/modules; \
		cp -rfa $(MERGE_ROOTFS)/lib/modules/. $(ROMFSDIR)/lib/modules/; \
	fi

# === Binary Copy ===
bin_copy:
	@echo "\033[1;34m===> 📦 Copying merged binaries...\033[0m"
	@if [ ! -f "$(MERGE_DIR)/file_list" ]; then \
		echo "\033[1;33m===> ⚠️  No binary file list found at $(MERGE_DIR)/file_list\033[0m"; \
	else \
		grep -v '^#' $(MERGE_DIR)/file_list | while read -r f; do \
			if [ -f "$(MERGE_ROOTFS)/$$f" ]; then \
				mkdir -p $(dir $(ROMFSDIR)/$$f); \
				cp -fP $(MERGE_ROOTFS)/$$f $(ROMFSDIR)/$$f; \
			else \
				echo "\033[1;33m===> ⚠️  Skipping missing file: $$f\033[0m"; \
			fi; \
		done; \
	fi

# === ROMFS Build ===
ifeq ($(CONFIG_WITHOUT_KERNEL),y)
romfs: romfs.subdirs modules_copy bin_copy romfs.post
else
romfs: romfs.subdirs modules_install romfs.post
endif

romfs.subdirs: $(patsubst %,%_romfs,vendors $(DIRS))
	@echo "\033[1;32m===> ✅ ROMFS subdirectories completed\033[0m"

romfs.post:
	@echo "\033[1;34m===> 🧩 Post-processing ROMFS...\033[0m"
	@-find $(ROMFSDIR) -name CVS -exec rm -rf {} +
	@$(STRIPROMFS) || { echo "\033[1;31m===> ❌ ROMFS stripping failed!\033[0m"; exit 1; }
	@$(MAKEARCH) -C vendors romfs.post || { echo "\033[1;31m===> ❌ Vendors romfs.post failed\033[0m"; exit 1; }

# === Firmware Image ===
image:
	@echo "\033[1;34m===> 🧱 Generating firmware image...\033[0m"
	@mkdir -p $(IMAGEDIR)
	@$(MAKEARCH) -C vendors image 2> $(IMAGEDIR)/image_build.log || \
	{ echo "\033[1;31m===> ❌ Image generation failed, check $(IMAGEDIR)/image_build.log\033[0m"; exit 1; }

# === Sparse Checking ===
sparse:
	@echo "\033[1;34m===> 🔍 Performing sparse check...\033[0m"
	@$(MAKEARCH_KERNEL) -C $(LINUXDIR) C=1 $(LINUXTARGET) || exit 1

sparseall:
	@echo "\033[1;34m===> 🔍 Performing full sparse check...\033[0m"
	@$(MAKEARCH_KERNEL) -C $(LINUXDIR) C=2 $(LINUXTARGET) || exit 1

# === Subdirectory Build ===
subdirs: $(patsubst %,%_only,$(DIRS))

# === Clean Targets ===
clean: $(patsubst %,%_clean,tools kernel $(DIRS))
	@echo "\033[1;33m===> 🧹 Cleaning build artifacts...\033[0m"
	@rm -rf $(STAGEDIR) $(ROMFSDIR) $(IMAGEDIR) $(MERGE_DIR) $(CCACHE_WRAPPER_DIR)
	@rm -rf $(LINUXDIR)/net/ipsec/alg/libaes $(LINUXDIR)/net/ipsec/alg/perlasm
	@rm -f $(LINUXDIR)/arch/mips/ramdisk/*.gz $(LINUXDIR)/linux
	@echo "\033[1;32m===> ✅ Clean completed!\033[0m"

tools_clean:
	@echo "\033[1;33m===> 🧹 Cleaning tools...\033[0m"
	@$(MAKE) -C tools clean

kernel_clean:
	@echo "\033[1;33m===> 🧹 Cleaning kernel source...\033[0m"
	@$(MAKEARCH_KERNEL) -C $(LINUXDIR) distclean

# === Generic Rules ===
%_romfs:
	@case "$(@)" in \
	*/*) d=`expr $(@) : '\([^/]*\)/.*'`; \
	     t=`expr $(@) : '[^/]*/\(.*\)'`; \
	     $(MAKEARCH) -j$(JOBS) -C $$d $$t;; \
	*)   $(MAKEARCH) -j$(JOBS) -C $(@:_romfs=) romfs;; \
	esac

%_only:
	@case "$(@)" in \
	*/*) d=`expr $(@) : '\([^/]*\)/.*'`; \
	     t=`expr $(@) : '[^/]*/\(.*\)'`; \
	     $(MAKEARCH) -C $$d $$t;; \
	*)   $(MAKEARCH) -C $(@:_only=);; \
	esac

%_clean:
	@case "$(@)" in \
	*/*) d=`expr $(@) : '\([^/]*\)/.*'`; \
	     t=`expr $(@) : '[^/]*/\(.*\)'`; \
	     $(MAKEARCH) -C $$d $$t;; \
	*)   $(MAKEARCH) -C $(@:_clean=) clean;; \
	esac
