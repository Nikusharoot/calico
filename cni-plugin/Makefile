include ../metadata.mk

PACKAGE_NAME = github.com/projectcalico/calico/cni-plugin

# Add in local static-checks
LOCAL_CHECKS=check-boring-ssl

# Name of the images.
# e.g., <registry>/<name>:<tag>
CNI_PLUGIN_IMAGE ?=cni
WINDOWS_IMAGE ?=cni-windows
BUILD_IMAGES   ?=$(CNI_PLUGIN_IMAGE)

###############################################################################
# Download and include ../lib.Makefile
#   Additions to EXTRA_DOCKER_ARGS need to happen before the include since
#   that variable is evaluated when we declare DOCKER_RUN and siblings.
###############################################################################
include ../lib.Makefile

###############################################################################
LDFLAGS = -X main.VERSION=$(GIT_VERSION)

SRC_FILES=$(shell find pkg cmd internal -name '*.go')
TEST_SRC_FILES=$(shell find tests -name '*.go')
WINFV_SRCFILES=$(shell find win_tests -name '*.go')

# fail if unable to download
CURL=curl -C - -sSf

# Use forked CNI plugin URL and corresponding tagged artifacts.
CNI_VERSION=v1.1.1-calico+go-1.20.7
CNI_ARTIFACTS_URL=https://github.com/projectcalico/containernetworking-plugins/releases/download
FLANNEL_VERSION=v1.1.2

# By default set the CNI_SPEC_VERSION to 0.3.1 for tests.
CNI_SPEC_VERSION?=0.3.1

# Markers for various images we produce.
DEPLOY_CONTAINER_STATIC_MARKER=cni_deploy_container-$(ARCH).created
DEPLOY_CONTAINER_FIPS_MARKER=cni_deploy_container-$(ARCH)-fips.created

# Set FIPS to true to enable a FIPS compliant build using BoringSSL and CGO.
# Note that this produces binaries that are dynamically linked, and so have dependencies
# on the host machine.
FIPS ?= false

ifeq ($(FIPS), true)
CGO_ENABLED=1
BIN=bin/$(ARCH)-fips
DEPLOY_CONTAINER_MARKER=$(DEPLOY_CONTAINER_FIPS_MARKER)
VALIDARCHES=amd64
else
CGO_ENABLED=0
BIN=bin/$(ARCH)
DEPLOY_CONTAINER_MARKER=$(DEPLOY_CONTAINER_STATIC_MARKER)
endif

.PHONY: clean clean-windows
clean: clean-windows
	# Clean .created files which indicate images / releases have been built.
	find . -name '.*.created*' -type f -delete
	find . -name '.*.published*' -type f -delete
	rm -rf $(BIN) bin $(DEPLOY_CONTAINER_MARKER) .go-pkg-cache pkg/install/install.test
	rm -f *.created
	rm -rf config/
	rm -rf dist/

clean-windows: clean-windows-builder
	rm -rf $(WINDOWS_BIN) $(WINDOWS_DIST)

###############################################################################
# Building the binary
###############################################################################
.PHONY: build
build: $(BIN)/install $(BIN)/calico $(BIN)/calico-ipam
ifeq ($(ARCH),amd64)
# Go only supports amd64 for Windows builds.
WINDOWS_BIN=bin/windows
build: $(WINDOWS_BIN)/install.exe $(WINDOWS_BIN)/calico.exe $(WINDOWS_BIN)/calico-ipam.exe
endif
# If ARCH is arm based, find the requested version/variant
ifeq ($(word 1,$(subst v, ,$(ARCH))),arm)
ARM_VERSION := $(word 2,$(subst v, ,$(ARCH)))
endif
# Define go architecture flags to support arm variants
GOARCH_FLAGS :=-e GOARCH=$(ARCH)
ifdef ARM_VERSION
GOARCH_FLAGS :=-e GOARCH=arm -e GOARM=$(ARM_VERSION)
endif

build-all: $(addprefix sub-build-,$(VALIDARCHES))
sub-build-%:
	$(MAKE) build ARCH=$*

## Build the Calico network plugin and ipam plugins
$(BIN)/install binary: $(SRC_FILES)
ifeq ($(FIPS), true)
	$(call build_cgo_boring_binary, $(PACKAGE_NAME)/cmd/calico, $(BIN)/install)
else
	$(DOCKER_RUN) -e CGO_ENABLED=$(CGO_ENABLED) $(CALICO_BUILD) sh -c '\
		$(GIT_CONFIG_SSH) go build -buildvcs=false -v -o $(BIN)/install -ldflags "$(LDFLAGS)" $(PACKAGE_NAME)/cmd/calico'
