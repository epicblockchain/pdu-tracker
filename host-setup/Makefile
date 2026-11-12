HOST_SETUP_DIR := $(abspath $(dir $(abspath $(lastword $(MAKEFILE_LIST)))))
COMPOSE_FILE := $(HOST_SETUP_DIR)/container-compose.yml
PROJECT_NAME := pdu-tracker
IMAGE ?= ghcr.io/epicblockchain/pdu-tracker:latest
DATA_VOLUME ?= pdu-data
PDU_TRACKER_CONFIG_FILE ?= $(HOME)/.config/pdu-tracker/config.toml
PDU_TRACKER_CONFIG_DIR ?= $(patsubst %/,%,$(dir $(PDU_TRACKER_CONFIG_FILE)))
export PDU_TRACKER_CONFIG_FILE
export PDU_TRACKER_CONFIG_DIR
CONTAINER_NAME := pdu-tracker
PODMAN ?= podman
PODMAN_COMPOSE ?= podman-compose
PODLET_IMAGE ?= ghcr.io/containers/podlet:latest
QUADLET_DIR ?= $(HOME)/.config/containers/systemd
PODLET_SYSTEMD_DIR := /home/podlet/.config/containers/systemd
PODLET_RUN := $(PODMAN) run --rm --userns keep-id \
	-e HOME=/home/podlet \
	-v "$(QUADLET_DIR):$(PODLET_SYSTEMD_DIR):Z" \
	-v "$(HOST_SETUP_DIR):$(HOST_SETUP_DIR):Z" \
	-w "$(HOST_SETUP_DIR)" $(PODLET_IMAGE)
SERVICE_NAME := $(PROJECT_NAME).service
QUADLET_NAME := $(PROJECT_NAME).container

.PHONY: prepare-config setup reconfig quadlet install start disable uninstall

prepare-config:
	@umask 077; \
	mkdir -p "$(dir $(PDU_TRACKER_CONFIG_FILE))"; \
	touch "$(PDU_TRACKER_CONFIG_FILE)"

setup: prepare-config
	cd "$(HOST_SETUP_DIR)" && $(PODMAN_COMPOSE) -f "$(COMPOSE_FILE)" \
		run --rm --no-deps pdu-tracker config setup

reconfig: prepare-config
	cd "$(HOST_SETUP_DIR)" && $(PODMAN_COMPOSE) -f "$(COMPOSE_FILE)" \
		run --rm --no-deps pdu-tracker config setup --reconfigure

quadlet: prepare-config
	mkdir -p "$(QUADLET_DIR)"
	@set -e; \
	compose_file="$(HOST_SETUP_DIR)/.container-compose.yml"; \
	trap 'rm -f "$$compose_file"' EXIT; \
	cd "$(HOST_SETUP_DIR)"; \
	$(PODMAN_COMPOSE) config > "$$compose_file"; \
	sed -i '/x-service/,$$d' "$$compose_file"; \
	sed -i '/^    name: pdu-data$$/d' "$$compose_file"; \
	$(PODLET_RUN) -u --skip-services-check --overwrite -i -a \
		compose "$$compose_file"
	systemctl --user daemon-reload

install:
	$(MAKE) --no-print-directory -f "$(HOST_SETUP_DIR)/Makefile" setup
	$(MAKE) --no-print-directory -f "$(HOST_SETUP_DIR)/Makefile" quadlet
	systemctl --user enable --now podman-auto-update.timer
	systemctl --user start "$(SERVICE_NAME)"
	@if command -v loginctl >/dev/null 2>&1 \
		&& loginctl --no-ask-password enable-linger "$$(id -u)"; then \
		echo "User lingering is enabled; the service will start at boot and run after logout."; \
	else \
		echo "Warning: could not enable user lingering." >&2; \
		echo "Run 'loginctl enable-linger $$(id -u)' to enable it; host policy may require administrator authorization." >&2; \
	fi

start: quadlet
	systemctl --user start "$(SERVICE_NAME)"

disable:
	systemctl --user stop "$(SERVICE_NAME)" || true
	cd "$(HOST_SETUP_DIR)" && $(PODMAN_COMPOSE) -f "$(COMPOSE_FILE)" \
		--podman-stop-args="--force" down
	$(PODMAN) quadlet rm --force --ignore "$(QUADLET_NAME)"
	systemctl --user daemon-reload

uninstall:
	systemctl --user stop "$(SERVICE_NAME)" || true
	@echo "Showing the latest daily and monthly readings saved in the database:"
	@if $(PODMAN) volume exists "$(DATA_VOLUME)"; then \
		if $(PODMAN) image exists "$(IMAGE)"; then \
			if ! $(PODMAN) run --rm --pull=never \
				--volume "$(DATA_VOLUME):/app/data" \
				--volume "$(PDU_TRACKER_CONFIG_DIR):/app/config:z" \
				--env PDU_TRACKER_CONFIG_FILE=/app/config/config.toml \
				"$(IMAGE)" --saved-readings; then \
				echo "Could not read saved baselines; continuing with uninstall." >&2; \
			fi; \
		else \
			echo "PDU Tracker image is unavailable; skipping the saved reading report." >&2; \
		fi; \
	else \
		echo "Podman volume $(DATA_VOLUME) is not present; no saved readings to report."; \
	fi
	cd "$(HOST_SETUP_DIR)" && $(PODMAN_COMPOSE) -f "$(COMPOSE_FILE)" \
		--podman-stop-args="--force" down
	$(PODMAN) quadlet rm --force --ignore "$(QUADLET_NAME)"
	systemctl --user daemon-reload
	@if $(PODMAN) container exists "$(CONTAINER_NAME)"; then \
		$(PODMAN) rm --force "$(CONTAINER_NAME)" > /dev/null; \
	fi
	@if $(PODMAN) volume exists "$(DATA_VOLUME)"; then \
		$(PODMAN) volume rm "$(DATA_VOLUME)"; \
	fi
	@if $(PODMAN) image exists "$(IMAGE)"; then \
		$(PODMAN) image rm --no-prune "$(IMAGE)"; \
	fi
	@echo "PDU Tracker service, container, data volume, and image have been removed."
