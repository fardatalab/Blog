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
  
    - ```sh
      sudo echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
      sudo iptables -t nat -A POSTROUTING -o eno1 -j MASQUERADE
      ```
  
  - Update [Dec. 3, 2023]: "Burn Target Firmware" and "DOCA Software Package" installation failed. Need to make the following changes before proceeding to the two steps:
    - ***Probably NOT needed anymore***: Change IP of `google.com` and `packages.cloud.google.com` to `142.251.215.238` in `/etc/hosts`
  
    - Update [Jan. 29, 2024]: didn't need to change or do the other steps here, but did need to change `nvidia.com` to `23.33.40.145` in `/etc/hosts`, in order to continue with DOCA Software Package installation (this is only for checking internet connection which ping's `nvidia.com` who doesn't respond back)
  
    - ~~Update [Mar 16]: kubernetes deprecated google packages and apt.kubernetes.io, however, nvidia sdk manager is still hard coded to use those ppa repo by **overwriting** the `kubernetes.list`, so I had to use the following, then be running a tight bash loop to overwrite back what nvidia sdk manager has overwritten~~
  
    - ```shell
      # Update: all these are unnecessary with the newer SDK manager
      sudo mkdir -p /etc/apt/keyrings
      echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
      
      curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
      ```
      
    - ~~`echo "deb [signed-by=/etc/apt/keyrings/kubernetes.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list`~~
  
    - ~~`curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes.gpg`~~
  
  - On BF-2 (sudo), run `echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf` to set a DNS server and `ping google.com` to finally check Internet connection
    
    - ```sh
      echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
      echo "23.33.40.145 nvidia.com" | sudo tee -a /etc/hosts
      ```
    
    - To log into the DPU, `ssh ubuntu@192.168.100.2`
    
  - Update [Feb. 2, 2024]: Might also need to make sure unattended upgrade is not running during all this (`apt update` from DOCA will be unable to proceed in that case. If getting an error mentioning `dpkg`, kill that process and `sudo apt-get purge unattended-upgrades`, or purge/remove at the beforehand.
  
- The whole installation takes about an hour. Upon successful installation, ssh to BF-2 to access its resources

## Scalable Functions
All the operations below are performed on BF-2

if needed, `mst start; mst status` to find mst devices

- Make sure BF-2 is in the SmartNIC (Embedded Function) mode: `mlxconfig -d /dev/mst/mt41686_pciconf0 q | grep -i internal_cpu_model`  (41692 for BF-3)
  - Expected result is EMBEDDED_CPU(1)
  - Otherwise, make the change with `mlxconfig -d /dev/mst/mt41686_pciconf0 s INTERNAL_CPU_MODEL=1` and power cycle the machine
- To enable scalable functions (SFs), change the PCIe address for each port by `mlxconfig -d 0000:03:00.0 s PF_BAR2_ENABLE=0 PER_PF_NUM_SF=1 PF_TOTAL_SF=236 PF_SF_BAR_SIZE=10`

   - `devlink dev eswitch set pci/[0000:03:00.0] mode switchdev` if it says something about eswitch mode
   - Power cycle the machine after the change

- `devlink dev show` to check the sfnum to use, the sfnum value in the next command should be the largest shown + 1
- Add an SF with `/opt/mellanox/iproute2/sbin/mlxdevm port add pci/0000:03:00.0 flavour pcisf pfnum 0 sfnum 4` (Note: `sfnum` needs to bigger than existing ones(?), with `devlink dev show` )

  - Check the result with `/opt/mellanox/iproute2/sbin/mlxdevm port show`, which shows something like "pci/0000:03:00.0/229377:..." -- 229377 is the SF index
- Configure the SF with `/opt/mellanox/iproute2/sbin/mlxdevm port function set pci/0000:03:00.0/[sf_index, e.g., 229377] hw_addr 00:00:00:00:01:00 trust on state active`, where 00:00:00:00:01:00 is the mac address of the created SF
- ~~Query the next SF id with `devlink dev show`~~
  
  - ~~The SF id in the next command should be the largest shown id + 1 (Note: should be the same as the `sfnum`(?))~~
- Let the drive recognize this SF with `echo mlx5_core.sf.[sf id, e.g., 4] > /sys/bus/auxiliary/drivers/mlx5_core.sf_cfg/unbind` and `echo mlx5_core.sf.[sf id, e.g., 4] > /sys/bus/auxiliary/drivers/mlx5_core.sf/bind`

   - ```sh
      echo mlx5_core.sf.4 > /sys/bus/auxiliary/drivers/mlx5_core.sf_cfg/unbind
      echo mlx5_core.sf.4 > /sys/bus/auxiliary/drivers/mlx5_core.sf/bind
      ```

- Finally, check the result with `devlink dev show` again

## OVS Bridges
We can use Open vSwitch (OVS) bridges to configure the connectivity between different interfaces (physical ports, physical functions, virtual functions, etc.)
- Delete the old bridge: `ovs-vsctl del-br ovsbr1`
- Add a new bridge: `ovs-vsctl add-br sf_bridge1`
- Connect a physical port to the bridge: `ovs-vsctl add-port sf_bridge1 p0`
- ~~Connect an SF to the bridge: `ovs-vsctl add-port sf_bridge1 en3f0pf0sf1`~~
- Connect the new SF to the bridge: `ovs-vsctl add-port sf_bridge1 en3f0pf0sf4 `
- Add the second bridge: `ovs-vsctl add-br sf_bridge2`
- Connect the host-facing port (representor) to the bridge: `ovs-vsctl add-port sf_bridge2 pf0hpf`
- [Optionally?]Connect another SF to the bridge: `ovs-vsctl add-port sf_bridge2 en3f0pf0sf2`
- Check the result `ovs-vsctl show`

Note: we can also directly use the existing bridge

- put the SFs under the same port (e.g. p0) and its representor (pf0hpf)

## Host-BF2 RDMA
The host and BF-2 can communicate with RDMA through the CX-6 interface
- First, pick one of the SFs (e.g., `en3f0pf0sf4`) and configure its representor (`enp3s0f0s4`) with an IP address (in the local subnet): `ifconfig enp3s0f0s4 10.10.1.42/24` ==NOTE: use the actual subnet==
  - Note: ==not just any one, pick the one that's with `pf0hpf`== (i.e., `p0`, `pf0hpf` and the SF should be under the same ovs bridge)

- Configure the MTU value to, say 9000, on the port representor, otherwise default of 1024 seems to mess with RDMA `ib_read_lat` etc.

- Test if the above interface is RDMA-accessible with rping: `rping -s -a 10.10.1.42`

  - ```
    #server side (host)
    rping -s -a 10.10.1.2
    
    #client side (dpu)
    rping -c -a 10.10.1.2
    ```


  - `ib_read_lat` etc. would require an IB device, `mlx5_*`, the number should be the same as SF num, i.e. `4` in this case.
    - `sudo ibdev2netdev -v`
    - `ibstat` and `ibv_devinfo -d [mlx5_*]` can be useful


## DOCA Build Setup
PATH must be configured on the BF-2/host to use DOCA libraries (e.g., to build DOCA sample code). Below are steps to configure BF-2
- `export PKG_CONFIG_PATH=${PKG_CONFIG_PATH}:/opt/mellanox/doca/lib/aarch64-linux-gnu/pkgconfig`
- `export PATH=${PATH}:/opt/mellanox/doca/tools`

To use DPDK:
- `export PKG_CONFIG_PATH=${PKG_CONFIG_PATH}:/opt/mellanox/dpdk/lib/aarch64-linux-gnu/pkgconfig`
- `export PKG_CONFIG_PATH=${PKG_CONFIG_PATH}:/opt/mellanox/dpdk/lib64/pkgconfig`

## Client-Server Setup
To set up a client-server cluster where the client connects to the server with a 100 Gbps link (BF-2 is on the server), we use r6525 as the client (the server is still r7525).
Below shows the CloudLab profile (`smartnic-2nodes` in fardatalab).

```
"""two nodes, one of which has a smartnic

Instructions:
boot it up"""

#
# NOTE: This code was machine converted. An actual human would not
#       write code like this!
#

# Import the Portal object.
import geni.portal as portal
# Import the ProtoGENI library.
import geni.rspec.pg as pg
# Import the Emulab specific extensions.
import geni.rspec.emulab as emulab

# Create a portal object,
pc = portal.Context()

# Create a Request object to start building the RSpec.
request = pc.makeRequestRSpec()

# Node node-0
node_0 = request.RawPC('node-0')
node_0.hardware_type = 'r7525'
node_0.disk_image = 'urn:publicid:IDN+emulab.net+image+emulab-ops//UBUNTU20-64-STD'
iface0 = node_0.addInterface('interface-1')
iface0.component_id = "eth4"

# Node node-1
node_1 = request.RawPC('node-1')
node_1.hardware_type = 'r6525'
node_1.disk_image = 'urn:publicid:IDN+emulab.net+image+emulab-ops//UBUNTU20-64-STD'
iface1 = node_1.addInterface('interface-0')
iface1.component_id = "eth4"

# Link link-0
link_0 = request.Link('link-0')
link_0.Site('undefined')
link_0.addInterface(iface1)
link_0.addInterface(iface0)


# Print the generated rspec
pc.printRequestRSpec(request)
```

The two machines are expected to connect with the 10.10.1.0/24 network. 
However, for some reason, the CX-6 interface in server's BF-2 was not properly set up.
We had to manually configure the interface (`ens5f0`) using `netplan` and `/etc/network/interfaces`:

First, set `/etc/netplan/bf_config.yaml` as
```
network:
  version: 2
  renderer: networkd
  ethernets:
    tmfifo_net0:
      addresses: [192.168.100.1/24]
      dhcp4: no
    ens5f0:
      addresses: [10.10.1.2/24]
      dhcp4: no
```
And run `sudo netplan apply`.

Then, set `/etc/network/interfaces` as
```
auto ens5f0
iface ens5f0 inet static
        address 10.10.1.2
        netmask 255.255.255.0
        gateway 10.10.1.1
```
And restart the network: `sudo /etc/init.d/networking restart`. 

Finally, reboot the machine, and you'll see the network working:
```
node-0:~> ping 10.10.1.1
PING 10.10.1.1 (10.10.1.1) 56(84) bytes of data.
64 bytes from 10.10.1.1: icmp_seq=1 ttl=64 time=0.115 ms
64 bytes from 10.10.1.1: icmp_seq=2 ttl=64 time=0.084 ms
64 bytes from 10.10.1.1: icmp_seq=3 ttl=64 time=0.075 ms
64 bytes from 10.10.1.1: icmp_seq=4 ttl=64 time=0.078 ms
64 bytes from 10.10.1.1: icmp_seq=5 ttl=64 time=0.081 ms
```

We can now send traffic from the client to the server through BF-2.

## Useful Commands
- `sudo ibdev2netdev -v`: maps IB devices to network ports
- or maybe use `ibv_devinfo -d [mlx5_2]` `ibstat` (for non Nvidia NICs?)

## References
- https://docs.nvidia.com/doca/sdk/installation-guide-for-linux/index.html
- https://medium.com/codex/getting-your-hands-dirty-with-mellanox-bluefield-2-dpus-deployed-in-cloudlabs-clemson-facility-bcb4e689c7e6
- https://docs.nvidia.com/doca/sdk/scalable-functions/index.html
- https://docs.nvidia.com/doca/archive/doca-v1.3/troubleshooting/index.html
- https://gist.github.com/githubfoam/da75951b97e9aec21dcebadf68a6a360
- https://serverspace.us/support/help/configuring-the-network-interface-in-ubuntu-18-04/