endif

## Build the Calico network plugin and ipam plugins for Windows
$(WINDOWS_BIN)/install.exe: $(SRC_FILES)
	-mkdir -p $(WINDOWS_BIN)
	$(DOCKER_RUN) \
	-e GOOS=windows \
	-e GOARCH=amd64 \
	-e CGO_ENABLED=1 \
		$(CALICO_BUILD) sh -c '$(GIT_CONFIG_SSH) \
		go build -v -buildvcs=false -o $(WINDOWS_BIN)/install.exe -ldflags "$(LDFLAGS)" $(PACKAGE_NAME)/cmd/calico'

$(WINDOWS_BIN)/calico-ipam.exe $(WINDOWS_BIN)/calico.exe: $(WINDOWS_BIN)/install.exe
	cp "$<" "$@"

# NOTE: WINDOWS_IMAGE_REQS must be defined with the requirements to build the windows
# image. These must be added as reqs to 'image-windows' (originally defined in
# lib.Makefile) on the specific package Makefile otherwise they are not correctly
# recognized.
WINDOWS_IMAGE_REQS := Dockerfile-windows build fetch-win-cni-bins
image-windows: $(WINDOWS_IMAGE_REQS)

###############################################################################
# Building the image
###############################################################################
image: $(DEPLOY_CONTAINER_MARKER)
image-all: $(addprefix sub-image-,$(VALIDARCHES)) sub-image-fips-amd64
sub-image-%:
	$(MAKE) image ARCH=$*
sub-image-fips-%:
	$(MAKE) image FIPS=true ARCH=$*

# Builds the statically compiled binaries into a container.
$(DEPLOY_CONTAINER_STATIC_MARKER): Dockerfile.$(ARCH) build fetch-cni-bins
	$(DOCKER_BUILD) --build-arg BIN_DIR=$(BIN) -t $(CNI_PLUGIN_IMAGE):latest-$(ARCH) -f Dockerfile.$(ARCH) . --load
	$(MAKE) retag-build-images-with-registries VALIDARCHES=$(ARCH) IMAGETAG=latest
	touch $@

# Builds the FIPS binaries into a container.
$(DEPLOY_CONTAINER_FIPS_MARKER): Dockerfile.$(ARCH) build fetch-cni-bins
	$(DOCKER_BUILD) --build-arg BIN_DIR=$(BIN) -t $(CNI_PLUGIN_IMAGE):latest-fips-$(ARCH) -f Dockerfile.$(ARCH) . --load
	$(MAKE) retag-build-images-with-registries VALIDARCHES=$(ARCH) IMAGETAG=latest-fips LATEST_IMAGE_TAG=latest-fips
	touch $@

.PHONY: fetch-cni-bins fetch-win-cni-bins
fetch-cni-bins: $(BIN)/flannel $(BIN)/loopback $(BIN)/host-local $(BIN)/portmap $(BIN)/tuning $(BIN)/bandwidth
fetch-win-cni-bins: $(WINDOWS_BIN)/flannel.exe

$(BIN)/loopback $(BIN)/host-local $(BIN)/portmap $(BIN)/tuning $(BIN)/bandwidth:
	-mkdir -p $(BIN)
	$(CURL) -L --retry 5 $(CNI_ARTIFACTS_URL)/$(CNI_VERSION)/cni-plugins-linux-$(subst v7,,$(ARCH))-$(CNI_VERSION).tgz | tar -xz -C $(BIN) ./loopback ./host-local ./portmap ./tuning ./bandwidth

$(BIN)/flannel:
	-mkdir -p $(BIN)
	$(CURL) -L --retry 5 https://github.com/flannel-io/cni-plugin/releases/download/$(FLANNEL_VERSION)/cni-plugin-flannel-linux-$(subst v7,,$(ARCH))-$(FLANNEL_VERSION).tgz | tar -xz -C $(BIN) ./flannel-$(subst v7,,$(ARCH))
	mv $(BIN)/flannel-$(subst v7,,$(ARCH)) $@

$(WINDOWS_BIN)/flannel.exe:
	-mkdir -p $(WINDOWS_BIN)
	$(CURL) -L --retry 5 https://github.com/flannel-io/cni-plugin/releases/download/$(FLANNEL_VERSION)/cni-plugin-flannel-windows-amd64-$(FLANNEL_VERSION).tgz | tar -xz -C $(WINDOWS_BIN) ./flannel-amd64.exe
	mv $(WINDOWS_BIN)/flannel-amd64.exe $@

