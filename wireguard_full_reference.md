# সম্পূর্ণ সেটআপ গাইড: WireGuard + Mikhmon + Residential BD VLESS Node

> এই গাইডটি ধারাবাহিকভাবে তিনটি বড় কাজ কভার করে:
> ১. **MikroTik রাউটার ↔ VPS** এর মধ্যে WireGuard টানেল তৈরি
> ২. **Mikhmon** হোস্টিং ও কানেকশন
> ৩. **Residential BD IP** দিয়ে VLESS নোড চালানো (চীন বা যেকোনো দেশের জন্য)

---

## 🗺️ আর্কিটেকচার (পুরো সিস্টেমটি কীভাবে কাজ করে)

```
[ক্লায়েন্ট (চীনে)]
       |
       | VLESS Connection
       ↓
[VPS - 31.97.215.132]
  ├── V2bX (Node 1)  →  সরাসরি VPS IP দিয়ে ইন্টারনেট
  └── V2bX-BD (Node 7) → WireGuard Tunnel → MikroTik → BD ISP → ইন্টারনেট
       |
       | WireGuard Tunnel (10.0.0.0/24, UDP 51820)
       ↓
[MikroTik hAP ax lite - বাসায়]
  └── BD Broadband ISP → ইন্টারনেট (Residential IP)
```

**টানেলের আইপি ম্যাপিং:**
| ডিভাইস | WireGuard আইপি | পাবলিক আইপি |
|---|---|---|
| VPS (Server) | 10.0.0.1 | 31.97.215.132 |
| MikroTik (Client) | 10.0.0.2 | Dynamic (ISP) |

**WireGuard কী (Keys):**
| | Private Key | Public Key |
|---|---|---|
| VPS | `sTChSa8KY4/bfWtq7Bu3xQO373Q0dZuZKRvnP4AiYss=` | `6nfKUt+ivFb/CxCR4xdU6NROVFpBlAPXUo2hBqLyuiA=` |
| MikroTik | `hJ5ljMd1ETlvULZCaZdJK0e/uzlhIZJrIQ6xW/iwoCg=` | `RtL4ZV+iYM+VP2M5Fq9p/OYNV41g0qu4dp8pH61Wyhw=` |

---

## ✅ পর্ব ১: VPS-এ WireGuard সার্ভার সেটআপ

VPS-এর টার্মিনালে (SSH বা aaPanel Terminal) নিচের পুরো ব্লকটি একবারে কপি-পেস্ট করুন:

```bash
# ধাপ ১: WireGuard ইনস্টল করা
apt update && apt install wireguard -y

# ধাপ ২: ফোল্ডার তৈরি ও পারমিশন সেট
mkdir -p /etc/wireguard
chmod 700 /etc/wireguard

# ধাপ ৩: কনফিগারেশন ফাইল তৈরি
# (Table = off দিলে VPS-এর মেইন ইন্টারনেট রাউটিং নষ্ট হয় না)
# (AllowedIPs = 0.0.0.0/0 দিলে রাউটার সব ইন্টারনেট ট্রাফিক ফরোয়ার্ড করতে পারে)
cat <<EOF > /etc/wireguard/wg0.conf
[Interface]
PrivateKey = sTChSa8KY4/bfWtq7Bu3xQO373Q0dZuZKRvnP4AiYss=
Address = 10.0.0.1/24
ListenPort = 51820
Table = off

[Peer]
PublicKey = RtL4ZV+iYM+VP2M5Fq9p/OYNV41g0qu4dp8pH61Wyhw=
AllowedIPs = 0.0.0.0/0
EOF

# ধাপ ৪: ফায়ারওয়্যালে পোর্ট খোলা
ufw allow 51820/udp 2>/dev/null || true

# ধাপ ৫: WireGuard চালু করা এবং অটো-স্টার্ট সেট করা
wg-quick down wg0 2>/dev/null || true
wg-quick up wg0
systemctl enable wg-quick@wg0

# ধাপ ৬: চেক করা (peer দেখালে সফল)
wg show
```

