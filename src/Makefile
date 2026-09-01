# ---- Display info during parsing phase ----
SILENT:=$(findstring s,$(word 1, $(MAKEFLAGS)))
ifneq ($(SILENT),s)
    $(info Using Makefile: $(realpath $(firstword $(MAKEFILE_LIST))) $(MAKECMDGOALS))
endif

ifneq ($(shell echo),)
  CMD_EXE = 1
endif

C_PROGS = ar65     \
          ca65     \
          cc65     \
          chrcvt65 \
          cl65     \
          co65     \
          da65     \
          grc65    \
          ld65     \
          od65     \
          sim65    \
          sp65


.SUFFIXES:

bindir  := $(PREFIX)/bin
datadir := $(PREFIX)/share/cc65

CA65_INC = $(datadir)/asminc
CC65_INC = $(datadir)/include
CL65_TGT = $(datadir)/target
LD65_LIB = $(datadir)/lib
LD65_OBJ = $(datadir)/lib
LD65_CFG = $(datadir)/cfg

ifdef CMD_EXE
  NULLDEV = nul:
  DIRLIST = $(strip $(foreach dir,$1,$(wildcard $(dir))))
  MKDIR = mkdir $(subst /,\,$1)
  RMDIR = $(if $(DIRLIST),rmdir /s /q $(subst /,\,$(DIRLIST)))
  COPY = copy /Y $(subst /,\,$1) $(subst /,\,$2)
else
  NULLDEV = /dev/null
  MKDIR = mkdir -p $1
  RMDIR = $(RM) -r $1
  COPY = cp $1 $2
endif

ifeq ($(SILENT),s)
    QUIET = 1
    NULLOUT = >$(NULLDEV)
    NULLERR = 2>$(NULLDEV)
endif

ifdef QUIET
    .SILENT:
    PQ = "QUIET=1"
    PD = --no-print-directory
ifndef CMD_EXE
    CATERR = 2> $@.errlog || (cat $@.errlog && false)
endif
endif

CC = $(CROSS_COMPILE)gcc
AR = $(CROSS_COMPILE)ar
CARGO ?= cargo
CARGO_TARGET_DIR := $(abspath ../wrk/cargo)
CARGO_PROFILE ?= release

ifeq ($(CARGO_PROFILE),release)
  CARGO_PROFILE_ARG = --release
  CARGO_PROFILE_DIR = release
else ifeq ($(CARGO_PROFILE),debug)
  CARGO_PROFILE_ARG =
  CARGO_PROFILE_DIR = debug
else
  CARGO_PROFILE_ARG = --profile $(CARGO_PROFILE)
  CARGO_PROFILE_DIR = $(CARGO_PROFILE)
endif

ifdef RUST_TARGET
  CARGO_TARGET_ARG = --target $(RUST_TARGET)
  RUST_TARGET_DIR = $(RUST_TARGET)/
endif

ifdef CROSS_COMPILE
  $(info CC: $(CC))
  $(info AR: $(AR))
endif

ifdef USER_CFLAGS
  $(info USER_CFLAGS: $(USER_CFLAGS))
endif

ifndef BUILD_ID
  BUILD_ID := Git $(shell git rev-parse --short HEAD 2>$(NULLDEV) || svnversion 2>$(NULLDEV))
  ifneq ($(words $(BUILD_ID)),2)
    BUILD_ID := N/A
  endif
endif

# k816 is built by Cargo, which not every contributor to the C tools has and
# which no cross build here is set up for. It is therefore part of `make bin`
# only when it can actually be built: `make bin` on a stock machine builds the
# C tools and says once why k816 is not among them, and `make k816` asks for it
# explicitly and fails with the reason.
#
# A CROSS_COMPILE build leaves it out unconditionally. Renaming a host binary
# to k816.exe is not a Windows build, and supplying a Rust target is a task
# nobody has done.
CARGO_FOUND := $(shell command -v $(CARGO) 2>$(NULLDEV))

ifdef CROSS_COMPILE
  RUST_PROGS =
else ifeq ($(CARGO_FOUND),)
  RUST_PROGS =
else
  RUST_PROGS = k816
endif

PROGS = $(C_PROGS) $(RUST_PROGS)

.PHONY: all mostlyclean clean install zip avail unavail bin k816 $(PROGS)

ifneq ($(SILENT),s)
  $(info BUILD_ID: $(BUILD_ID))
ifeq ($(RUST_PROGS),)
ifndef CROSS_COMPILE
  $(info NOTE: k816 is not built - no Cargo on PATH. The C tools are unaffected.)
endif
endif
endif

CFLAGS += -MMD -MP -O3 -I common \
          -Wall -Wextra -Wno-char-subscripts $(USER_CFLAGS) \
          -DCA65_INC="\"$(CA65_INC)\"" -DCC65_INC="\"$(CC65_INC)\"" -DCL65_TGT="\"$(CL65_TGT)\"" \
          -DLD65_LIB="\"$(LD65_LIB)\"" -DLD65_OBJ="\"$(LD65_OBJ)\"" -DLD65_CFG="\"$(LD65_CFG)\"" \
          -DBUILD_ID="\"$(BUILD_ID)\""

LDLIBS += -lm

ifdef CMD_EXE
  EXE_SUFFIX=.exe
endif

ifdef CROSS_COMPILE
  EXE_SUFFIX=.exe
endif

all bin: $(PROGS)
ifeq ($(MAKELEVEL),0)
ifndef PREFIX
ifndef DESTDIR
	$(warning Warning: PREFIX and DESTDIR are empty - make install will not work)
endif
endif
endif

mostlyclean:
	$(call RMDIR,../wrk)

clean:
	$(call RMDIR,../wrk ../bin)

ifdef CMD_EXE

install avail unavail:

else # CMD_EXE

INSTALL = install

define AVAIL_recipe

ln -s $(abspath ../bin/$(prog)) /usr/local/bin/$(prog)

endef # AVAIL_recipe

define UNAVAIL_recipe

$(RM) /usr/local/bin/$(prog)

endef # UNAVAIL_recipe

install:
	$(INSTALL) -d $(DESTDIR)$(bindir)
	$(INSTALL) ../bin/* $(DESTDIR)$(bindir)

avail:
ifneq ($(patsubst %,../bin/%,$(PROGS)),$(wildcard $(patsubst %,../bin/%,$(PROGS))))
	$(error executables are missing, please run make first)
else
	$(foreach prog,$(PROGS),$(AVAIL_recipe))
endif

unavail:
	$(foreach prog,$(PROGS),$(UNAVAIL_recipe))

endif # CMD_EXE

zip:
	@cd .. && zip cc65 bin/*
	@echo 'https://github.com/cc65/cc65/commits/'$(word 2,$(BUILD_ID))|zip -zq ../cc65

define OBJS_template

$1_OBJS := $$(patsubst %.c,../wrk/%.o,$$(wildcard $1/*.c))

$$($1_OBJS): | ../wrk/$1

../wrk/$1:
	@$$(call MKDIR,$$@)

DEPS += $$($1_OBJS:.o=.d)

endef # OBJS_template

define PROG_template

$$(eval $$(call OBJS_template,$1))

../bin/$1$(EXE_SUFFIX): $$($1_OBJS) ../wrk/common/common.a | ../bin
	$$(CC) $$(LDFLAGS) -o $$@ $$^ $$(LDLIBS)

$1: ../bin/$1$(EXE_SUFFIX)

endef # PROG_template

../wrk/%.o: %.c
	$(if $(QUIET),echo CC:$@)
	$(CC) -c $(CFLAGS) -o $@ $<

../bin:
	@$(call MKDIR,$@)

$(eval $(call OBJS_template,common))

../wrk/common/common.a: $(common_OBJS)
	$(if $(QUIET),echo AR:$@)
	$(AR) r $@ $? $(CATERR)

$(foreach prog,$(C_PROGS),$(eval $(call PROG_template,$(prog))))

../bin/k816$(EXE_SUFFIX): FORCE | ../bin
	$(if $(QUIET),echo RUST:$@)
	BUILD_ID="$(BUILD_ID)" $(CARGO) build --manifest-path k816/Cargo.toml --target-dir $(CARGO_TARGET_DIR) $(CARGO_TARGET_ARG) $(CARGO_PROFILE_ARG)
	$(call COPY,$(CARGO_TARGET_DIR)/$(RUST_TARGET_DIR)$(CARGO_PROFILE_DIR)/k816$(EXE_SUFFIX),$@)

# Asked for by name, k816 is built whether or not it is in RUST_PROGS - and
# says why it cannot be, rather than reporting a missing Cargo.toml or a
# command not found from three lines into a recipe.
ifeq ($(RUST_PROGS),)
k816:
	@echo 'k816 needs Cargo, and $(if $(CROSS_COMPILE),this is a CROSS_COMPILE build - k816 has no cross build,none was found on PATH (set CARGO, or install Rust from https://rustup.rs)).' >&2
	@false
else
k816: ../bin/k816$(EXE_SUFFIX)
endif

.PHONY: FORCE
FORCE:


.PHONY: dbginfo dbgsh test

test: dbginfo dbgsh

$(eval $(call OBJS_template,dbginfo))

dbginfo: $(dbginfo_OBJS)

../wrk/dbgsh$(EXE_SUFFIX): $(dbginfo_OBJS) ../wrk/common/common.a
	$(if $(QUIET),echo LINK:$@)
	$(CC) $(LDFLAGS) -o $@ $^ $(LDLIBS)

dbgsh: ../wrk/dbgsh$(EXE_SUFFIX)


-include $(DEPS)
