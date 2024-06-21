This post describes how to set up BlueField-2 (BF-2) development environment on CloudLab.

## Environment
r7525 servers in Clemson with Ubuntu 20.04 LTS

## Basic Setup
- Download NVIDIA SDK Manager installer [here](https://developer.nvidia.com/sdk-manager) (NVIDIA developer account login required) 

- Upload the installer (`sdkmanager_2.0.0-11402_amd64.deb`) to the server host and install it with `sudo apt install ./sdkmanager_2.0.0-11402_amd64.deb` 

- Install BF-2 drivers and DOCA SDK with `sdkmanager --cli install --logintype devzone --product DOCA --version 1.5.1 --targetos Linux --host --target BLUEFIELD2_DPU_TARGETS --flash all` (NVIDIA developer account login required) 

- During the above installation, it requires specifying the account password for DPU login (user: `ubuntu`, password: `ubuntu`)

- A complete installation also requires Internet access on BF-2. To enable that:
  - On the host (sudo), run `echo 1 | tee /proc/sys/net/ipv4/ip_forward` to allow IP forwarding and `iptables -t nat -A POSTROUTING -o [public IP interface, e.g., eno1] -j MASQUERADE` to enable NAT on the public interface
  
  - Update [Dec. 3, 2023]: "Burn Target Firmware" and "DOCA Software Package" installation failed. Need to make the following changes before proceeding to the two steps:
    - ***Probably NOT needed anymore***: Change IP of `google.com` and `packages.cloud.google.com` to `142.251.215.238` in `/etc/hosts`
    
    - Update [Jan. 29, 2024]: didn't need to change or do the other steps here, but did need to change `nvidia.com` to `23.33.40.145` in `/etc/hosts`, in order to continue with DOCA Software Package installation (this is only for checking internet connection which ping's `nvidia.com` who doesn't respond back)
    
    - Update [Mar 16]: kubernetes deprecated google packages and apt.kubernetes.io, however, nvidia sdk manager is still hard coded to use those ppa repo by **overwriting** the `kubernetes.list`, so I had to use the following, then be running a tight bash loop to overwrite back what nvidia sdk manager has overwritten
    
    - `sudo mkdir -p /etc/apt/keyrings`
    
    - ```shell
      echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
      
      curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
      ```
    
    - ~~`echo "deb [signed-by=/etc/apt/keyrings/kubernetes.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list`~~
    
    - ~~`curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes.gpg`~~
    
    - Up
    
  - On BF-2 (sudo), run `echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf` to set a DNS server and `ping google.com` to finally check Internet connection
    - To log into the DPU, `ssh ubuntu@192.168.100.2`
    
  - Update [Feb. 2, 2024]: Might also need to make sure unattended upgrade is not running during all this. If getting an error from `dpkg`, kill that process and `sudo apt-get purge unattended-upgrades`, or purge/remove at the beginning.
  
- The whole installation takes about an hour. Upon successful installation, ssh to BF-2 to access its resources

## Scalable Functions
All the operations below are performed on BF-2
- Make sure BF-2 is in the SmartNIC (Embedded Function) mode: `mlxconfig -d /dev/mst/mt41686_pciconf0 q | grep -i internal_cpu_model` 
  - Expected result is EMBEDDED_CPU(1)
  - Otherwise, make the change with `mlxconfig -d /dev/mst/mt41686_pciconf0 s INTERNAL_CPU_MODEL=1` and power cycle the machine
- To enable scalable functions (SFs), change the PCIe address for each port by `mlxconfig -d 0000:03:00.0 s PF_BAR2_ENABLE=0 PER_PF_NUM_SF=1 PF_TOTAL_SF=236
 PF_SF_BAR_SIZE=10`
  - Power cycle the machine after the change
- Add an SF with `/opt/mellanox/iproute2/sbin/mlxdevm port add pci/0000:03:00.0 flavour pcisf pfnum 0 sfnum 1`
  - Check the result with `/opt/mellanox/iproute2/sbin/mlxdevm port show`, which shows something like "pci/0000:03:00.0/229377:..." -- 229377 is the SF index
- Configure the SF with `/opt/mellanox/iproute2/sbin/mlxdevm port function set pci/0000:03:00.0/[sf_index, e.g., 229377] hw_addr 00:00:00:00:01:00 trust on state active`, where 00:00:00:00:01:00 is the mac address of the created SF
- Query the next SF id with `devlink dev show`
  - The SF id in the next command should be the largest shown id + 1
- Let the drive recognize this SF with `echo mlx5_core.sf.[sf id, e.g., 3] > /sys/bus/auxiliary/drivers/mlx5_core.sf_cfg/unbind` and `echo mlx5_core.sf.[sf id, e.g., 3] > /sys/bus/auxiliary/drivers/mlx5_core.sf/bind`
- Finally, check the result with `devlink dev show` again

## DOCA Build Setup
PATH must be configured on the BF-2/host to use DOCA libraries (e.g., to build DOCA sample code). Below are steps to configure BF-2
- `export PKG_CONFIG_PATH=${PKG_CONFIG_PATH}:/opt/mellanox/doca/lib/aarch64-linux-gnu/pkgconfig`
- `export PATH=${PATH}:/opt/mellanox/doca/tools`

To use DPDK:
- `export PKG_CONFIG_PATH=${PKG_CONFIG_PATH}:/opt/mellanox/dpdk/lib/aarch64-linux-gnu/pkgconfig`
- `export PKG_CONFIG_PATH=${PKG_CONFIG_PATH}:/opt/mellanox/dpdk/lib64/pkgconfig`

## References
- https://docs.nvidia.com/doca/sdk/installation-guide-for-linux/index.html
- https://medium.com/codex/getting-your-hands-dirty-with-mellanox-bluefield-2-dpus-deployed-in-cloudlabs-clemson-facility-bcb4e689c7e6
- https://docs.nvidia.com/doca/sdk/scalable-functions/index.html
- https://docs.nvidia.com/doca/archive/doca-v1.3/troubleshooting/index.html