> ⚠️ **গুরুত্বপূর্ণ:** MikroTik-এ WireGuard সেটআপ করার পরে যদি Peer-এর Public Key পরিবর্তন হয়, তাহলে নিচের কমান্ড দিয়ে VPS-এ আপডেট করতে হবে এবং রিস্টার্ট দিতে হবে:
> ```bash
> wg set wg0 peer [ROUTER_PUBLIC_KEY] allowed-ips 0.0.0.0/0
> ```

---

## ✅ পর্ব ২: MikroTik রাউটারে WireGuard ক্লায়েন্ট সেটআপ

### পদ্ধতি ক: Winbox GUI দিয়ে

**ধাপ ১ — WireGuard ইন্টারফেস তৈরি:**
1. Winbox বাম মেনু → **WireGuard** → প্রথম ট্যাব-এ থাকুন
2. **Plus (+)** বাটন চাপুন
3. নিচের তথ্য দিন:
   - **Name:** `wireguard-mikhmon`
   - **Listen Port:** `51820`
   - **Private Key:** `hJ5ljMd1ETlvULZCaZdJK0e/uzlhIZJrIQ6xW/iwoCg=`
4. **Apply** চাপুন → Public Key অটো-জেনারেট হবে → **OK**

**ধাপ ২ — VPS কে Peer হিসেবে যোগ করা:**
1. একই WireGuard উইন্ডোর → **Peers** ট্যাবে যান
2. **Plus (+)** চাপুন
3. নিচের তথ্য দিন:
   - **Interface:** `wireguard-mikhmon`
   - **Endpoint:** `31.97.215.132`
   - **Endpoint Port:** `51820`
   - **Public Key:** `6nfKUt+ivFb/CxCR4xdU6NROVFpBlAPXUo2hBqLyuiA=`
   - **Allowed Address:** `10.0.0.1/32`
   - **Persistent Keepalive:** `25s`
4. **Apply** → **OK**

**ধাপ ৩ — টানেলের আইপি বসানো:**
1. Winbox বাম মেনু → **IP** → **Addresses**
2. **Plus (+)** চাপুন
3. নিচের তথ্য দিন:
   - **Address:** `10.0.0.2/24`
   - **Interface:** `wireguard-mikhmon`
4. **Apply** → **OK**

---

### পদ্ধতি খ: MikroTik Terminal (Winbox → New Terminal) দিয়ে

```ros
# ধাপ ১: ইন্টারফেস তৈরি
/interface wireguard add listen-port=51820 mtu=1420 name=wireguard-mikhmon private-key="hJ5ljMd1ETlvULZCaZdJK0e/uzlhIZJrIQ6xW/iwoCg="

# ধাপ ২: VPS-কে Peer হিসেবে যোগ
/interface wireguard peers add allowed-address=10.0.0.1/32 endpoint-address=31.97.215.132 endpoint-port=51820 interface=wireguard-mikhmon public-key="6nfKUt+ivFb/CxCR4xdU6NROVFpBlAPXUo2hBqLyuiA=" persistent-keepalive=25s

# ধাপ ৩: আইপি বসানো
/ip address add address=10.0.0.2/24 interface=wireguard-mikhmon
```

---

## ✅ পর্ব ৩: কানেকশন যাচাই (Ping Test)

VPS টার্মিনালে রান করুন:
```bash
ping -c 4 10.0.0.2
```

যদি `0% packet loss` আসে, তাহলে টানেল সফল। যদি না আসে, নিচে দেখুন:

**সমস্যা হলে:** MikroTik-এর Winbox-এ WireGuard Interface-এ গিয়ে Public Key কপি করুন এবং VPS-এ আপডেট করুন:
```bash
wg set wg0 peer [ROUTER_REAL_PUBLIC_KEY] allowed-ips 0.0.0.0/0
```

