This post describes how to upgrade to [DOCA v3.1.x](https://docs.nvidia.com/doca/archive/3-1-0/index.html) (from v2.5.y) on a server with a single BlueField-3 (BF-3).

## Environment

### Host
```[sh]
host$ hostnamectl
 Static hostname: farnet1
       Icon name: computer-server
         Chassis: server
      Machine ID: 380c2d62777143b2a91e06337b6f61b0
         Boot ID: f96e67d7123947e0aeae53c225e1f078
Operating System: Ubuntu 22.04.5 LTS
          Kernel: Linux 5.15.0-122-generic
    Architecture: x86-64
 Hardware Vendor: Dell Inc.
  Hardware Model: PowerEdge R7625
```

### DPU
```[sh]
bf-3$ hostnamectl
   Static hostname: n/a
Transient hostname: localhost
         Icon name: computer
        Machine ID: 4c3eb0bb1032439b91c929718ae2c32e
           Boot ID: 4de3b2a868994fd1adab10cca05130c1
  Operating System: Ubuntu 22.04.4 LTS
            Kernel: Linux 5.15.0-1045-bluefield
      Architecture: arm64
   Hardware Vendor: https://www.mellanox.com
    Hardware Model: BlueField-3 SmartNIC Main Card
```

## Update DOCA on the host

This phase of the installation process is based on the [documentation](https://docs.nvidia.com/doca/archive/3-1-0/installation+and+setup/index.html) provided by NVIDIA.

1. Since we are upgrading from DOCA v2.5.y, we begin by removing all DOCA and OFED related packages.

    ```[sh]
    host$ for f in $( dpkg --list | grep -E 'doca|flexio|dpa-gdbserver|dpa-stats|dpa-resource-mgmt|dpaeumgmt' | awk '{print $2}' ); do echo $f ; sudo apt remove --purge $f -y ; done
    host$ sudo /usr/sbin/ofed_uninstall.sh --force
    host$ sudo apt-get autoremove
    ```

2. Install the [DOCA host repository](https://developer.nvidia.com/doca-downloads).

    ```[sh]
    host$ wget https://www.mellanox.com/downloads/DOCA/DOCA_v3.1.0/host/doca-host_3.1.0-091000-25.07-ubuntu2204_amd64.deb
    host$ sudo dpkg -i doca-host_3.1.0-091000-25.07-ubuntu2204_amd64.deb
    host$ sudo apt-get update
    host$ sudo apt-get install -y doca-all mlnx-fw-updater
    ```

3. Reload the `openibd` service.

    ```[sh]
    host$ sudo /etc/init.d/openibd restart
    ```

4. Initialize the NVIDIA Firmware Tools (MFT). (This was previously known as the Mellanox Software Tools.)

    ```[sh]
    host$ sudo mst restart
    ```

### SoC Management Interface

The [SoC management interface](https://docs.nvidia.com/networking/display/bluefieldbsp4120/soc+management+interface+(rshim)) (RShim) allows the host to operate the BF-3 monitor its operational state.
If `rshim` is not running, it can be restarted with the following:

```[sh]
host$ sudo systemctl restart rshim
host$ systemctl status rshim
● rshim.service - rshim driver for BlueField SoC
     Loaded: loaded (/lib/systemd/system/rshim.service; disabled; vendor preset: enabled)
     Active: active (running) since Fri 2025-10-17 16:06:17 EDT; 6min ago
       Docs: man:rshim(8)
    Process: 89657 ExecStart=/usr/sbin/rshim $OPTIONS (code=exited, status=0/SUCCESS)
   Main PID: 89658 (rshim)
      Tasks: 8 (limit: 154011)
     Memory: 964.0K
        CPU: 2.124s
     CGroup: /system.slice/rshim.service
             └─89658 /usr/sbin/rshim
```

If it is running properly, it exposes a `sysfs` device `/dev/rshim0/*` and a virtual Ethernet interface `tmfifo_net0`.
The following displays the status of `rshim` and logs for device initialization:

```[sh]
host$ sudo cat /dev/rshim0/misc
DISPLAY_LEVEL   2 (0:basic, 1:advanced, 2:log)
BF_MODE         DPU mode
BOOT_MODE       1 (0:rshim, 1:emmc, 2:emmc-boot-swap)
BOOT_TIMEOUT    300 (seconds)
USB_TIMEOUT     40 (seconds)
DROP_MODE       0 (0:normal, 1:drop)
SW_RESET        0 (1: reset)
DEV_NAME        pcie-0000:e1:00.2
DEV_INFO        BlueField-3(Rev 1)
OPN_STR         N/A
UP_TIME         4172(s)
SECURE_NIC_MODE 0 (0:no, 1:yes)
FORCE_CMD       0 (1: send Force command)
---------------------------------------
             Log Messages
---------------------------------------
 INFO[MISC]: DPU is ready
 INFO[MISC]: : DPU is ready
```

Further information on the features and functionality of the management interface can be found in the [documentation](https://docs.nvidia.com/networking/display/bluefieldbsp4120/soc+management+interface+(rshim)).
The network settings on both host and BF-3 can now be configured.

## Install BF-Bundle on the DPU

This phase of the installation process is based on this [guide](https://docs.nvidia.com/doca/archive/3-1-0/bf-bundle+installation+and+upgrade/index.html) provided by NVIDIA.

1. Download the BF-Bundle [BFB image](https://developer.nvidia.com/doca-3-1-0-core-update-downloads) onto the host.

2. Install the BF-Bundle using `rshim`.

    ```[sh]
    host$ sudo bfb-install --bfb bf-bundle-3.1.0-76_25.07_ubuntu-22.04_prod.bfb --rshim rshim0
    ```

3. After installation, wait for a minute or two and verify the BF-3 device has completed the boot sequence with the following command:
    
    ```[sh]
    host$ sudo cat /dev/rshim0/misc
    ```

4. View the packages installed (as part of BF-Bundle) and their versions.

    ```[sh]
    bf-3$ sudo bf-info
    ```

## Configure Network Settings

* This command configures the host side of `tmfifo_net0` with a static IP and enables IPv4-based communication to the BlueField OS:

    ```[sh]
    host$ sudo ip addr add dev tmfifo_net0 192.168.100.1/30
    ```

    This enables logging into the BlueField-3 operating system over the virtual ethernet interface:

    ```[sh]
    host$ ssh ubuntu@192.168.100.2
    ```

* On the DPU, if the default route is through `tmfifo_net0` and the OOB interface has not been assigned an IP address (through DHCP), the DPU will have no outbound connectivity.
    This can be checked with the following commands:

    ```[ssh]
    bf-3$ ip route
    default via 192.168.100.1 dev tmfifo_net0 proto static metric 1025
    192.168.100.0/30 dev tmfifo_net0 proto kernel scope link src 192.168.100.2 metric 100
    bf-3$ nmcli con show
     NAME                 UUID                                  TYPE      DEVICE
    netplan-tmfifo_net0  77742971-069b-34fa-9761-413eb2bd29a4  ethernet  tmfifo_net0
    netplan-oob_net0     aa93b667-6aac-3804-91e9-4958e07fdb2f  ethernet  -- 
    ```

    This command enables the host to forward packets from the DPU (`tmfifo_net0`) to the internet:

    ```[sh]
    host$ ip route
    default via 10.70.0.254 dev eno8303 proto dhcp src 10.70.60.11 metric 100
    10.70.0.0/16 dev eno8303 proto kernel scope link src 10.70.60.11 metric 100
    10.70.0.254 dev eno8303 proto dhcp scope link src 10.70.60.11 metric 100
    host$ sudo sysctl -w net.ipv4.ip_forward=1
    host$ sudo iptables -t nat -A POSTROUTING -o eno8303 -j MASQUERADE
    ```

    Here, `eno8303` is the internet-facing interface on the host.
    To manually configure the DNS resolver on the DPU, these two commands can be used:

    ```[sh]
    bf-3$ echo -e "[main]\ndns=none" | sudo tee /etc/NetworkManager/conf.d/90-dns-none.conf > /dev/null
    bf-3$ echo -e "nameserver 1.1.1.1\nnameserver 8.8.8.8" | sudo tee /etc/resolv.conf > /dev/null
    ```

    The first line disallows the `NetworkManager` service from managing the DNS settings while the second sets the DNS servers.
    To check if this succeeded, reload the `NetworkManager` service and check the `/etc/resolv.conf` file.

    ```[sh]
    bf-3$ systemctl reload NetworkManager
    bf-3$ cat /etc/resolv.conf
    ```

* Before trying to reboot the BlueField-3 system, check whether the environment supports it with this query:

    ```[sh]
    host$ sudo mlxfwreset -d /dev/mst/mt41692_pciconf0 q
    ```

    If the output includes the following two lines, then system reboots are supported:
    
    ```[sh]
    3: Driver restart and PCI reset                                  -Supported     (default)
    ```
    ```[sh]
    1: NIC Driver is the owner                                       -Supported     (default)
    ```

    Note these are not sequential.
    A BlueField-3 system reboot command can then be issued:

    ```[sh]
    host$ sudo mlxfwreset -d /dev/mst/mt41692_pciconf0 -y -l 3 --sync 1 r
    ```
    
    See the [documentation](https://docs.nvidia.com/networking/display/bluefieldbsp4120/appendix+-+nvidia+bluefield+reset+and+reboot+procedures#src-4161919038_AppendixNVIDIABlueFieldResetandRebootProcedures-System-levelResetforBlueFieldinDPUModeSystem-levelResetforBlueFieldinDPUMode) for other ways to reset the BlueField-3.

<!-- [Installation guide](https://docs.nvidia.com/doca/archive/3-1-0/bf-bundle+installation+and+upgrade/index.html)

Add DOCA online repository
```[sh]
bf-3$ export DOCA_REPO="https://linux.mellanox.com/public/repo/doca/3.1.0/ubuntu22.04/dpu-arm64"
bf-3$ curl $DOCA_REPO/GPG-KEY-Mellanox.pub | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/GPG-KEY-Mellanox.pub > /dev/null
bf-3$ echo "deb [signed-by=/etc/apt/trusted.gpg.d/GPG-KEY-Mellanox.pub] $DOCA_REPO ./" | sudo tee /etc/apt/sources.list.d/doca.list > /dev/null
```

Update index
```[sh]
bf-3$ sudo apt-get update
bf-3$ sudo apt-get install bf-fwbundle
``` -->