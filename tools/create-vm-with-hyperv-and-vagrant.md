# Windows 10 使用 Hyper-V 和 Vagrant 创建虚拟机环境

以前我都是用 Vagrant + VirtualBox 快速创建虚拟机环境。通过 Vagrant 配置文件，我们可以快速初始化多个关联的虚拟机，并省去了设置网络和存储的时间。还可以将 Vagrant 项目直接转给别人，让别人快速搭建类似的环境。用了 Kubernetes Desktop 后，需要开启 Windows 的 Hyper-V，这样就无法使用 VirtualBox 了。所以，为了同时用 Kubernetes 和虚拟化，使用 Hyper-V 代替 VirtualBox会是一个自然的选择。不过目前 Vagrant 还不支持 Hyper-V 网络初始化，所以要有需要自定义的步骤。

本文创建 vagrant 项目的完整代码在[这里](https://github.com/rocksun/vagrant-hyperv)，大家直接使用。 

## 启用 Hyper-V 和 SMB 1.0/CIFS 文件共享支持

我们 Windows 10 默认没有开启，所以需要手工开启，使用管理员运行 Powershell ，然后执行：

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
Enable-WindowsOptionalFeature -Online -FeatureName "SMB1Protocol" -All
```

安装完后需要重启计算机。

## 安装 Vagrant

直接从[这里](https://www.vagrantup.com/downloads)下载安装即可，安装完后需要重启计算机。

因为要用到 reload 插件，所以还需要运行：


```cmd
vagrant plugin install vagrant-reload
```


## 准备 Vagrant 目录

准备一个目录，作为 vagrant 项目的根目录。添加文件 scripts\create-nat-hyperv-switch.ps1：

```powershell
# See: https://www.petri.com/using-nat-virtual-switch-hyper-v

If ("NATSwitch" -in (Get-VMSwitch | Select-Object -ExpandProperty Name) -eq $FALSE) {
    'Creating Internal-only switch named "NATSwitch" on Windows Hyper-V host...'

    New-VMSwitch -SwitchName "NATSwitch" -SwitchType Internal

    New-NetIPAddress -IPAddress 192.168.0.1 -PrefixLength 24 -InterfaceAlias "vEthernet (NATSwitch)"

    New-NetNAT -Name "NATNetwork" -InternalIPInterfaceAddressPrefix 192.168.0.0/24
}
else {
    '"NATSwitch" for static IP configuration already exists; skipping'
}

If ("192.168.0.1" -in (Get-NetIPAddress | Select-Object -ExpandProperty IPAddress) -eq $FALSE) {
    'Registering new IP address 192.168.0.1 on Windows Hyper-V host...'

    New-NetIPAddress -IPAddress 192.168.0.1 -PrefixLength 24 -InterfaceAlias "vEthernet (NATSwitch)"
}
else {
    '"192.168.0.1" for static IP configuration already registered; skipping'
}

If ("192.168.0.0/24" -in (Get-NetNAT | Select-Object -ExpandProperty InternalIPInterfaceAddressPrefix) -eq $FALSE) {
    'Registering new NAT adapter for 192.168.0.0/24 on Windows Hyper-V host...'

    New-NetNAT -Name "NATNetwork" -InternalIPInterfaceAddressPrefix 192.168.0.0/24
}
else {
    '"192.168.0.0/24" for static IP configuration already registered; skipping'
}
```

然后再添加 scripts\set-hyperv-switch.ps1 ：

```powershell
# See: https://www.thomasmaurer.ch/2016/01/change-hyper-v-vm-switch-of-virtual-machines-using-powershell/

Get-VM "homestead" | Get-VMNetworkAdapter | Connect-VMNetworkAdapter -SwitchName "NATSwitch"
```

增加修改 IP 的脚本，首先是 CentOS 的脚本 scripts\centos\configure-static-ip.sh ：

```bash
#!/bin/sh

echo 'Setting static IP address for Hyper-V...'

cat << EOF > /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
PREFIX=24
IPADDR=192.168.0.2
GATEWAY=192.168.0.1
DNS1=8.8.8.8
EOF

# Be sure NOT to execute "systemctl restart network" here, so the changes take
# effect on reboot instead of immediately, which would disconnect the provisioner.
```

然后是 Ubuntu 



然后准备一个名为 Vagrantfile 的文件： 

```ruby
Vagrant.configure("2") do |config|
    config.trigger.before :up do |trigger|
        trigger.info = "Creating 'NATSwitch' Hyper-V switch if it does not exist..."
    
        trigger.run = {privileged: "true", powershell_elevated_interactive: "true", path: "./scripts/create-nat-hyperv-switch.ps1"}
    end
    
    config.trigger.before :reload do |trigger|
        trigger.info = "Setting Hyper-V switch to 'NATSwitch' to allow for static IP..."
    
        trigger.run = {privileged: "true", powershell_elevated_interactive: "true", path: "./scripts/set-hyperv-switch.ps1"}
    end
    
    config.vm.provision "shell", path: "./scripts/centos/configure-static-ip.sh"
    
    config.vm.provision :reload

    config.vm.provider "hyperv" do |hv|
        hv.vmname = "homestead"
    end
    config.vm.hostname = "hypervhost"
    config.vm.box = "centos/7"
end
```

## 初次执行

使用管理员运行 powershell，进入 Vagrantfile 所在目录，运行：

```
vagrant up
```

当询问使用哪个 switch ，选择 “1) Default Switch”，然后就可以看到虚拟机启动又重启，如果没有报错，即可通过 ssh 客户端，使用 .\.vagrant\machines\default\hyperv 下的 private_key 访问 192.168.0.2 。

## 如何修改一些配置


如果需要修改 IP，需要修改 "./scripts/centos/configure-static-ip.sh" 中的内容。如果用的是 Ubuntu ，需要修改 “config.vm.provision "shell"” 执行 "./scripts/ubuntu/configure-static-ip.sh"。

如果修改的 IP 不是 192.168.0.* ，则还需要修改 switch 相关的脚本，先将网关配置好。

## 参考

* [Enable Hyper-V and Install Vagrant on Windows 10](https://computingforgeeks.com/enable-hyper-v-and-install-vagrant-in-windows/)
* [HyperV - Static Ip with Vagrant](https://superuser.com/questions/1354658/hyperv-static-ip-with-vagrant)