# Husky A200 — Step-by-Step Connection Guide

This guide explains how to connect a Mac to the PSU13 Clearpath Husky A200, log into its onboard computer, inspect ROS 2, and view live LiDAR data. It assumes the robot already has Ubuntu 24.04, ROS 2 Jazzy, and the Clearpath software installed.

Based on the [HuskyA200 repository](https://github.com/greenaandblue/HuskyA200/tree/10a9272193111f80aa2e31e0f74b96080be0ee81), reviewed on September 29, 2026. The instructions were checked against documentation; they have not been tested on the physical robot during this review.

## Project Summary

The repository records this robot's migration from ROS Kinetic to ROS 2 Jazzy, its network settings, daily operation, visualization, teleoperation, and SLAM commands. LiDAR troubleshooting, calibration, IMU/GPS integration, the first saved map, and autonomous navigation remain listed as unfinished work.

The configuration backups are incomplete at the reviewed commit: `config/robot.yaml` is missing, and `config/50-clearpath-bridge.yaml` is empty. Obtain working configuration files from the robot or its maintainer before attempting a reinstall. See the [reviewed repository contents](https://github.com/greenaandblue/HuskyA200/tree/10a9272193111f80aa2e31e0f74b96080be0ee81/config).

## Connection Details

| Item | Value for this robot |
|---|---|
| Robot | PSU13, Husky A200 |
| Serial number | `a200-0544` |
| ROS namespace | `a200_0544` — use an underscore |
| Robot IP | `192.168.131.1` |
| SSH username | `admin` |
| Wi-Fi network | `PSU13 Waypoint` |
| Waypoint router IP | `192.168.131.51` |
| Velodyne VLP-16 IP | `192.168.131.20`, UDP port `2368` |
| Foxglove connection | `ws://192.168.131.1:8765` |

These are installation-specific values from the [original README](https://github.com/greenaandblue/HuskyA200/blob/10a9272193111f80aa2e31e0f74b96080be0ee81/README.md). Ask the robot maintainer for the Wi-Fi and `admin` passwords; neither is provided in the repository.

## Before You Start

- Have the robot, a charged battery, and a Mac with Terminal available.
- Install the Foxglove desktop application if you want live visualization.
- Keep an Ethernet cable and a compatible Mac adapter available for the wired fallback.
- Keep the area clear and the emergency stop accessible. Network checks do not require driving; leave motion disabled until you are ready for a controlled movement test.

**Where commands run:** “Mac” means a local Terminal window. “Robot” means the terminal after a successful SSH login. All `ros2`, `apt`, and `systemctl` commands below run on the robot. You do not need a local ROS installation for this SSH/Foxglove workflow.

## Step 1 — Power On

Turn on the robot and wait approximately 1–2 minutes for the onboard computer and services to start. Confirm that the Waypoint router is powered before looking for its Wi-Fi network.

## Step 2 — Join the Robot Network

### Option A: Wi-Fi

On the Mac, open the Wi-Fi menu, select **PSU13 Waypoint**, and enter the network password. A “no Internet” indication does not by itself prevent local access to the robot.

### Option B: Direct Ethernet

If Wi-Fi is unavailable, connect the Mac to the robot's accessible Ethernet/debug port. In the Mac's network settings, select the Ethernet adapter and configure IPv4 manually:

| Setting | Value |
|---|---|
| Mac IP address | `192.168.131.99`, only if unused |
| Subnet mask | `255.255.255.0` |
| Router / gateway | Leave blank for this direct connection |
| DNS | Not required for this direct connection |

Do not assign the robot, router, or LiDAR address to the Mac. The wired fallback still requires the robot's own Ethernet configuration to be working. This follows Clearpath's [offboard networking guidance](https://docs.clearpathrobotics.com/docs/ros/installation/offboard_pc/).

## Step 3 — Check Network Access

**Mac:**

```bash
ping -c 4 192.168.131.1
nc -vz -G 5 192.168.131.1 22
```

Ping replies confirm IP reachability; a successful port-22 check confirms that an SSH endpoint is reachable. Some networks block ping, so also try the SSH login even if ping receives no replies.

If both fail, check the selected network, cable, and Mac IP address. On Wi-Fi, also try `ping -c 4 192.168.131.51`: reaching the router alone does not establish a connection to the robot.

## Step 4 — Log In with SSH

**Mac:**

```bash
ssh admin@192.168.131.1
```

On the first connection, verify the displayed host fingerprint with the maintainer before accepting it. Enter the robot's `admin` password when prompted. No characters appear while typing a password; this is normal.

After login, commands in that window run on the robot. Check:

```bash
whoami
hostname
```

`whoami` should return `admin`. Use this installation's account even if generic Clearpath documentation shows a different username. The general connection procedure is described in [Clearpath's SSH guide](https://docs.clearpathrobotics.com/docs/ros/networking/ssh_sftp/).

## Step 5 — Load ROS 2 and Check Services

**Robot:**

```bash
source /opt/ros/jazzy/setup.bash
source /etc/clearpath/setup.bash
echo "$ROS_DISTRO"
systemctl list-units --all 'clearpath*' --no-pager
ros2 topic list
```

The ROS distribution should be `jazzy`. Look for topics beginning with `/a200_0544/`. Repeat both `source` commands in every new SSH terminal used for ROS commands.

For the configured base and LiDAR, check that `clearpath-platform.service` and `clearpath-sensors.service` are running. Optional services may legitimately be inactive. For a failed service, inspect its logs:

```bash
journalctl -u clearpath-platform.service -n 80 --no-pager
journalctl -u clearpath-sensors.service -n 80 --no-pager
```

Clearpath uses separate services for the base and sensors; see the [service reference](https://docs.clearpathrobotics.com/docs/ros/config/services/).

## Step 6 — Check LiDAR Data

**Robot:**

```bash
ros2 topic info /a200_0544/sensors/lidar3d_0/points
ros2 topic hz /a200_0544/sensors/lidar3d_0/points
```

Let the rate command run for several seconds, then press **Ctrl+C**. The repository gives approximately 10 Hz as the intended rate, but also records an unresolved no-output issue. Treat a successful SSH login, visible topics, and actual sensor messages as separate checks.

If no messages arrive, inspect the sensor path:

```bash
ping -c 4 192.168.131.20
sudo tcpdump -ni any -c 20 udp port 2368
journalctl -u clearpath-sensors.service -n 100 --no-pager
```

Run `tcpdump` if installed; press **Ctrl+C** if no packets arrive. Incoming packets indicate network traffic from the LiDAR, but do not prove the ROS driver is publishing valid point clouds.

## Step 7 — Connect Foxglove on the Mac

Open the Foxglove desktop application, choose **Open connection → Foxglove WebSocket**, and enter:

```text
ws://192.168.131.1:8765
```

Add a **3D** panel, enable `/a200_0544/sensors/lidar3d_0/points`, and select `base_link` as the display frame if available. If transforms are missing, check `/a200_0544/tf` and `/a200_0544/tf_static`, and select a frame actually present in the data.

### If Foxglove cannot connect

**Mac:**

```bash
nc -vz -G 5 192.168.131.1 8765
```

**Robot:**

```bash
sudo ss -ltnp 'sport = :8765'
```

If no bridge is listening, start a temporary bridge in a robot SSH terminal after loading both setup files from Step 5:

```bash
# Install only if missing; requires robot Internet access and working ROS apt sources.
sudo apt update
sudo apt install ros-jazzy-foxglove-bridge

ros2 launch foxglove_bridge foxglove_bridge_launch.xml
```

Keep that terminal open while viewing data. Do not start a second bridge if one already owns port 8765. A listening port with a failed Mac connection points to network access, firewall, or listen-address settings.

The desktop app matches the repository's workflow. Foxglove also supports its web app in Chrome; browser permissions can affect local connections. See the [official ROS 2 connection guide](https://docs.foxglove.dev/docs/getting-started/frameworks/ros2).

## Step 8 — Confirm the Result

You have established robot access when SSH works and the robot's ROS environment exposes the expected topics. Live point-cloud messages and a working Foxglove view additionally confirm the sensor/visualization path.

When finished, stop any foreground diagnostic commands with **Ctrl+C**, then run `exit` to close SSH. Exiting SSH does not power off the robot.

## Optional — Test Manual Driving

Only proceed with a clear test area and an operator ready at the emergency stop. Release the emergency stop and confirm that the lockout permits motion when ready to drive.

For an already paired PS4 controller, hold **L1** and use the left stick for slow driving. For keyboard control, run the following **on the robot**, with the ROS environment loaded:

```bash
# Only if the keyboard teleoperation package is missing:
sudo apt install ros-jazzy-teleop-twist-keyboard

ros2 run teleop_twist_keyboard teleop_twist_keyboard \
  --ros-args -p stamped:=true \
  -p speed:=0.1 -p turn:=0.2 \
  -r cmd_vel:=/a200_0544/cmd_vel
```

Follow the displayed key bindings; press **k** to stop and **Ctrl+C** to exit. Stop other velocity publishers before testing. This installation uses `geometry_msgs/msg/TwistStamped` commands. See [Clearpath's driving guide](https://docs.clearpathrobotics.com/docs/ros/tutorials/driving/) and the [keyboard teleoperation package, version 2.4.1](https://github.com/ros2/teleop_twist_keyboard/blob/2.4.1/teleop_twist_keyboard.py).

## Troubleshooting

| Symptom | What to check |
|---|---|
| SSH times out | Robot power, Wi-Fi/cable, Mac address, and route to `192.168.131.1`. Try direct Ethernet. |
| SSH says “Permission denied” | Use `admin` and the current robot password. |
| SSH reports a changed host key | Confirm the robot identity and whether it was reinstalled. Only then remove the old entry on the Mac with `ssh-keygen -R 192.168.131.1` and reconnect. |
| `ssh.service` is inactive | On a local robot console, also check `systemctl status ssh.socket --no-pager`; socket activation can make an inactive service normal. |
| `ros2` is not found | Run the Step 5 setup commands on the robot. A missing setup file indicates an installation problem. |
| Expected topics are missing | Check the ROS environment and services. Use namespace `a200_0544`, with an underscore. |
| Foxglove connects but shows no cloud | Enable the point-cloud topic, confirm incoming messages, and check the display frame and transforms. |
| The robot will not move | Check the emergency stop, lockout, controller pairing, platform service, message type, and competing command sources. |

### If the robot's Ethernet configuration is broken

Use a monitor and keyboard connected to the robot if SSH is unavailable. Inspect the current configuration before changing it:

```bash
ip -br address
ip -br link
sudo netplan get
```

For the documented installation, `br0` should carry `192.168.131.1/24`. Reconstruct the configuration using the real interface names and existing network requirements; the empty repository file cannot restore it. In Netplan YAML, `bridges:` is a sibling of `ethernets:`, not nested inside it.

After an intentional configuration edit, validate it with `sudo netplan generate`, then use `sudo netplan try` from a local console. Keep local access available while testing network changes. See Clearpath's [robot installation and standard bridge instructions](https://docs.clearpathrobotics.com/docs/ros/installation/robot/).

## After Connection: Mapping and Configuration

The repository also provides SLAM and map-saving commands. Before using them, verify a working laser scan, transforms, and odometry. A network connection alone does not establish mapping readiness; consult the [original SLAM instructions](https://github.com/greenaandblue/HuskyA200/blob/10a9272193111f80aa2e31e0f74b96080be0ee81/README.md) once the sensor checks pass.

Use the robot's actual `/etc/clearpath/robot.yaml` as the source for this installation's configuration. For RViz on a separate Ubuntu 24.04 computer, follow [Clearpath's offboard setup](https://docs.clearpathrobotics.com/docs/ros/installation/offboard_pc/), including copying the real configuration and generating its ROS environment.
