# Windows 共享网络到 RK3568 并配置 SSH

## 1. 连接方式

适用于：电脑通过校园网/Wi-Fi 上网，RK3568 通过网线直连电脑。

```text
校园网/Wi-Fi
   ↓
Windows 电脑
   ↓ 网线
RK3568 开发板
```

如果开发板自带网口可用，直接用网线连接电脑和开发板，不需要 USB 网卡。

---

## 2. Windows 开启网络共享

1. 按 `Win + R`
2. 输入：

```text
ncpa.cpl
```

3. 找到正在上网的网络，例如 `WLAN` 或 `Wi-Fi`
4. 右键 → `属性` → `共享`
5. 勾选：

```text
允许其他网络用户通过此计算机的 Internet 连接来连接
```

6. 家庭网络连接选择连接开发板的那个 `以太网`
7. 确认后，Windows 通常会把电脑有线网口设置为：

```text
192.168.137.1
```

---

## 3. 开发板配置静态 IP

在开发板串口终端中查看网卡：

```bash
ip addr
```

如果看到：

```text
eth1: LOWER_UP
```

说明 `eth1` 是当前连接电脑的网口。

手动设置 IP：

```bash
ip link set eth1 up
ip addr flush dev eth1
ip addr add 192.168.137.2/24 dev eth1
ip route replace default via 192.168.137.1 dev eth1
echo "nameserver 114.114.114.114" > /etc/resolv.conf
```
## 若是永久写入
cat > /etc/network/interfaces.d/eth1 <<'EOF'
auto eth1
allow-hotplug eth1

iface eth1 inet static
    address 192.168.137.2
    netmask 255.255.255.0
    gateway 192.168.137.1
    dns-nameservers 114.114.114.114 223.5.5.5
EOF

检查：

```bash
ip addr show eth1
ip route
```

应该看到：

```text
inet 192.168.137.2/24
default via 192.168.137.1 dev eth1
```

测试电脑和开发板是否连通：

```bash
ping -c 3 192.168.137.1
```

能 ping 通就说明电脑和开发板的局域网已经正常。

---

## 4. 启动或重启 SSH 服务

查看 SSH 服务：

```bash
systemctl status ssh --no-pager || systemctl status sshd --no-pager
```

重启 SSH：

```bash
systemctl restart ssh || systemctl restart sshd
```

设置开机自启：

```bash
systemctl enable ssh || systemctl enable sshd
```

---

## 5. root 登录问题

Debian/OpenSSH 默认可能禁止 root 密码登录。

如果要允许 root 通过 SSH 登录：

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sed -i 's/^#\?PermitRootLogin .*/PermitRootLogin yes/' /etc/ssh/sshd_config
sed -i 's/^#\?PasswordAuthentication .*/PasswordAuthentication yes/' /etc/ssh/sshd_config
grep -q '^PermitRootLogin ' /etc/ssh/sshd_config || echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config
grep -q '^PasswordAuthentication ' /etc/ssh/sshd_config || echo 'PasswordAuthentication yes' >> /etc/ssh/sshd_config
sshd -t
systemctl restart ssh || systemctl restart sshd
```

设置 root 密码：

```bash
passwd root
```

确认配置：

```bash
grep -E "PermitRootLogin|PasswordAuthentication|UsePAM" /etc/ssh/sshd_config
```

期望看到：

```text
PermitRootLogin yes
PasswordAuthentication yes
UsePAM yes
```

---

## 6. VS Code Remote SSH 配置

Windows 用户 SSH 配置文件：

```text
C:\Users\你的用户名\.ssh\config
```

添加：

```sshconfig
Host atompi-rk3568
    HostName 192.168.137.2
    User root
    Port 22
```

VS Code 中执行：

```text
Remote-SSH: Connect to Host → atompi-rk3568
```

---

## 7. 常见问题

### 只 ping 通 192.168.137.1，ping 不通外网

说明电脑和开发板直连正常，但 Windows 网络共享/NAT 或校园网共享可能有问题。

这不影响 SSH、SCP、VS Code Remote SSH。

### `dhclient eth1` 卡住

说明 Windows 没有给开发板正常分配 DHCP。可以直接使用本文的静态 IP 配置。

### VS Code 提示密码错误

优先检查是否是 root 登录被禁止，并确认 `PermitRootLogin yes` 和 `PasswordAuthentication yes` 已生效。
## cmd的ssh命令
ssh root@192.168.137.2