# ============================
#   Toolchain Build Makefile
#   Author: TurBoTse
# ============================

TOPDIR      := $(CURDIR)
CT_TARGET   := mipsel-linux-musl
CT_PREFIX   := $(TOPDIR)/$(CT_TARGET)
CT_NG       := ./ct-ng

.PHONY: all build clean

all: build

build:
	@echo "\033[1;36m===> 🔧 Starting toolchain build for:\033[0m \033[1;34m$(CT_TARGET)\033[0m"
	@if [ ! -x "$(CT_NG)" ]; then \
		echo "\033[1;33m===> ⏳ Bootstrapping crosstool-ng...\033[0m"; \
		./bootstrap && ./configure --enable-local && make || { \
			echo "\033[1;31m===> ❌ Failed to bootstrap ct-ng. Check dependencies or permissions.\033[0m"; exit 1; }; \
	fi
	@echo "\033[1;36m===> 📦 Configuring target: $(CT_TARGET)\033[0m"
	@$(CT_NG) $(CT_TARGET) || { echo "\033[1;31m===> ❌ Failed to configure $(CT_TARGET).\033[0m"; exit 1; }
	@echo "\033[1;32m===> 🚧 Building toolchain...\033[0m"
	@$(CT_NG) build || { echo "\033[1;31m===> ❌ Toolchain build failed.\033[0m"; exit 1; }
	@echo "\033[1;32m===> ✅ Toolchain build completed at:\033[0m \033[1;34m$(CT_PREFIX)\033[0m"

clean:
	@echo "\033[1;36m===> 🧹 Cleaning build artifacts...\033[0m"
	@if [ -x "$(CT_NG)" ]; then $(CT_NG) distclean || { echo "\033[1;31m===> ❌ Failed to run ct-ng distclean.\033[0m"; exit 1; }; fi
	@if [ -f Makefile ]; then make distclean || { echo "\033[1;31m===> ❌ Failed to run make distclean.\033[0m"; exit 1; }; fi
	@if [ -d "$(CT_PREFIX)" ]; then rm -rf "$(CT_PREFIX)"; fi
	@echo "\033[1;32m===> ♻️  Clean completed.\033[0m"
