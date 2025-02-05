# SPDX-FileCopyrightText: 2021 SAP SE or an SAP affiliate company and Gardener contributors
#
# SPDX-License-Identifier: Apache-2.0

all: build

CLANG_OS_FLAGS = ""

ifeq ($(shell lsb_release -si 2>/dev/null), Ubuntu)
	CLANG_OS_FLAGS="-I/usr/include/x86_64-linux-gnu"
endif

.PHONY: build
build: bpf
	go build -o bin/connectivity-exporter main.go

.PHONY: bpf
bpf:
	clang $(CLANG_OS_FLAGS) -target bpf -O2 -g -c -x c packet/c/cap.c -o packet/c/cap.o
	clang $(CLANG_OS_FLAGS) -target bpf -O2 -g -c -x c packet/c/cap.c -o packet/c/cap-testing.o -DTEST_ENABLED=1

BUILT_TOOLS_DIR := $(abspath ./.built-tools)
TOOLS_MOD_DIR := ./internal/tools
BUILT_TOOLS_DEPS := $(TOOLS_MOD_DIR)/go.mod $(TOOLS_MOD_DIR)/go.sum $(TOOLS_MOD_DIR)/tools.go
GOLANGCI_LINT := $(BUILT_TOOLS_DIR)/golangci-lint

$(BUILT_TOOLS_DIR):
	@mkdir -p $(BUILT_TOOLS_DIR)

$(BUILT_TOOLS_DIR)/golangci-lint: $(BUILT_TOOLS_DEPS) | $(BUILT_TOOLS_DIR)
	cd $(TOOLS_MOD_DIR); \
	go build -o $@ github.com/golangci/golangci-lint/cmd/golangci-lint

.PHONY: lint
lint: $(GOLANGCI_LINT)
	$(GOLANGCI_LINT) run --fix
	$(GOLANGCI_LINT) run

.PHONY: tidy
tidy:
	set -e; \
	for mod in . $(TOOLS_MOD_DIR); do \
	  (cd "$${mod}" && go mod tidy); \
	done

.PHONY: sync-base-image
sync-base-image:
	./base-image/update-go-files.sh

.PHONY: check-clean-work-tree
check-clean-work-tree:
	@if ! git diff --quiet; then \
	  echo; \
	  echo 'Working tree is not clean.'; \
	  echo; \
	  git status; \
	  exit 1; \
	fi

.PHONY: test
test: bpf
ifneq ($(shell id -u),0)
	$(warning ***Root privileges are required for executing BPF-related tests***)
	sudo $(shell which go) test -tags testing -v ./... -count=1
else
	go test -tags testing -v ./... -count=1
endif

.PHONY: benchmark
benchmark: bpf
ifneq ($(shell id -u),0)
	$(warning ***Root privileges are required for executing BPF-related tests***)
	sudo $(shell which go) test -run=$^ -bench=. ./...
else
	go test -run=$^ -bench=. ./...
endif

.PHONY: clean
clean:
	rm -rf connectivity-exporter packet/c/cap.o $(BUILT_TOOLS_DIR)