---

## ✅ পর্ব ৪: Mikhmon কানেকশন

1. ব্রাউজারে যান: `https://wifi.swdrana.com/admin.php`
2. লগইন করুন
3. **Session Settings** → **IP MikroTik** বক্সে লিখুন: `10.0.0.2`
4. Username ও Password দিন → **Connect**

Mikhmon এখন আপনার বাসার MikroTik-এর সাথে কথা বলবে WireGuard টানেলের মাধ্যমে।

---

## ✅ পর্ব ৫: Residential BD IP দিয়ে VLESS Node (V2bX-BD)

এই সেটআপে, xBoard-এর **Node 7** শুধুমাত্র আপনার বাসার MikroTik রাউটারের ISP IP ব্যবহার করে ইন্টারনেটে বের হবে।

### ৫.১ — MikroTik-এ NAT রুল যোগ করা
Winbox → New Terminal:
```ros
# VPS থেকে আসা ট্রাফিক রাউটারের নিজের IP দিয়ে বের করার পারমিশন
/ip firewall nat add chain=srcnat action=masquerade src-address=10.0.0.0/24 comment="NAT for V2bX-BD VPS"
```

### ৫.২ — VPS-এ Linux Policy Routing সেটআপ
```bash
# ফাঁকা রাউটিং টেবিল তৈরি এবং Mark 255 ট্রাফিককে wg0 দিয়ে পাঠানো
ip rule add fwmark 255 table 200
ip route add default dev wg0 table 200

# IPv6 Leak বন্ধ করা (খুবই গুরুত্বপূর্ণ!)
# এটি না দিলে ক্লায়েন্ট VPS-এর USA IPv6 দেখবে, BD IP নয়
ip -6 rule add fwmark 255 table 200
ip -6 route add unreachable default table 200

# Reverse Path Filtering রিল্যাক্স করা (না দিলে পিং আসবে না)
sysctl -w net.ipv4.conf.all.rp_filter=2
sysctl -w net.ipv4.conf.default.rp_filter=2
sysctl -w net.ipv4.conf.wg0.rp_filter=2

# TCP MSS Fix (না দিলে বড় সাইট লোড হবে না / Timeout আসবে)
iptables -t mangle -A POSTROUTING -p tcp --tcp-flags SYN,RST SYN -o wg0 -j TCPMSS --clamp-mss-to-pmtu 2>/dev/null || true
```

### ৫.৩ — V2bX-BD সার্ভিস তৈরি করা

**পূর্বশর্ত:** xBoard প্যানেলে নতুন VLESS Node তৈরি করুন (অরিজিনাল Node-এর চেয়ে আলাদা পোর্টে)। Node ID ধরুন **7**।

