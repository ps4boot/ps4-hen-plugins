# Library metadata.

DEBUG_FLAGS = -D__FINAL__=1
LOG_TYPE = -D__USE_PRINTF__

ifeq ($(DEBUG),1)
    DEBUG_FLAGS = -D__FINAL__=0
endif

TYPE         := $(BUILD_TYPE)
FINAL        := $(DEBUG_FLAGS)
BUILD_FOLDER := $(shell pwd)/../bin/plugins
OUTPUT_PRX   := $(shell basename $(CURDIR))
TARGET       := $(BUILD_FOLDER)/prx$(TYPE)/$(OUTPUT_PRX)
TARGET_ELF   := $(BUILD_FOLDER)/elf$(TYPE)/$(OUTPUT_PRX)
TARGETSTUB   := $(OUTPUT_PRX).so
COMMON_DIR    := ../common

# Libraries linked into the ELF.
LIBS := -lSceLibcInternal -lkernel -lScePsm -lmonosgen -lSceRegMgr

EXTRAFLAGS := -fvisibility=hidden -fshort-wchar $(DEBUG_FLAGS) $(LOG_TYPE) -fcolor-diagnostics -Wall -Wno-unused-function -Wno-unused-const-variable

# You likely won't need to touch anything below this point.
# Root vars
TOOLCHAIN     := $(OO_PS4_TOOLCHAIN)
PROJDIR       := ../$(shell basename $(CURDIR))/source
INTDIR        := ../$(shell basename $(CURDIR))/build
INCLUDEDIR    := ../$(shell basename $(CURDIR))/include

# Define objects to build
CFILES      := $(wildcard $(PROJDIR)/*.c)
CPPFILES    := $(wildcard $(PROJDIR)/*.cc)
COMMONFILES := $(wildcard $(COMMONDIR)/*.cc)
OBJS        := $(patsubst $(PROJDIR)/%.c, $(INTDIR)/%.o, $(CFILES)) $(patsubst $(PROJDIR)/%.cc, $(INTDIR)/%.o, $(CPPFILES)) $(patsubst $(COMMONDIR)/%.cc, $(INTDIR)/%.o, $(COMMONFILES))
STUBOBJS    := $(patsubst $(PROJDIR)/%.c, $(INTDIR)/%.o, $(CFILES)) $(patsubst $(PROJDIR)/%.cc, $(INTDIR)/%.o.stub, $(CPPFILES)) $(patsubst $(COMMONDIR)/%.cc, $(INTDIR)/%.o.stub, $(COMMONFILES))

# Define final C/C++ flags
CFLAGS      := --target=x86_64-pc-freebsd12-elf -fPIC -funwind-tables -c $(EXTRAFLAGS) -isysroot $(TOOLCHAIN) -isystem $(TOOLCHAIN)/include
CXXFLAGS    := -std=c++17 $(CFLAGS) -isystem $(TOOLCHAIN)/include/c++/v1
LDFLAGS     := -m elf_x86_64 -pie --script $(TOOLCHAIN)/link.x -e _init --eh-frame-hdr -L$(TOOLCHAIN)/lib $(LIBS) $(TOOLCHAIN)/lib/crtlib.o

# Check for linux vs macOS and account for clang/ld path
UNAME_S     := $(shell uname -s)

ifeq ($(UNAME_S),Linux)
        CC      := clang
        CCX     := clang++
        LD      := ld.lld
        CDIR    := linux
endif
ifeq ($(UNAME_S),Darwin)
        CC      := /usr/local/opt/llvm/bin/clang
        CCX     := /usr/local/opt/llvm/bin/clang++
        LD      := /usr/local/opt/llvm/bin/ld.lld
        CDIR    := macos
endif

bundle_data:
	xxd -i data/hen_settings.xml | sed 's/^unsigned /static const unsigned /' > source/hen_settings.inc.c
	xxd -i data/hen_settings_icon.png | sed 's/^unsigned /static const unsigned /' > source/hen_settings_icon.inc.c

$(INTDIR):
	mkdir -p $(INTDIR)

$(BUILD_FOLDER)/prx$(TYPE):
	mkdir -p $(BUILD_FOLDER)/prx$(TYPE)

$(BUILD_FOLDER)/elf$(TYPE):
	mkdir -p $(BUILD_FOLDER)/elf$(TYPE)

$(TARGET): bundle_data $(INTDIR) $(BUILD_FOLDER)/prx$(TYPE) $(BUILD_FOLDER)/elf$(TYPE) $(OBJS)
	$(LD) $(INTDIR)/*.o -o $(TARGET_ELF).elf $(LDFLAGS)
	$(TOOLCHAIN)/bin/$(CDIR)/create-fself -in=$(TARGET_ELF).elf -out=$(TARGET_ELF).oelf --lib=$(TARGET).prx --paid 0x3800000000000011

$(TARGETSTUB): $(INTDIR) $(STUBOBJS)
	$(CC) $(INTDIR)/*.o.stub -o $(TARGETSTUB) -target x86_64-pc-linux-gnu -shared -fuse-ld=lld -ffreestanding -nostdlib -fno-builtin -L$(TOOLCHAIN)/lib $(LIBS)

$(INTDIR)/%.o: $(PROJDIR)/%.c
	mkdir -p $(dir $@)
	$(CC) $(CFLAGS) -o $@ $<

$(INTDIR)/%.o: $(PROJDIR)/%.cc
	$(CCX) $(CXXFLAGS) -o $@ $<

$(INTDIR)/%.o.stub: $(PROJDIR)/%.c
	$(CC) -target x86_64-pc-linux-gnu -ffreestanding -nostdlib -fno-builtin -fPIC $(O_FLAG) -s -c -o $@ $<

$(INTDIR)/%.o.stub: $(PROJDIR)/%.cc
	$(CCX) -target x86_64-pc-linux-gnu -ffreestanding -nostdlib -fno-builtin -fPIC $(O_FLAG) -s -c -o $@ $<

build-info:
	$(shell echo "#define GIT_COMMIT \"$(shell git rev-parse HEAD)\"" > $(COMMON_DIR)/git_ver.h)
	$(shell echo "#define GIT_VER \"$(shell git branch --show-current)\"" >> $(COMMON_DIR)/git_ver.h)
	$(shell echo "#define GIT_NUM $(shell git rev-list HEAD --count)" >> $(COMMON_DIR)/git_ver.h)
	$(shell echo "#define GIT_NUM_STR \"$(shell git rev-list HEAD --count)\"" >> $(COMMON_DIR)/git_ver.h)
	$(shell echo "#define BUILD_DATE \"$(shell date '+%b %d %Y @ %T')\"" >> $(COMMON_DIR)/git_ver.h)

mono:
	make -C mono_bin clean send

send: $(TARGET)
	curl -T $(TARGET).prx ftp://$(PS4_HOST):2121/data/hen/$(OUTPUT_PRX).prx
	curl -T data/hen_settings.xml ftp://$(PS4_HOST):2121/data/hen/shellui_data/hen_settings.xml

.PHONY: clean
.DEFAULT_GOAL := all

all: build-info $(TARGET)

clean:
	rm -rf $(PROJDIR)/*.inc.c $(TARGET) $(TARGETSTUB) $(INTDIR) $(OBJS)
