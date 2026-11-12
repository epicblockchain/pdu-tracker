# PDU Tracker

PDU Tracker tracks energy usage from PDUs and sends daily and monthly reports.
Use the provided [Makefile](./host-setup/Makefile) to install, manage, or
uninstall it as a rootless Podman service.

Currently supports Doyontek PDUs.

## Requirements

- Linux host with systemd and a running user service manager
- Podman (runs rootless), `podman-compose`, and GNU Make
- A supported 64-bit architecture: `x86_64` (AMD64) or `aarch64` (ARM64)

Run the commands as the regular user who will own and run the service. Podman
must be configured for rootless use.

## Install

Clone this repository, enter its directory, and run:

```sh
git clone https://github.com/epicblockchain/pdu-tracker.git
cd pdu-tracker
make -f host-setup/Makefile install
```

The installer prompts for PDU addresses and optional report settings when no
devices have been configured. It stores configuration in (by default) 
`~/.config/pdu-tracker/config.toml` and readings in the persistent `pdu-data`
volume. It creates and starts a systemd user service and enables Podman's
auto-update timer. It also attempts to enable user lingering so the service
can run after logout and start at boot.

You can edit `config.toml` directly to change the report time and timezone,
configure daily and monthly Google Chat incoming webhook URLs, or update the
configured PDUs and their active status.

To store the config file at a different location, pass its path to Make:

```sh
make -f host-setup/Makefile install \
  PDU_TRACKER_CONFIG_FILE="$HOME/pdu-tracker/config.toml"
```

## Manage the service

```sh
systemctl --user status pdu-tracker.service
make -f host-setup/Makefile reconfig   # edit PDU and report settings
make -f host-setup/Makefile disable   # stop service, keep data
make -f host-setup/Makefile start     # start it again
make -f host-setup/Makefile uninstall # remove service and saved data
```

`reconfig` reruns the interactive setup and shows the current PDU, webhook,
timezone, and report-time settings. Press `Enter` at a setting prompt to keep its
current value. Previous readings can also be edited.

`uninstall` deletes the `pdu-data` volume, including saved readings. The host
configuration file is retained.