এরপর VPS টার্মিনালে নিচের পুরো ব্লকটি একবারে পেস্ট করুন:
```bash
# ধাপ ১: অরিজিনাল V2bX ফোল্ডারের কপি তৈরি (অরিজিনালটি অক্ষত থাকবে)
cp -r /etc/V2bX /etc/V2bX-BD

# ধাপ ২: নতুন কনফিগারে Node ID এবং ফাইলের পাথ আপডেট করা
sed -i 's/"NodeID": [0-9]*/"NodeID": 7/g' /etc/V2bX-BD/config.json
sed -i 's|/etc/V2bX/|/etc/V2bX-BD/|g' /etc/V2bX-BD/config.json

# ধাপ ৩: Custom Outbound তৈরি
# mark=255 → Linux Kernel এই ট্রাফিককে wg0 দিয়ে পাঠাবে
# UseIPv4 → IPv6 লিক হবে না, সবসময় IPv4 ব্যবহার করবে
cat <<'EOF' > /etc/V2bX-BD/custom_outbound.json
[
  {
    "protocol": "freedom",
    "tag": "bd-out",
    "settings": {
      "domainStrategy": "UseIPv4"
    },
    "streamSettings": {
      "sockopt": {
        "mark": 255
      }
    }
  }
]
EOF

# ধাপ ৪: Custom Route তৈরি
# সব TCP/UDP ট্রাফিক "bd-out" ট্যাগের Outbound দিয়ে বের হবে
cat <<'EOF' > /etc/V2bX-BD/custom_route.json
{
  "domainStrategy": "AsIs",
  "rules": [
    {
      "type": "field",
      "network": "tcp,udp",
      "outboundTag": "bd-out"
    }
  ]
}
EOF

# ধাপ ৫: কনফিগ ফাইলে রুট ফাইলের পাথ লিংক করা
sed -i -E 's|"RouteConfigPath":.*|"RouteConfigPath": "/etc/V2bX-BD/custom_route.json"|g' /etc/V2bX-BD/config.json

# ধাপ ৬: নতুন Systemd সার্ভিস ফাইল তৈরি
# (ExecStart-এর পাথ: systemctl cat V2bX দিয়ে দেখে নিন)
cat <<'EOF' > /etc/systemd/system/V2bX-BD.service
[Unit]
Description=V2bX BD Residential Service
After=network.target nss-lookup.target
Wants=network.target

[Service]
User=root
Group=root
Type=simple
LimitAS=infinity
LimitRSS=infinity
LimitCORE=infinity
LimitNOFILE=999999
WorkingDirectory=/usr/local/V2bX/
ExecStart=/usr/local/V2bX/V2bX server -c /etc/V2bX-BD/config.json
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# ধাপ ৭: সার্ভিস চালু করা
systemctl daemon-reload
systemctl enable V2bX-BD
systemctl restart V2bX-BD
systemctl status V2bX-BD --no-pager
```

স্ট্যাটাসে **`active (running)`** দেখালে সেটআপ সম্পন্ন।

---

## ✅ পর্ব ৬: চূড়ান্ত যাচাই

**১. রাউটার পিং চেক:**
```bash
ping -c 4 10.0.0.2
# ফলাফল: 0% packet loss → সফল
```

**২. BD IP দিয়ে ইন্টারনেট চেক:**
```bash
# Mark 255 দিয়ে গুগল পিং (রাউটারের মাধ্যমে যাবে)
ping -m 255 -c 4 8.8.8.8
# ফলাফল: 64 bytes from 8.8.8.8 → সফল
```

**৩. ক্লায়েন্ট চেক:**
- Shadowrocket বা v2rayNG-এ Node 7 কানেক্ট করুন
- [ipinfo.io](https://ipinfo.io) বা [whatismyip.com](https://whatismyip.com) এ চেক করুন
- **"org"** বা **"ISP"** সেকশনে বাংলাদেশের ব্রডব্যান্ড প্রোভাইডারের নাম দেখাবে ✅

---

## 🔧 সাধারণ সমস্যা ও সমাধান

| সমস্যা | কারণ | সমাধান |
|---|---|---|
| VPS থেকে `10.0.0.2` পিং হচ্ছে না | MikroTik-এর Public Key ভুল | `wg set wg0 peer [REAL_KEY] allowed-ips 0.0.0.0/0` |
| `ping -m 255 8.8.8.8` কাজ করছে না | `rp_filter` ব্লক করছে | `sysctl -w net.ipv4.conf.all.rp_filter=2` |
| ক্লায়েন্টে Timeout আসছে | TCP MSS সমস্যা | iptables TCPMSS clamp কমান্ড দিন |
| VPS-এর USA IPv6 দেখাচ্ছে | IPv6 Leak হচ্ছে | `ip -6 route add unreachable default table 200` |
| V2bX-BD সার্ভিস স্টার্ট হচ্ছে না | পোর্ট কনফ্লিক্ট বা JSON এরর | `journalctl -u V2bX-BD -n 30 --no-pager` দিয়ে লগ দেখুন |