###############################################################################
# Unit Tests
###############################################################################
## Run the unit tests.
ut: run-k8s-controller-manager $(BIN)/install $(BIN)/host-local $(BIN)/calico-ipam $(BIN)/calico
	$(MAKE) ut-datastore DATASTORE_TYPE=etcdv3
	$(MAKE) ut-datastore DATASTORE_TYPE=kubernetes

check-boring-ssl: $(BIN)/install
	$(DOCKER_RUN) -e CGO_ENABLED=$(CGO_ENABLED) $(CALICO_BUILD) $(CHECK_BORINGSSL)
	-rm -f $(BIN)/tags.txt

$(BIN)/calico-ipam $(BIN)/calico: $(BIN)/install
	cp "$<" "$@"

ut-datastore:
	# The tests need to run as root
	docker run --rm -t --privileged --net=host \
	$(EXTRA_DOCKER_ARGS) \
	-e ETCD_IP=$(LOCAL_IP_ENV) \
	-e LOCAL_USER_ID=$(LOCAL_USER_ID) \
	-e RUN_AS_ROOT=true \
	-e ARCH=$(ARCH) \
	-e PLUGIN=calico \
	-e BIN=/go/src/$(PACKAGE_NAME)/$(BIN) \
	-e CNI_SPEC_VERSION=$(CNI_SPEC_VERSION) \
	-e DATASTORE_TYPE=$(DATASTORE_TYPE) \
	-e ETCD_ENDPOINTS=http://$(LOCAL_IP_ENV):2379 \
	-e KUBECONFIG=/home/user/certs/kubeconfig \
	-v $(CURDIR)/../:/go/src/github.com/projectcalico/calico:rw \
	-v $(CERTS_PATH):/home/user/certs \
	$(CALICO_BUILD) sh -c '$(GIT_CONFIG_SSH) \
			cd  /go/src/$(PACKAGE_NAME) && \
			ginkgo -cover -r -skipPackage pkg/install $(GINKGO_ARGS)'

ut-etcd: run-k8s-controller-manager build $(BIN)/host-local
	$(MAKE) ut-datastore DATASTORE_TYPE=etcdv3
	make stop-etcd
	make stop-k8s-controller

ut-kdd: run-k8s-controller-manager build $(BIN)/host-local
	$(MAKE) ut-datastore DATASTORE_TYPE=kubernetes
	make stop-etcd
	make stop-k8s-controller

## Run the tests in a container (as root) for different CNI spec versions
## to make sure we don't break backwards compatibility.
.PHONY: test-cni-versions
test-cni-versions:
	for cniversion in "0.2.0" "0.3.1" ; do \
		if make ut CNI_SPEC_VERSION=$$cniversion; then \
			echo "CNI version $$cniversion PASSED"; \
		else \
			echo "CNI version $$cniversion FAILED"; \
			exit 1; \
		fi; \
	done

