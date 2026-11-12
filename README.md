# os-rpi: stainSTORM robot controller pallet

This repo is openUC2 OS configured for a Raspberry Pi next to a lab robot, which the Pi provides to
an [Arkitekt](https://arkitekt.live) server. ImSwitch, the openUC2 documentation and the ImSwitch
build steps (Hikrobot camera driver, CAN overlay, `dx` mode) are removed. The infrastructure
(networking, Caddy, Cockpit, file browser, Forklift itself) is imported via Forklift pallet file
imports from [the openUC2/os-rpi repo](https://github.com/openUC2/os-rpi).

This repo is both the [Forklift](https://github.com/PlanktoScope/forklift) pallet for the OS, and
the automated build system for creating OS images which can be flashed onto SD cards for booting
Raspberry Pi computers in the OS.

## Robots

| Deployment | Robot | Container | Page |
|---|---|---|---|
| `fairino` | Fairino arm | [stainSTORM/fairinogale](https://github.com/stainSTORM/fairinogale) | `http://<machine>/fairino/` |
| `opentrons` | Opentrons OT-2 | [stainSTORM/OT2windy_service](https://github.com/stainSTORM/OT2windy_service) | `http://<machine>/opentrons/` |
| `webcam` | USB camera, to watch the robots | [stainSTORM/dockerwebcam](https://github.com/stainSTORM/dockerwebcam) | `http://<machine>/webcam/` |

Both are on by default, and the machine's home page links to them. On a robot's page:

1. Edit `config.yaml` (the robot's IP, and for the arm its stations) and press **Save and restart**.
2. Enter the Arkitekt server and press **Bind**. Open the approval link it shows in any browser and
   check the code. The login is stored, so the service reconnects after a reboot.

Each service only talks to its robot when an Arkitekt action runs, so an unused one just waits.
To switch it off, run `forklift plt disable-depl opentrons` (or `fairino`), then
`forklift plt apply`.

The config, teach points and stored login live in `/home/pi/arkitekt/<deployment>/`. To set up a
robot before the first boot, put its `config.yaml` on the SD card's boot partition, in
`init-root/as-pi/home/pi/arkitekt/<deployment>/`; it is moved into place at the next boot.

The Pi must be able to reach the robot's IP: on the same network, or on a direct cable with an
address in the robot's subnet. The OT-2's USB connection uses link-local `169.254.x.x` addresses.

The robot services use the host's network (`network_mode: host`; their pages listen on 8091 and
8092 and Caddy proxies to them). A robot cabled straight to the Pi gets an address from the Pi's
NetworkManager "shared" connection (eth0, eth1, usb0, usb1: `192.168.5-8.x`), and its nftables
rules reject new connections forwarded from a Docker bridge: from a bridged container the robot
answers "Connection refused" while `curl` on the Pi works.

The webcam reads `/dev/video0`; set `CAM_INDEX` in `deployments/webcam.pkg/deployment.compose.yml`
for another device. It starts without a camera and picks one up when it is plugged in.

To ship a new robot service version, push to its repo (GitHub Actions builds
`ghcr.io/stainstorm/<repo>:sha-<commit>`), then set that tag in
`deployments/<deployment>.pkg/deployment.compose.yml`.

## Usage

These are usage instructions for developers.

### Downloading an OS image

Go to this repo's [releases page](https://github.com/openUC2/os-rpi-arkitekt/releases), and
download the `.img.xz` SD card image file for the release or pre-release you want to run.
Flash it to an SD card with
[Raspberry Pi Imager](https://www.raspberrypi.com/software/).
The image works without internet at first boot.

### Deploying a published OS update to your machine

1. Once you've booted your machine into the OS, from a terminal (either the Cockpit terminal or
   an SSH remote session) you can run the following command to upgrade the local pallet to the
   latest commit on the branch the image was built from (`arkitekt-service` for images built from
   pushes to it):

   ```bash
   forklift plt upgrade
   ```

   If it gives you a warning that you may have changes in your local pallet which have not been
   committed/pushed up to GitHub, but you're sure that you won't lose any important changes by
   wiping your local pallet, then you should run:

   ```bash
   forklift plt upgrade --force
   ```

2. To apply all changes in the upgraded local pallet (including changes to OS configuration files),
   you should reboot or soft-reboot (e.g. via `sudo systemctl soft-reboot`).

   If you'd like to immediately apply changes to Docker apps before your next reboot, you can run:

   ```bash
   sudo systemctl restart forklift-apply
   ```

   If you are in an SSH session or you are in a GNU screen or byobu session in the Cockpit terminal,
   the following command will also work instead as an equivalent substitute to the above command:

   ```bash
   forklift stage apply
   ```

### Disabling/enabling functionalities

To disable a deployment on your RPi, e.g. the Opentrons service, run:

```bash
forklift plt disable-depl opentrons
```

To apply your modified configuration, then you can either

1) run `forklift plt apply`, or
2) run `forklift plt stage` and then reboot or soft-reboot (e.g. via `sudo systemctl soft-reboot`).

`forklift plt enable-depl opentrons` switches it on again. Note that `forklift plt upgrade --force`
discards such local changes.

To see the full list of deployments you can disable or enable, run `forklift plt ls-depl`. Note that
for some deployments, especially some deployments whose names begin with `provisioning/`,
`networking/`, `infra/`, and `dev/`, you will have to reboot/soft-reboot in order for changes to actually take
effect.

## Licensing

Any source code provided with this Forklift pallet is covered by the following information, except
where otherwise indicated (see also notes below on imported files & dependencies):

**Copyright openUC2 project contributors**

SPDX-License-Identifier: `MIT`

You can use the source code provided here under the
[MIT License](https://spdx.org/licenses/MIT.html).

### Imported Files & Dependencies

Forklift packages deployed by this pallet have their own software licenses, as specified in the
declaration files for those packages.
