# SPDX-License-Identifier: GPL-2.0-or-later
# Copyright (c) International Business Machines  Corp., 2009

top_srcdir ?= ../../../..

LTPLIBS = ltpsigwait

include $(top_srcdir)/include/mk/testcases.mk

LDLIBS  := -lltpsigwait $(LDLIBS)

include $(top_srcdir)/include/mk/generic_leaf_target.mk
