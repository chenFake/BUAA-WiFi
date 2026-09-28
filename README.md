# 北航 BUAA-WiFi 下无浏览器 IoT 设备接入方法

- 以小爱音箱为例，使用 Ubuntu 临时克隆 MAC 完成校园网认证

北航 `BUAA-WiFi` 采用“开放 Wi-Fi + Web Portal”的认证方式。手机和电脑连接后，可以通过浏览器登录校园网账号；但小爱音箱等 IoT 设备通常没有浏览器，无法完成 Portal 认证，因此可能出现“已识别 Wi-Fi，但配网失败”的情况。

本次使用一种已实际验证的方法：**使用 Ubuntu 电脑临时克隆 IoT 设备的 MAC 地址，替设备完成一次校园网认证，再将网络交还给真实设备。**

测试环境：

```
校园网络：BUAA-WiFi
目标设备：小爱音箱
认证方式：Web Portal
操作系统：Ubuntu
无线网卡：MediaTek MT7921
```

该方法成功后，音箱直接连接校园网，无需电脑热点，无需路由器

---

## 一、基本原理

设 IoT 设备的 MAC 地址为：

```
50:92:6A:XX:XX:XX
```

操作过程为：

```
IoT 设备断电
      ↓
Ubuntu 无线网卡临时使用该 MAC
      ↓
连接 BUAA-WiFi
      ↓
浏览器完成校园网账号认证
      ↓
Ubuntu 断开并恢复原 MAC
      ↓
IoT 设备重新启动
      ↓
使用自己的 MAC 接入 BUAA-WiFi
```

实测中，认证状态可以由真实设备继续使用，因此小爱音箱能够完成联网。

需要说明的是，该方法依赖校园网当前的认证机制，后续若认证系统调整，效果可能发生变化。

---

## 二、获取 IoT 设备 MAC 地址

优先在设备 App 中查看 MAC 地址。

如果无法直接查看，小爱音箱进入配网模式后通常会建立类似以下名称的 Wi-Fi：

```
xiaomi-wifispeaker-lx06_miapxxxx
```

电脑连接该热点后执行：

```
ipconfig
```

若默认网关为：

```
10.0.0.1
```

继续执行：

```
arp -a
```

可能得到：

```
10.0.0.1    50-92-6A-XX-XX-XX
```

其中对应 `10.0.0.1` 的物理地址即为音箱 MAC。

---

## 三、先让设备尝试连接一次 BUAA-WiFi

先在米家或对应设备 App 中正常选择：

```
BUAA-WiFi
```

即使最终提示“配网失败”，也不必立即恢复出厂设置。

在本次测试中，小爱音箱虽然报告配网失败，但已经记录了 `BUAA-WiFi` 的 SSID。完成后续校园网认证后，设备可以直接重新尝试连接。

---

## 四、在 Ubuntu 中确认无线网卡

进入 Ubuntu，打开终端：

```
nmcli device status
```

例如：

```
DEVICE   TYPE  STATE
wlp2s0   wifi  connected
```

其中 `wlp2s0` 为无线网卡接口。

下面的脚本会自动识别无线接口，因此一般不需要手动修改接口名称。

---

## 五、检查并关闭代理

如果 Ubuntu 中配置过代理，可能导致校园网认证页面无法访问。

检查：

```
env | grep -i proxy
```

如果存在：

```
HTTP_PROXY=...
HTTPS_PROXY=...
http_proxy=...
https_proxy=...
```

临时清除：

```
unset http_proxy
unset https_proxy
unset HTTP_PROXY
unset HTTPS_PROXY
unset all_proxy
unset ALL_PROXY
```

同时关闭 GNOME 系统代理：

```
gsettings set org.gnome.system.proxy mode 'none'
```

确认：

```
gsettings get org.gnome.system.proxy mode
```

正常应返回：

```
'none'
```

浏览器 如Firefox 中也建议设置为：

```
设置 → 常规 → 网络设置 → 不使用代理
```

---

## 六、运行临时代认证脚本

执行前，**必须先将 IoT 设备完全断电**，避免真实设备与电脑同时使用相同 MAC 地址。

修改脚本中的：

```
TARGET_MAC="50:92:6a:xx:xx:xx"
```

为自己的设备 MAC。

然后将下面代码整体复制到 Ubuntu 终端：

