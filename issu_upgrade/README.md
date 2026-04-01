# AOS-CX ISSU Ansible Playbook

## Overview

This Ansible playbook automates an **In-Service Software Upgrade (ISSU)** for HPE Aruba Networking AOS-CX switches. In-Service Software Upgrade ([ISSU](https://arubanetworking.hpe.com/techdocs/AOS-CX/10.15/HTML/fundamentals_6300-6400/Content/Chp_Cfg_FW_mgt/issu-sum.htm)) is the name given to a mechanism used to upgrade a device without affecting the service it is providing. In the case of switches, this means that the traffic won't be interrupted during the upgrade and if an error happens and traffic is interrupted to keep the interruption as short as possible.

***

## Requirements & Assumptions

### Software Versions

- Devices must be running **AOS-CX 10.13** or **AOS-CX 10.16** prior to upgrade.
- Target firmware versions supported:
  - **AOS-CX 10.16.1030** and above
  - **AOS-CX 10.13.1161** and above

### Hardware

- Target devices must be AOS-CX switches capable of ISSU upgrades, 6300s in VSF stacks were used for testing/validation.

### Network Infrastructure

This workflow assumes the following regarding the network infrastructure in which this playbook will be ran:

- A **TFTP server must be reachable** from the target devices and is used as the source to pull the desired firmware image.
- The playbook uses the variable `tftp_server_path` to specify the TFTP server address and path, and `x_image_version` to specify the firmware image filename.

### Device Management

- Target devices must **not** be connected to or managed by **HPE Aruba Networking Central**. If devices are being managed by Central, then [support mode](https://arubanetworking.hpe.com/techdocs/AOS-S/16.10/MCG/WB/content/common%20files/aru-cen-sup-mod.htm) must be enabled before executing commands.

### Ansible

- Ansible version **2.18 or lower**
- The [**`arubanetworks.aoscx`**](https://galaxy.ansible.com/ui/repo/published/arubanetworks/aoscx/) Ansible collection must be installed.
- Required Ansible variables:

| Variable           | Description                                                                |
| ------------------ | -------------------------------------------------------------------------- |
| `tftp_server_path` | TFTP server IP/hostname and directory path (e.g., `192.168.1.10/firmware`) |
| `x_image_version`  | Firmware image filename (e.g., `ArubaOS-CX_10_16_1030.swi`)                |
| `vrf_name`         | VRF used for the TFTP copy operation (e.g., `mgmt`)                        |

***

## Workflow Steps

The playbook is divided into three tagged stages that can be run independently or sequentially.

### Stage 1: `copy_firmware`

**Task — Copy new firmware to devices**

Copies the target firmware image from the TFTP server to the **secondary** boot partition on each device.

- Issues the `copy tftp://...` command, automatically confirming the overwrite prompt.
- Waits for the switch to report `Verifying and writing system firmware...` to confirm the copy is in progress.
- Retries up to **40 times** with a **10-second interval** between checks.
- Uses an extended command timeout of **400 seconds** to accommodate large image transfers.

***

### Stage 2: `validate_issu_readiness`

**Task 1 — Validate ISSU Readiness**

Runs `issu update-software validate` and waits for all of the following conditions to pass before proceeding:

- `Current Image Valid               Pass`
- `Target Image Valid                Pass`
- `Target Version Compatible         Pass`
- `Management Modules Ready          Pass`
- `Line Modules Ready                Pass`
- `Features Ready                    Pass`

Retries up to **40 times** with a **10-second interval**.

**Task 2 — Run ISSU Update Software**

Initiates ISSU to the alternate boot location by running `issu update-software`, automatically confirming the prompt. The newer operating system image must be downloaded to the alternate boot location prior to running this command. Additionally, the current running operating system version must match the version that is stored in the current boot location or ISSU will not be allowed.

**Task 3 — Check ISSU Status**

Polls `show issu` and waits for:

- `Prepare for Switchover              In Progress`

Retries up to **40 times** with a **10-second interval**.

> **Known Issue — "Socket is closed" error**
>
> Due to the nature of ISSU, the active management module performs a switchover to the standby module during the upgrade. This causes the existing SSH session to be terminated. As a result, the **"Check ISSU Status"** task may fail with the error:
>
> ```
> Socket is closed
> ```
>
> This is **expected behavior** and does not indicate a failed upgrade. If this error occurs, wait for the switch to complete its switchover and come back online, then proceed to Stage 3 (`validate_and_confirm_issu_completion`) using the tag `validate_and_confirm_issu_completion` once the device is reachable again.

***

### Stage 3: `validate_and_confirm_issu_completion`

**Task 1 — Validate ISSU Completion**

Polls `show issu` and waits for:

- `ISSU Complete                       Complete`

Retries up to **2 times** with a **10-second interval**.

**Task 2 — Confirm ISSU Completion**

Runs `issu update-software confirm` to finalize the upgrade and cancel the rollback timer. The task accepts any of the following responses as successful completion:

- `The ISSU has been confirmed and the rollback timer has been cancelled.`
- `No rollback timer has been started, no action was done.`
- `No rollback timer has been configured, no action was done.`

***

## Running the Playbook

Run all stages sequentially:

```bash
ansible-playbook issu_playbook.yml -e "tftp_server_path=<server_path> x_image_version=<image_file> vrf_name=<vrf>"
```

Run a specific stage using tags:

```bash
# Stage 1 only
ansible-playbook issu_playbook.yml --tags copy_firmware -e "..."

# Stage 2 only
ansible-playbook issu_playbook.yml --tags validate_issu_readiness -e "..."

# Stage 3 only
ansible-playbook issu_playbook.yml --tags validate_and_confirm_issu_completion -e "..."
```

***

## Expected Results

| Stage                                  | Expected Outcome                                                                                          |
| -------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `copy_firmware`                        | Firmware image successfully written to the secondary partition on all target devices.                     |
| `validate_issu_readiness`              | All ISSU readiness checks pass and the ISSU upgrade process is initiated. The switchover begins.          |
| `validate_and_confirm_issu_completion` | ISSU is confirmed as complete, the rollback timer is cancelled, and devices are running the new firmware. |

After a successful run, verify the new firmware version on each device with:

```
show version
```