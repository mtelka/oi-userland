#
# CDDL HEADER START
#
# The contents of this file are subject to the terms of the
# Common Development and Distribution License (the "License").
# You may not use this file except in compliance with the License.
#
# You can obtain a copy of the license at usr/src/OPENSOLARIS.LICENSE
# or http://www.illumos.org/license/CDDL.
# See the License for the specific language governing permissions
# and limitations under the License.
#
# CDDL HEADER END
#
# Copyright 2017 Dariusz Sendkowski
#
include ../../../make-rules/shared-macros.mk

COMPONENT_NAME=         xcb-util-xrm
COMPONENT_VERSION=      1.2
COMPONENT_FMRI=         x11/library/xcb-util-xrm
COMPONENT_CLASSIFICATION=Development/X11
COMPONENT_SRC=		$(COMPONENT_NAME)-$(COMPONENT_VERSION)
COMPONENT_PROJECT_URL=  http://github.com/Airblader/xcb-util-xrm
COMPONENT_ARCHIVE=      $(COMPONENT_SRC).tar.gz
COMPONENT_ARCHIVE_HASH= sha256:c4a1d64d4a6973c649e3b6e3c418242f11e39c65d4d227d555d48f6df0558567
COMPONENT_ARCHIVE_URL=  http://github.com/Airblader/$(COMPONENT_NAME)/releases/download/v$(COMPONENT_VERSION)/$(COMPONENT_ARCHIVE)
COMPONENT_LICENSE=      MIT
COMPONENT_LICENSE_FILE= COPYING
COMPONENT_SUMMARY=      XCB utility functions for the X resource manager.


include $(WS_MAKE_RULES)/prep.mk
include $(WS_MAKE_RULES)/configure.mk
include $(WS_MAKE_RULES)/ips.mk

CONFIGURE_OPTIONS+= --disable-static

build: $(BUILD_32_and_64)

install: $(INSTALL_32_and_64)

test: $(NO_TESTS)


REQUIRED_PACKAGES += system/library
REQUIRED_PACKAGES += system/library/math
REQUIRED_PACKAGES += x11/library/libxcb
REQUIRED_PACKAGES += x11/library/xcb-util