###############################################################################
# Install test
###############################################################################
# We pre-build the test binary so that we can run it outside a container and allow it
# to interact with docker.
pkg/install/install.test: pkg/install/*.go
	$(DOCKER_RUN) $(CALICO_BUILD) sh -c '$(GIT_CONFIG_SSH) \
			cd /go/src/$(PACKAGE_NAME) && \
			go test ./pkg/install -c --tags install_test -o ./pkg/install/install.test'

.PHONY: test-install-cni
## Test the install
test-install-cni: run-k8s-apiserver image pkg/install/install.test
	cd pkg/install && CONTAINER_NAME=$(CNI_PLUGIN_IMAGE):latest-$(ARCH) CERTS_PATH=$(CERTS_PATH) ./install.test

###############################################################################
# CI/CD
###############################################################################
.PHONY: ci
ci: clean mod-download build static-checks test-cni-versions image-all test-install-cni

## Deploys images to registry
cd: image-all cd-common

## Build fv binary for Windows
$(WINDOWS_BIN)/win-fv.exe: $(WINFV_SRCFILES)
	$(DOCKER_RUN) -e GOOS=windows $(CALICO_BUILD) sh -c '$(GIT_CONFIG_SSH) go test ./win_tests -c -o $(WINDOWS_BIN)/win-fv.exe'

###############################################################################
# Release
###############################################################################
## Produces a clean build of release artifacts at the specified version.
release-build: .release-$(VERSION).created
.release-$(VERSION).created:
	$(MAKE) clean image-all RELEASE=true
	$(MAKE) retag-build-images-with-registries RELEASE=true IMAGETAG=$(VERSION)
	$(MAKE) retag-build-images-with-registries RELEASE=true IMAGETAG=latest
	$(MAKE) FIPS=true retag-build-images-with-registries RELEASE=true IMAGETAG=$(VERSION)-fips LATEST_IMAGE_TAG=latest-fips
	$(MAKE) FIPS=true retag-build-images-with-registries RELEASE=true IMAGETAG=latest-fips LATEST_IMAGE_TAG=latest-fips
	$(MAKE) release-verify
	touch $@

## Verifies the release artifacts produces by `make release-build` are correct.
release-verify: release-prereqs
	# Check the reported version is correct for each release artifact.
	$(MAKE) release-verify-version IMAGE=calico/cni:$(VERSION)-$(ARCH)
	$(MAKE) release-verify-version IMAGE=calico/cni:$(VERSION)-fips-$(ARCH)
	$(MAKE) release-verify-version IMAGE=quay.io/calico/cni:$(VERSION)-$(ARCH)
	$(MAKE) release-verify-version IMAGE=quay.io/calico/cni:$(VERSION)-fips-$(ARCH)
	# Check that the FIPS binaries have the correct symbols.
	$(MAKE) release-verify-fips IMAGE=calico/cni:$(VERSION)-fips-$(ARCH)
	$(MAKE) release-verify-fips IMAGE=quay.io/calico/cni:$(VERSION)-fips-$(ARCH)

release-verify-version:
	docker run --rm $(IMAGE) calico -v | grep -x $(VERSION) || ( echo "Reported version does not match" && exit 1 )
	docker run --rm $(IMAGE) calico-ipam -v | grep -x $(VERSION) || ( echo "Reported version does not match" && exit 1 )

release-verify-fips:
	rm -rf .tmp && mkdir -p .tmp
	# Copy binaries from the image so we can analyze them.
	sh -c "docker create --name calico-cni-verify $(IMAGE); docker cp calico-cni-verify:/opt/cni/bin/install .tmp/calico; docker rm -f calico-cni-verify"
	go tool nm .tmp/calico | grep '_Cfunc__goboringcrypto_' 1> /dev/null || echo "ERROR: Binary in image '$(IMAGE)' is missing expected goboring symbols"
	rm -rf .tmp

release-publish: release-prereqs .release-$(VERSION).published
.release-$(VERSION).published:
	$(MAKE) push-images-to-registries push-manifests IMAGETAG=$(VERSION) RELEASE=$(RELEASE) CONFIRM=$(CONFIRM)
	$(MAKE) FIPS=true push-images-to-registries push-manifests IMAGETAG=$(VERSION)-fips RELEASE=$(RELEASE) CONFIRM=$(CONFIRM)

	# Push Windows images.
	$(MAKE) release-windows IMAGETAG=$(VERSION) CONFIRM=$(CONFIRM)
	touch $@

# WARNING: Only run this target if this release is the latest stable release. Do NOT
# run this target for alpha / beta / release candidate builds, or patches to earlier Calico versions.
## Pushes `latest` release images. WARNING: Only run this for latest stable releases.
release-publish-latest: release-prereqs
	# Check latest versions match.
	if ! docker run $(CNI_PLUGIN_IMAGE):latest-$(ARCH) calico -v | grep '^$(VERSION)$$'; then echo "Reported version:" `docker run $(CNI_PLUGIN_IMAGE):latest-$(ARCH) calico -v` "\nExpected version: $(VERSION)"; false; else echo "\nVersion check passed\n"; fi
	if ! docker run quay.io/$(CNI_PLUGIN_IMAGE):latest-$(ARCH) calico -v | grep '^$(VERSION)$$'; then echo "Reported version:" `docker run quay.io/$(CNI_PLUGIN_IMAGE):latest-$(ARCH) calico -v` "\nExpected version: $(VERSION)"; false; else echo "\nVersion check passed\n"; fi

	$(MAKE) push-images-to-registries push-manifests RELEASE=true IMAGETAG=latest RELEASE=$(RELEASE) CONFIRM=$(CONFIRM)

	# Push Windows images.
	$(MAKE) release-windows IMAGETAG=latest CONFIRM=$(CONFIRM)

###############################################################################
# Developer helper scripts (not used by build or test)
###############################################################################
.PHONY: test-watch
## Run the unit tests, watching for changes.
test-watch: $(BIN)/install run-etcd run-k8s-apiserver
	# The tests need to run as root
	CGO_ENABLED=0 ETCD_IP=127.0.0.1 PLUGIN=calico GOPATH=$(GOPATH) $(shell which ginkgo) watch -skipPackage pkg/install
