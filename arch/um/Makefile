# 
# Copyright (C) 2002 Jeff Dike (jdike@karaya.com)
# Licensed under the GPL
#

ARCH_DIR := arch/um
OS := $(shell uname -s)
# We require bash because the vmlinux link and loader script cpp use bash
# features.
SHELL := /bin/bash

filechk_gen_header = $<

core-y			+= $(ARCH_DIR)/kernel/		\
			   $(ARCH_DIR)/drivers/		\
			   $(ARCH_DIR)/os-$(OS)/

# Have to precede the include because the included Makefiles reference them.
SYMLINK_HEADERS := archparam.h system.h sigcontext.h processor.h ptrace.h \
	arch-signal.h module.h vm-flags.h
SYMLINK_HEADERS := $(foreach header,$(SYMLINK_HEADERS),include/asm-um/$(header))

# XXX: The "os" symlink is only used by arch/um/include/os.h, which includes
# ../os/include/file.h
#
# These are cleaned up during mrproper. Please DO NOT fix it again, this is
# the Correct Thing(tm) to do!
ARCH_SYMLINKS = include/asm-um/arch $(ARCH_DIR)/include/sysdep $(ARCH_DIR)/os \
	$(SYMLINK_HEADERS) $(ARCH_DIR)/include/uml-config.h

GEN_HEADERS += $(ARCH_DIR)/include/task.h $(ARCH_DIR)/include/kern_constants.h

um-modes-$(CONFIG_MODE_TT) += tt
um-modes-$(CONFIG_MODE_SKAS) += skas

MODE_INCLUDE	+= $(foreach mode,$(um-modes-y),\
		   -I$(srctree)/$(ARCH_DIR)/kernel/$(mode)/include)

MAKEFILES-INCL	+= $(foreach mode,$(um-modes-y),\
		   $(srctree)/$(ARCH_DIR)/Makefile-$(mode))

ifneq ($(MAKEFILES-INCL),)
  include $(MAKEFILES-INCL)
endif

ARCH_INCLUDE	:= -I$(ARCH_DIR)/include
SYS_DIR		:= $(ARCH_DIR)/include/sysdep-$(SUBARCH)

include $(srctree)/$(ARCH_DIR)/Makefile-$(SUBARCH)

core-y += $(SUBARCH_CORE)
libs-y += $(SUBARCH_LIBS)

# -Derrno=kernel_errno - This turns all kernel references to errno into
# kernel_errno to separate them from the libc errno.  This allows -fno-common
# in CFLAGS.  Otherwise, it would cause ld to complain about the two different
# errnos.

CFLAGS += $(CFLAGS-y) -D__arch_um__ -DSUBARCH=\"$(SUBARCH)\" \
	$(ARCH_INCLUDE) $(MODE_INCLUDE)

USER_CFLAGS := $(patsubst -I%,,$(CFLAGS))
USER_CFLAGS := $(patsubst -D__KERNEL__,,$(USER_CFLAGS)) $(ARCH_INCLUDE) \
	$(MODE_INCLUDE) $(ARCH_USER_CFLAGS)
CFLAGS += -Derrno=kernel_errno -Dsigprocmask=kernel_sigprocmask
CFLAGS += $(call cc-option,-fno-unit-at-a-time,)

#This will adjust *FLAGS accordingly to the platform.
include $(srctree)/$(ARCH_DIR)/Makefile-os-$(OS)

# These are needed for clean and mrproper, since in that case .config is not
# included; the values here are meaningless

CONFIG_NEST_LEVEL ?= 0
CONFIG_KERNEL_HALF_GIGS ?= 0

SIZE = (($(CONFIG_NEST_LEVEL) + $(CONFIG_KERNEL_HALF_GIGS)) * 0x20000000)

ifeq ($(CONFIG_MODE_SKAS), y)
$(SYS_HEADERS) : $(ARCH_DIR)/include/skas_ptregs.h
endif

.PHONY: linux

all: linux

linux: vmlinux
	ln -f $< $@

define archhelp
  echo '* linux		- Binary kernel image (./linux) - for backward'
  echo '		   compatibility only, this creates a hard link to the'
  echo '		   real kernel binary, the the "vmlinux" binary you'
  echo '		   find in the kernel root.'
endef

$(shell cd $(ARCH_DIR) && ln -sf Kconfig_$(SUBARCH) Kconfig_arch)

prepare: $(ARCH_SYMLINKS) $(SYS_HEADERS) $(GEN_HEADERS) \
	$(ARCH_DIR)/kernel/vmlinux.lds.S

LINK-$(CONFIG_LD_SCRIPT_STATIC) += -static
LINK-$(CONFIG_LD_SCRIPT_DYN) += -Wl,-rpath,/lib

LD_SCRIPT-$(CONFIG_LD_SCRIPT_STATIC) := uml.lds.S
LD_SCRIPT-$(CONFIG_LD_SCRIPT_DYN) := dyn.lds.S

CPP_MODE-$(CONFIG_MODE_TT) := -DMODE_TT
CONFIG_KERNEL_STACK_ORDER ?= 2
STACK_SIZE := $(shell echo $$[ 4096 * (1 << $(CONFIG_KERNEL_STACK_ORDER)) ] )