```
bash <<'EOF'
set -u

SSID="BUAA-WiFi"

# 修改为目标设备 MAC
TARGET_MAC="50:92:6a:xx:xx:xx"

CONN="BUAA-IOT-TEMP"

IFACE=$(nmcli -t -f DEVICE,TYPE device status |
    awk -F: '$2=="wifi"{print $1; exit}')

if [ -z "$IFACE" ]; then
    echo "未找到 Wi-Fi 网卡。"
    exit 1
fi

ORIGINAL_MAC=$(cat "/sys/class/net/$IFACE/address")

cleanup() {
    echo
    echo "正在恢复电脑网络状态……"

    sudo nmcli connection down "$CONN" 2>/dev/null || true
    sudo nmcli connection delete "$CONN" 2>/dev/null || true

    sudo ip link set "$IFACE" down 2>/dev/null || true
    sudo ip link set "$IFACE" address "$ORIGINAL_MAC" 2>/dev/null || true
    sudo ip link set "$IFACE" up 2>/dev/null || true

    sleep 2

    echo "当前电脑 MAC："
    cat "/sys/class/net/$IFACE/address"
}

trap cleanup EXIT INT TERM

echo "无线接口：$IFACE"
echo "电脑原 MAC：$ORIGINAL_MAC"
echo "目标 MAC：$TARGET_MAC"

# 关闭代理
gsettings set org.gnome.system.proxy mode 'none' 2>/dev/null || true
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY all_proxy ALL_PROXY

# 清除旧临时配置
sudo nmcli connection down "$CONN" 2>/dev/null || true
sudo nmcli connection delete "$CONN" 2>/dev/null || true
sudo nmcli device disconnect "$IFACE" 2>/dev/null || true

sleep 2

# 创建 BUAA-WiFi 连接
sudo nmcli connection add \
    type wifi \
    ifname "$IFACE" \
    con-name "$CONN" \
    ssid "$SSID"

# 克隆目标设备 MAC
sudo nmcli connection modify "$CONN" \
    802-11-wireless.cloned-mac-address "$TARGET_MAC"

sudo nmcli connection modify "$CONN" \
    ipv4.method auto \
    ipv6.method auto \
    connection.autoconnect no

# 连接 BUAA-WiFi
sudo nmcli connection up "$CONN" ifname "$IFACE"

sleep 5

ACTUAL_MAC=$(cat "/sys/class/net/$IFACE/address")
IPV4=$(ip -4 -o addr show "$IFACE" |
    awk '{print $4}' |
    head -1)

echo
echo "当前 MAC：$ACTUAL_MAC"
echo "目标 MAC：$TARGET_MAC"
echo "IPv4：${IPV4:-未获得}"

if [ "${ACTUAL_MAC,,}" != "${TARGET_MAC,,}" ]; then
    echo "MAC 克隆失败，请勿继续认证。"
    exit 1
fi

if [ -z "$IPV4" ]; then
    echo "未获得 IPv4 地址，请勿继续认证。"
    exit 1
fi

echo
echo "MAC 克隆和 DHCP 均成功。"

curl --noproxy '*' \
    -I \
    --max-time 5 \
    http://gw.buaa.edu.cn \
    2>/dev/null |
    head -10 || true

echo
echo "请打开浏览器访问："
echo "https://gw.buaa.edu.cn"
echo
echo "完成校园网认证并确认可以正常访问互联网。"

xdg-open "https://gw.buaa.edu.cn" >/dev/null 2>&1 &

read -r -p \
"确认已经认证并可以正常上网后输入 YES：" \
OK </dev/tty

if [ "$OK" != "YES" ]; then
    echo "未确认认证成功，本次操作结束。"
    exit 0
fi

if curl --noproxy '*' \
        -I \
        --max-time 8 \
        https://www.baidu.com \
        >/dev/null 2>&1
then
    echo "互联网连接正常。"
else
    echo "互联网测试失败，请检查认证状态。"
fi

echo
echo "即将断开临时连接并恢复电脑原 MAC。"
echo "脚本结束后等待约 10 秒，再启动 IoT 设备。"

EOF
```

---

## 七、确认认证状态

脚本成功连接后，应看到类似：

```
当前 MAC：50:92:6a:xx:xx:xx
目标 MAC：50:92:6a:xx:xx:xx
IPv4：10.xxx.xxx.xxx/xx
```

这表示电脑已经以目标设备的 MAC 接入 `BUAA-WiFi`，并成功获得校园网 IP。

此时设备仍应保持断电。

浏览器访问：

```
https://gw.buaa.edu.cn
```

使用自己的校园网账号完成认证。

随后打开普通 HTTPS 网站确认网络正常，也可执行：

```
curl --noproxy '*' -I --max-time 8 https://www.baidu.com
```

确认联网后，回到脚本终端并输入：

```
YES
```

脚本将自动断开临时连接，并恢复电脑原来的 MAC 地址。

---

## 八、启动 IoT 设备

脚本结束后等待约 10 秒，再给设备通电。

不要立即重新配网或恢复出厂设置，先等待设备自动连接 `BUAA-WiFi`。

本次测试中，小爱音箱随后直接提示配网成功，并能够正常使用天气、音乐等联网功能。

---

## 九、常见问题

**Windows 中无法修改 MAC**

部分无线网卡驱动没有提供 `Network Address` 功能。即使修改注册表中的 `NetworkAddress`，驱动也可能不采用，因此本文最终使用 Ubuntu 完成。

**WSL 是否可以使用**

一般不建议。WSL2 默认使用虚拟网络接口，无法直接控制 Windows 正在使用的 PCIe 无线网卡，因此不适合本方法。

**连接 BUAA-WiFi 时提示需要 WEP 密钥**

不要给开放网络设置：

```
802-11-wireless-security.key-mgmt none
```

这可能被 NetworkManager 解释为 WEP 配置。对于 `BUAA-WiFi`，不创建 `802-11-wireless-security` 配置即可。

**浏览器提示“代理服务器拒绝连接”**

检查：

```
env | grep -i proxy
```

并关闭系统代理及浏览器代理。

**过一段时间再次断网**

校园网认证可能存在有效期。如果认证状态失效，可以重新执行：

```
设备断电
→ Ubuntu 克隆 MAC
→ 校园网认证
→ 恢复电脑 MAC
→ 设备重新启动
```

通常无需重新配置设备 Wi-Fi。

---

## 十、注意事项

该方法只应应用于**本人拥有的设备和本人校园网账号**。

操作过程中不要让真实设备与电脑同时使用同一个 MAC 地址。

不同网卡驱动和不同版本的 NetworkManager 对 MAC 克隆的支持情况可能不同，因此应以脚本输出的：

```
当前 MAC
```

为准，而不能只根据配置是否写入来判断是否成功。

---

## 总结

对于能够连接 Wi-Fi、但无法打开校园网 Web Portal 的 IoT 设备，可以利用 Linux 的 MAC 克隆能力，由电脑临时代表该设备完成校园网认证，再恢复电脑自身 MAC，让真实设备上线。

在本次测试中，这一方法成功解决了小爱音箱接入 `BUAA-WiFi` 的问题，并且认证完成后电脑无需持续运行。
