# SPDX-License-Identifier: (LGPL-2.1 OR BSD-2-Clause)
OUTPUT := $(abspath .output)
CLANG ?= clang
LLVM_STRIP ?= llvm-strip
BPFTOOL_SRC := $(abspath ./bpftool/src)
BPFTOOL ?= $(OUTPUT)/bpftool/bpftool
LIBBPF_SRC := $(abspath ../src/cc/libbpf/src)
LIBBPF_OBJ := $(abspath $(OUTPUT)/libbpf.a)
INCLUDES := -I$(OUTPUT) -I../src/cc/libbpf/include/uapi
CFLAGS := -g -O2 -Wall
BPFCFLAGS := -g -O2 -Wall
INSTALL ?= install
prefix ?= /usr/local
ARCH := $(shell uname -m | sed 's/x86_64/x86/' | sed 's/aarch64/arm64/' | sed 's/ppc64le/powerpc/' | sed 's/mips.*/mips/')
BTFHUB_ARCHIVE ?= $(abspath btfhub-archive)

ifeq ($(wildcard $(ARCH)/),)
$(error Architecture $(ARCH) is not supported yet. Please open an issue)
endif

APPS = \
	bashreadline \
	bindsnoop \
	biolatency \
	biopattern \
	biosnoop \
	biostacks \
	bitesize \
	cachestat \
	cpudist \
	cpufreq \
	drsnoop \
	execsnoop \
	exitsnoop \
	filelife \
	filetop \
	fsdist \
	fsslower \
	funclatency \
	gethostlatency \
	hardirqs \
	klockstat \
	ksnoop \
	llcstat \
	mountsnoop \
	numamove \
	offcputime \
	oomkill \
	opensnoop \
	readahead \
	runqlat \
	runqlen \
	runqslower \
	softirqs \
	solisten \
	statsnoop \
	syscount \
	tcpconnect \
	tcpconnlat \
	tcprtt \
	tcpsynbl \
	vfsstat \
	#

# export variables that are used in Makefile.btfgen as well.
export OUTPUT BPFTOOL ARCH BTFHUB_ARCHIVE APPS

FSDIST_ALIASES = btrfsdist ext4dist nfsdist xfsdist
FSSLOWER_ALIASES = btrfsslower ext4slower nfsslower xfsslower
APP_ALIASES = $(FSDIST_ALIASES) $(FSSLOWER_ALIASES)

COMMON_OBJ = \
	$(OUTPUT)/trace_helpers.o \
	$(OUTPUT)/syscall_helpers.o \
	$(OUTPUT)/errno_helpers.o \
	$(OUTPUT)/map_helpers.o \
	$(OUTPUT)/uprobe_helpers.o \
	$(OUTPUT)/btf_helpers.o \
	$(if $(ENABLE_MIN_CORE_BTFS),$(OUTPUT)/min_core_btf_tar.o) \
	#

.PHONY: all
all: $(APPS) $(APP_ALIASES)

ifeq ($(V),1)
Q =
msg =
else
Q = @
msg = @printf '  %-8s %s%s\n' "$(1)" "$(notdir $(2))" "$(if $(3), $(3))";
MAKEFLAGS += --no-print-directory
endif

.PHONY: clean
clean:
	$(call msg,CLEAN)
	$(Q)rm -rf $(OUTPUT) $(APPS) $(APP_ALIASES)

$(OUTPUT) $(OUTPUT)/libbpf:
	$(call msg,MKDIR,$@)
	$(Q)mkdir -p $@

.PHONY: bpftool
bpftool:
	$(Q)mkdir -p $(OUTPUT)/bpftool
	$(Q)$(MAKE) OUTPUT=$(OUTPUT)/bpftool/ -C $(BPFTOOL_SRC)

$(APPS): %: $(OUTPUT)/%.o $(LIBBPF_OBJ) $(COMMON_OBJ) | $(OUTPUT)
	$(call msg,BINARY,$@)
	$(Q)$(CC) $(CFLAGS) $^ $(LDFLAGS) -lelf -lz -o $@

$(patsubst %,$(OUTPUT)/%.o,$(APPS)): %.o: %.skel.h

$(OUTPUT)/%.o: %.c $(wildcard %.h) $(LIBBPF_OBJ) | $(OUTPUT)
	$(call msg,CC,$@)
	$(Q)$(CC) $(CFLAGS) $(INCLUDES) -c $(filter %.c,$^) -o $@

$(OUTPUT)/%.skel.h: $(OUTPUT)/%.bpf.o | $(OUTPUT) bpftool
	$(call msg,GEN-SKEL,$@)
	$(Q)$(BPFTOOL) gen skeleton $< > $@

$(OUTPUT)/%.bpf.o: %.bpf.c $(LIBBPF_OBJ) $(wildcard %.h) $(ARCH)/vmlinux.h | $(OUTPUT)
	$(call msg,BPF,$@)
	$(Q)$(CLANG) $(BPFCFLAGS) -target bpf -D__TARGET_ARCH_$(ARCH)	      \
		     -I$(ARCH)/ $(INCLUDES) -c $(filter %.c,$^) -o $@ &&      \
	$(LLVM_STRIP) -g $@

btfhub-archive: force
	$(call msg,GIT,$@)
	$(Q)[ -d "$(BTFHUB_ARCHIVE)" ] || git clone -q https://github.com/aquasecurity/btfhub-archive/ $(BTFHUB_ARCHIVE)
	$(Q)cd $(BTFHUB_ARCHIVE) && git pull

ifdef ENABLE_MIN_CORE_BTFS
$(OUTPUT)/min_core_btf_tar.o: $(patsubst %,$(OUTPUT)/%.bpf.o,$(APPS)) btfhub-archive | bpftool
	$(Q)$(MAKE) -f Makefile.btfgen
endif

# Build libbpf.a
$(LIBBPF_OBJ): $(wildcard $(LIBBPF_SRC)/*.[ch]) | $(OUTPUT)/libbpf
	$(call msg,LIB,$@)
	$(Q)$(MAKE) -C $(LIBBPF_SRC) BUILD_STATIC_ONLY=1		      \
		    OBJDIR=$(dir $@)libbpf DESTDIR=$(dir $@)		      \
		    INCLUDEDIR= LIBDIR= UAPIDIR=			      \
		    install

$(FSSLOWER_ALIASES): fsslower
	$(call msg,SYMLINK,$@)
	$(Q)ln -f -s $^ $@

$(FSDIST_ALIASES): fsdist
	$(call msg,SYMLINK,$@)
	$(Q)ln -f -s $^ $@

install: $(APPS) $(APP_ALIASES)
	$(call msg, INSTALL libbpf-tools)
	$(Q)$(INSTALL) -m 0755 -d $(DESTDIR)$(prefix)/bin
	$(Q)$(INSTALL) $(APPS) $(DESTDIR)$(prefix)/bin
	$(Q)cp -a $(APP_ALIASES) $(DESTDIR)$(prefix)/bin

.PHONY: force
force:

# delete failed targets
.DELETE_ON_ERROR:
# keep intermediate (.skel.h, .bpf.o, etc) targets
.SECONDARY:
