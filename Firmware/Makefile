# GCVideo DVI Firmware
#
# Copyright (C) 2015-2021, Ingo Korb <ingo@akana.de>
# All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions are met:
#
# 1. Redistributions of source code must retain the above copyright notice,
#    this list of conditions and the following disclaimer.
# 2. Redistributions in binary form must reproduce the above copyright notice,
#    this list of conditions and the following disclaimer in the documentation
#    and/or other materials provided with the distribution.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
# AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
# ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE
# LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
# CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
# SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
# INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
# CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
# ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF
# THE POSSIBILITY OF SUCH DAMAGE.
#
#
# Makefile: program construction rules
#

VERSION := TEST

ifeq ($(TARGET), gc-dvi)
  EXTRA_CFLAGS := -DCONSOLE_GC
else ifeq ($(TARGET), wii-dvi)
  EXTRA_CFLAGS := -DCONSOLE_WII
else ifeq ($(TARGET), gc-dual)
  EXTRA_CFLAGS := -DCONSOLE_GC -DOUTPUT_DUAL
else ifeq ($(TARGET), wii-dual)
  EXTRA_CFLAGS := -DCONSOLE_WII -DOUTPUT_DUAL
else
dummy:
	@echo "TARGET variable not specified"
	@echo "Please select a build target using \"make TARGET=...\""
	@echo "Valid targets:"
	@echo "  - gc-dvi"
	@echo "  - wii-dvi"
	@echo "  - gc-dual"
	@echo "  - wii-dual"
	@false
endif

ifeq ($(MODULE),main)
# nothing yet
else ifeq ($(MODULE),flasher)
# still nothing (target hardware ID is stored in updater signature)
else
dummy_mod:
	@echo "MODULE variable invalid"
	@false
endif


PERL := perl
CROSS_PREFIX=zpu-elf-
CC := $(CROSS_PREFIX)gcc
LD := $(CROSS_PREFIX)ld
AS := $(CROSS_PREFIX)as
CFLAGS = -Wall -Werror -Os -g -std=gnu99 -ffunction-sections -fdata-sections -I. \
         -DMODULE_$(MODULE) -finline-limit=11 \
         -DVERSION=\"$(VERSION)\" $(EXTRA_CFLAGS) $(patsubst %,-D%,$(FEATURE_FLAGS))
LDFLAGS := -Tstandalone-bsd.ld -Wl,--gc-sections -Wl,--relax -nostartfiles

DATA2MEM := data2mem

ifndef FEATURE_FLAGS
  OBJDIR := obj-$(TARGET)-$(MODULE)
else
  space := $(subst ,, )
  FLAGSUFFIX := $(subst $(space),-,$(strip $(FEATURE_FLAGS)))
  OBJDIR := obj-$(TARGET)-$(MODULE)-$(FLAGSUFFIX)
endif

SRCFILES_COMMON := main.c osd.c printf.c pad.c vsync.c irrx.c \
	spiflash.c utils.c icap.c screen_irconfig.c

SRCFILES_main := modeset_common.c screen_about.c screen_allmodes.c \
	screen_idle.c screen_mainmenu.c screen_osdsettings.c screen_outputsettings.c \
	screen_picturesettings.c screen_advanced.c screen_scanlines.c settings-main.c \
	reblanker.c infoframe.c menu.c colormatrix.c

SRCFILES_flasher := flasher.c settings-flasher.c crc32mpeg.c exodecr.c \
	menu-lite.c flashviewer.c flasher-diag.c

COPYDIR     := build/$(MODULE)
BASENAME    := gcvideo-sw-$(TARGET)
FULLNAME    := $(BASENAME)-$(MODULE)
CRT0        := crt0-bsd.S
SRCFILES    := $(SRCFILES_COMMON) $(SRCFILES_$(MODULE))
BITFILE_IN  := ../HDL/gcvideo_dvi/$(COPYDIR)/toplevel_p2xh.bit
BITFILE_OUT := ../HDL/gcvideo_dvi/$(COPYDIR)/p2xh_output.bit
MIFFILE_OUT := ../HDL/gcvideo_dvi/$(COPYDIR)/$(BASENAME).mif
BMMFILE     := ../HDL/gcvideo_dvi/$(COPYDIR)/zpu_bootram_bd.bmm

BRAM_SIZE   := 16384

GENDEPFLAGS = -MMD -MP -MF .dep/$(@F).d

# Enable verbose compilation with "make V=1"
ifdef V
 Q :=
 E := @:
else
 Q := @
 E := @echo
endif

OBJFILES := $(patsubst %,$(OBJDIR)/%,$(SRCFILES:.c=.o) $(CRT0:.S=.o))

all: mif

.PHONY : inject mif

inject: $(OBJDIR)/$(FULLNAME).mem
	$(E) "  DATA2MEM"
	$(Q)$(DATA2MEM) -bm $(BMMFILE) -bt $(BITFILE_IN) -bd $< -o b $(BITFILE_OUT)

mif: $(OBJDIR)/$(FULLNAME).mif
	$(E) "  COPY     $<"
	$(Q)cp $< $(MIFFILE_OUT)

$(OBJDIR)/%.mif: $(OBJDIR)/%.bin
	$(E) "  BIN2MIF  $<"
	$(Q)$(PERL) ./bin2mif.pl -swapwords 32 $(BRAM_SIZE) $< $@

$(OBJDIR)/%.mem: $(OBJDIR)/%.bin
	$(E) "  BIN2MEM  $<"
	$(Q)$(PERL) ./bin2mem.pl $< $@

$(OBJDIR)/%.bin: $(OBJDIR)/%.elf
	$(E) "  BIN      $@"
	$(Q)$(CROSS_PREFIX)objcopy -O binary $< $@

$(OBJDIR)/$(FULLNAME).elf: $(OBJFILES)
	$(E) "  LINK     $@"
	$(Q)$(CC) $(CFLAGS) $(LDFLAGS) -o $@ $^

$(OBJDIR)/%.o: %.S | $(OBJDIR)
	$(E) "  AS       $<"
	$(Q)$(CC) $(CFLAGS) -c -o $@ $<

$(OBJDIR)/%.o: %.c | $(OBJDIR)
	$(E) "  CC       $<"
	$(Q)$(CC) $(CFLAGS) $(GENDEPFLAGS) -c -o $@ $<

# Create the output directory
$(OBJDIR):
	$(E) "  MKDIR  $(OBJDIR)"
	$(Q)mkdir -p $(OBJDIR)

clean:
	$(E) "  CLEAN"
	$(Q)-rm -f $(OBJDIR)/$(FULLNAME).bin $(OBJDIR)/$(FULLNAME).elf
	$(Q)-rm -f $(OBJDIR)/$(FULLNAME).mem $(OBJDIR)/$(FULLNAME).mif $(OBJFILES)
	$(Q)-rm -f .dep/*
	$(Q)-rmdir .dep $(OBJDIR)

.PRECIOUS : %.bin

-include $(shell mkdir .dep 2>/dev/null) $(wildcard .dep/*)