ifndef START
  START = $$(($(TOP_ADDR) - $(SIZE)))
endif

CPPFLAGS_vmlinux.lds = $(shell echo -U$(SUBARCH) \
	-DSTART=$(START) -DELF_ARCH=$(ELF_ARCH) \
	-DELF_FORMAT=\"$(ELF_FORMAT)\" $(CPP_MODE-y) \
	-DKERNEL_STACK_SIZE=$(STACK_SIZE))

#The wrappers will select whether using "malloc" or the kernel allocator.
LINK_WRAPS = -Wl,--wrap,malloc -Wl,--wrap,free -Wl,--wrap,calloc

CFLAGS_vmlinux = $(LINK-y) $(LINK_WRAPS)
define cmd_vmlinux__
	$(CC) $(CFLAGS_vmlinux) -o $@ \
	-Wl,-T,$(vmlinux-lds) $(vmlinux-init) \
	-Wl,--start-group $(vmlinux-main) -Wl,--end-group \
	-L/usr/lib -lutil \
	$(filter-out $(vmlinux-lds) $(vmlinux-init) $(vmlinux-main) \
	FORCE ,$^) ; rm -f linux
endef

#When cleaning we don't include .config, so we don't include
#TT or skas makefiles and don't clean skas_ptregs.h.
CLEAN_FILES += linux x.i gmon.out $(ARCH_DIR)/include/uml-config.h \
	$(GEN_HEADERS) $(ARCH_DIR)/include/skas_ptregs.h

MRPROPER_FILES += $(SYMLINK_HEADERS) $(ARCH_SYMLINKS) \
	$(addprefix $(ARCH_DIR)/kernel/,$(KERN_SYMLINKS)) $(ARCH_DIR)/os \
	$(ARCH_DIR)/Kconfig_arch

archclean:
	$(Q)$(MAKE) $(clean)=$(ARCH_DIR)/util
	@find . \( -name '*.bb' -o -name '*.bbg' -o -name '*.da' \
		-o -name '*.gcov' \) -type f -print | xargs rm -f

#We need to re-preprocess this when the symlink dest changes.
#So we touch it when needed.
$(ARCH_DIR)/kernel/vmlinux.lds.S: FORCE
	$(Q)if [ "$(shell readlink $@)" != "$(LD_SCRIPT-y)" ]; then \
		echo '  SYMLINK $@'; \
		ln -sf $(LD_SCRIPT-y) $@; \
		touch $@; \
	fi;

$(SYMLINK_HEADERS):
	@echo '  SYMLINK $@'
	$(Q)cd $(TOPDIR)/$(dir $@) ; \
	ln -sf $(basename $(notdir $@))-$(SUBARCH)$(suffix $@) $(notdir $@)

include/asm-um/arch:
	@echo '  SYMLINK $@'
	$(Q)cd $(TOPDIR)/include/asm-um && ln -sf ../asm-$(SUBARCH) arch

$(ARCH_DIR)/include/sysdep:
	@echo '  SYMLINK $@'
	$(Q)cd $(ARCH_DIR)/include && ln -sf sysdep-$(SUBARCH) sysdep

$(ARCH_DIR)/os:
	@echo '  SYMLINK $@'
	$(Q)cd $(ARCH_DIR) && ln -sf os-$(OS) os

# Generated files
define filechk_umlconfig
	sed 's/ CONFIG/ UML_CONFIG/'
endef

$(ARCH_DIR)/include/uml-config.h : include/linux/autoconf.h
	$(call filechk,umlconfig)

$(ARCH_DIR)/include/task.h: $(ARCH_DIR)/util/mk_task
	$(call filechk,gen_header)

$(ARCH_DIR)/include/user_constants.h: $(ARCH_DIR)/os/util/mk_user_constants
	$(call filechk,gen_header)

$(ARCH_DIR)/include/kern_constants.h: $(ARCH_DIR)/util/mk_constants
	$(call filechk,gen_header)

$(ARCH_DIR)/include/skas_ptregs.h: $(ARCH_DIR)/kernel/skas/util/mk_ptregs
	$(call filechk,gen_header)

$(ARCH_DIR)/os/util/mk_user_constants: $(ARCH_DIR)/os/util FORCE ;

$(ARCH_DIR)/util/mk_task $(ARCH_DIR)/util/mk_constants: $(ARCH_DIR)/include/user_constants.h $(ARCH_DIR)/util \
	FORCE ;

$(ARCH_DIR)/kernel/skas/util/mk_ptregs: $(ARCH_DIR)/kernel/skas/util FORCE ;

$(ARCH_DIR)/util: scripts_basic $(SYS_DIR)/sc.h FORCE
	$(Q)$(MAKE) $(build)=$@

$(ARCH_DIR)/kernel/skas/util: scripts_basic FORCE
	$(Q)$(MAKE) $(build)=$@

$(ARCH_DIR)/os/util: scripts_basic FORCE
	$(Q)$(MAKE) $(build)=$@

export SUBARCH USER_CFLAGS OS
