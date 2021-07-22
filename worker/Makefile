#
# make tasks for mediasoup-worker.
#

# We need Python 3 here.
PYTHON ?= $(shell command -v python3 2> /dev/null || echo python)
ROOT_DIR := $(shell dirname $(realpath $(lastword $(MAKEFILE_LIST))))
CORES ?= $(shell ${ROOT_DIR}/scripts/cpu_cores.sh || echo 4)
MEDIASOUP_OUT_DIR ?= out
MEDIASOUP_BUILDTYPE ?= Release
GULP = ./scripts/node_modules/.bin/gulp
LCOV = ./deps/lcov/bin/lcov
DOCKER ?= docker
MESON ?= $(MEDIASOUP_OUT_DIR)/pip/bin/meson

# Disable `*.pyc` files creation
export PYTHONDONTWRITEBYTECODE = 1
# Instruct `meson` where to look for ninja binary
export NINJA = $(MEDIASOUP_OUT_DIR)/pip/bin/ninja
# Instruct Python where to look for modules it needs, such that `meson` actually runs from installed location
export PYTHONPATH = $(MEDIASOUP_OUT_DIR)/pip

.PHONY:	\
	default install-meson-ninja setup clean clean-all mediasoup-worker xcode lint format test tidy \
	fuzzer fuzzer-run-all docker-build docker-run libmediasoup-worker

default: install-meson-ninja setup mediasoup-worker

install-meson-ninja:
	# Install `meson` and `ninja` using `pip` into custom location, so we don't depend on system-wide installation
	$(PYTHON) -m pip install --target=$(MEDIASOUP_OUT_DIR)/pip meson ninja

setup:
ifeq ($(MEDIASOUP_BUILDTYPE),Release)
	$(MESON) setup --buildtype release $(MEDIASOUP_OUT_DIR)/$(MEDIASOUP_BUILDTYPE)
else
	$(MESON) setup --buildtype debug $(MEDIASOUP_OUT_DIR)/$(MEDIASOUP_BUILDTYPE)
endif

clean:
	$(RM) -rf $(MEDIASOUP_OUT_DIR)/$(MEDIASOUP_BUILDTYPE)

clean-all:
	$(RM) -rf $(MEDIASOUP_OUT_DIR)

mediasoup-worker:
ifeq ($(MEDIASOUP_WORKER_BIN),)
	$(MESON) compile -C $(MEDIASOUP_OUT_DIR)/$(MEDIASOUP_BUILDTYPE) mediasoup-worker
endif

# TODO: Replace with meson command and remove `configure.py` file
xcode:
	$(PYTHON) ./scripts/configure.py --format=xcode

# TODO: Potentially replace with meson and remove these scripts
lint:
	$(GULP) --gulpfile ./scripts/gulpfile.js lint:worker

# TODO: Potentially replace with meson and remove these scripts
format:
	$(GULP) --gulpfile ./scripts/gulpfile.js format:worker

test:
ifeq ($(MEDIASOUP_WORKER_BIN),)
	$(MESON) compile -C $(MEDIASOUP_OUT_DIR)/$(MEDIASOUP_BUILDTYPE) mediasoup-worker-test
	$(LCOV) --directory ./ --zerocounters
	./$(MEDIASOUP_OUT_DIR)/$(MEDIASOUP_BUILDTYPE)/mediasoup-worker-test --invisibles --use-colour=yes $(MEDIASOUP_TEST_TAGS)
endif

tidy:
	$(PYTHON) ./scripts/clang-tidy.py \
		-clang-tidy-binary=./scripts/node_modules/.bin/clang-tidy \
		-clang-apply-replacements-binary=./scripts/node_modules/.bin/clang-apply-replacements \
		-header-filter='(Channel/**/*.hpp|DepLibSRTP.hpp|DepLibUV.hpp|DepLibWebRTC.hpp|DepOpenSSL.hpp|DepUsrSCTP.hpp|LogLevel.hpp|Logger.hpp|MediaSoupError.hpp|RTC/**/*.hpp|Settings.hpp|Utils.hpp|Worker.hpp|common.hpp|handles/**/*.hpp)' \
		-p=$(MEDIASOUP_OUT_DIR)/$(MEDIASOUP_BUILDTYPE) \
		-j=$(CORES) \
		-checks=$(MEDIASOUP_TIDY_CHECKS) \
		-quiet

fuzzer:
ifeq ($(MEDIASOUP_WORKER_BIN),)
	$(MESON) compile -C $(MEDIASOUP_OUT_DIR)/$(MEDIASOUP_BUILDTYPE) mediasoup-worker-fuzzer
endif

fuzzer-run-all:
	LSAN_OPTIONS=verbosity=1:log_threads=1 ./$(MEDIASOUP_OUT_DIR)/$(MEDIASOUP_BUILDTYPE)/mediasoup-worker-fuzzer -artifact_prefix=fuzzer/reports/ -max_len=1400 fuzzer/new-corpus deps/webrtc-fuzzer-corpora/corpora/stun-corpus deps/webrtc-fuzzer-corpora/corpora/rtp-corpus deps/webrtc-fuzzer-corpora/corpora/rtcp-corpus

docker-build:
ifeq ($(DOCKER_NO_CACHE),true)
	$(DOCKER) build -f Dockerfile --no-cache --tag mediasoup/docker:latest .
else
	$(DOCKER) build -f Dockerfile --tag mediasoup/docker:latest .
endif

docker-run:
	$(DOCKER) run \
		--name=mediasoupDocker -it --rm \
		--privileged \
		--cap-add SYS_PTRACE \
		-v $(shell pwd)/../:/mediasoup \
		mediasoup/docker:latest

libmediasoup-worker:
	echo ${PYTHONPATH}
	$(MESON) compile -C $(MEDIASOUP_OUT_DIR)/$(MEDIASOUP_BUILDTYPE) libmediasoup-worker
