# WireGuard VPN Setup Guide (VPS & MikroTik - Complete)

ভবিষ্যতে নতুন কোনো রাউটার বা সার্ভার সেটআপ করার জন্য এই গাইডটি রেফারেন্স হিসেবে ব্যবহার করতে পারবেন। এই গাইডের মাধ্যমে একটি VPS-কে সার্ভার এবং MikroTik-কে ক্লায়েন্ট হিসেবে যুক্ত করে লোকাল নেটওয়ার্ক (10.0.0.x) তৈরি করা হয়েছে। পরবর্তীতে এই ടানেলের মাধ্যমে একটি xBoard (V2bX) নোডকে রেসিডেন্সিয়াল ഐপি (Residential BD IP) দিয়ে চালানোর অ্যাডভান্সড সেটআপও যোগ করা হয়েছে।

## ১. আর্কিটেকচার (Architecture)
*   **VPS (Server):** আইপি `10.0.0.1/24`, পোর্ট `51820`
*   **MikroTik (Client):** আইপি `10.0.0.2/24`
*   উভয় ডিভাইসকে একে অপরের **Public Key** জানতে হবে। 

---

## ২. VPS (Server) সেটআপ (Debian/Ubuntu)

VPS-এ টার্মিনাল বা SSH দিয়ে লগইন করে নিচের কমান্ডগুলো রান করতে হবে। এটি স্বয়ংক্রিয়ভাবে WireGuard ইনস্টল করে কনফিগারেশন তৈরি করবে:

```bash
# ১. WireGuard ইনস্টল করা
apt update && apt install wireguard -y

# ২. কনফিগারেশনের ফোল্ডার তৈরি ও পারমিশন সেট
mkdir -p /etc/wireguard
chmod 700 /etc/wireguard

# ৩. কনফিগারেশন ফাইল তৈরি করা (/etc/wireguard/wg0.conf)
cat <<EOF > /etc/wireguard/wg0.conf
[Interface]
# সার্ভারের প্রাইভেট কী (বাস্তব উদাহরণ)
PrivateKey = sTChSa8KY4/bfWtq7Bu3xQO373Q0dZuZKRvnP4AiYss=
Address = 10.0.0.1/24
ListenPort = 51820
Table = off  # মেইন রাউটিং নষ্ট না হওয়ার জন্য

[Peer]
# রাউটারের পাবলিক কী (যা রাউটার অটো-জেনারেট করেছে) (বাস্তব উদাহরণ)
PublicKey = RtL4ZV+iYM+VP2M5Fq9p/OYNV41g0qu4dp8pH61Wyhw=
# 0.0.0.0/0 দিয়ে ইন্টারনেটের এবং ট্রাফিকের পারমিশন দেওয়া
AllowedIPs = 0.0.0.0/0
EOF

# ৪. সার্ভারের সিকিউরিটি পোর্ট ওপেন করা
ufw allow 51820/udp 2>/dev/null || true

# ৫. WireGuard চালু করা এবং রিবুটে অটো-স্টার্ট করা
wg-quick down wg0 2>/dev/null || true
wg-quick up wg0
systemctl enable wg-quick@wg0
```

---

## ৩. MikroTik (Client) সেটআপ (Winbox)

আপনার রাউটারে WireGuard সেটআপ করার জন্য আপনি ২টি পদ্ধতি (Terminal অথবা GUI) ব্যবহার করতে পারেন।

### পদ্ধতি ১: GUI বা মাউস ক্লিক দিয়ে সেটআপ (Winbox)
**ধাপ এ: ইন্টারফেস তৈরি করা**
১. Winbox-এর বাম দিকের মেনু থেকে **WireGuard**-এ ক্লিক করুন।
২. **WireGuard (প্রথম ট্যাব)**-এ থাকা অবস্থায় **প্লাস (+)** আইকনে ক্লিক করুন।
৩. বক্সে নিচের তথ্যগুলো বসান:
   - **Name:** wireguard-mikhmon
   - **Listen Port:** 51820
   - **Private Key:** hJ5ljMd1ETlvULZCaZdJK0e/uzlhIZJrIQ6xW/iwoCg=
   *(এটি দেওয়ার পর Apply ক্লিক করলেই Winbox অটোমেটিক আপনার রাউটারের একটি Public Key তৈরি করে নেবে, যা পরে VPS-এ দিতে হবে)।*
৪. **Apply** এবং **OK** ক্লিক করে সেভ করুন।

**ধাপ বি: সার্ভার (VPS) কে Peer হিসেবে অ্যাড করা**
১. একই উইন্ডোর ওপরের **Peers** ট্যাবে ক্লিক করুন > **প্লাস (+)** আইকনে ক্লিক করুন।
৩. বক্সে নিচের তথ্যগুলো বসান:
   - **Interface:** wireguard-mikhmon
   - **Endpoint:** 31.97.215.132 *(এটি VPS-এর আসল আইপি)*
   - **Endpoint Port:** 51820
   - **Public Key:** 6nfKUt+ivFb/CxCR4xdU6NROVFpBlAPXUo2hBqLyuiA= *(এটি VPS-এর আসল Public Key)*
   - **Allowed Address:** 10.0.0.1/32
   - **Persistent Keepalive:** 25s

**ধাপ সি: আইপি অ্যাড্রেস সেট করা**
১. Winbox-এর বাম দিকের মেনু থেকে **IP > Addresses**-এ যান > **প্লাস (+)** ক্লিক করুন।
   - **Address:** 10.0.0.2/24
   - **Interface:** wireguard-mikhmon

---

## ৪. Mikhmon কানেকশন
সবশেষে, আপনার ওয়েবে হোস্ট করা Mikhmon ড্যাশবোর্ডে গিয়ে **Session Settings**-এর **IP MikroTik** বক্সে রাউটারের লোকাল আইপি (`10.0.0.2`) দিয়ে Connect করবেন।

---
---

## ৫. xBoard + V2bX দিয়ে Residential BD IP তৈরি (চীন/অন্য দেশের জন্য)
এই সেটআপের মাধ্যমে আপনার ভিপিএস-এর একটি নির্দিষ্ট ভিলেস (VLESS) নোডের ট্রাফিক আপনার বাসার রাউটার দিয়ে ইন্টারনেটে বের হবে, এবং ক্লায়েন্ট আপনার বাসার আইপি (Residential IP) পাবে।

### ৫.১ MikroTik NAT সেটআপ
রাউটারের Winbox > New Terminal ওপেন করে নিচের কোডটি পেস্ট করুন:
```ros
/ip firewall nat add chain=srcnat action=masquerade src-address=10.0.0.0/24 comment="NAT for V2bX VPS"
```

### ৫.২ VPS Linux Policy Routing
ভিপিএস-এ লগইন করে নিচের কোডগুলো রান করুন:
```bash
# ১. IPv4 রাউটিং (Mark 255 ট্রাফিককে রাউটারের দিকে ফরোয়ার্ড করা)
ip rule add fwmark 255 table 200
ip route add default dev wg0 table 200

# ২. IPv6 Blackhole (যাতে IPv6 লিকেজ না হয়)
ip -6 rule add fwmark 255 table 200
ip -6 route add unreachable default table 200

# ৩. Reverse Path Filtering রিল্যাক্স করা (টাইমআউট ফিক্স)
sysctl -w net.ipv4.conf.all.rp_filter=2
sysctl -w net.ipv4.conf.default.rp_filter=2
sysctl -w net.ipv4.conf.wg0.rp_filter=2

# ৪. MTU/MSS Fix (বড় ওয়েবসাইটে টাইমআউট ঠেকানো)
iptables -t mangle -A POSTROUTING -p tcp --tcp-flags SYN,RST SYN -o wg0 -j TCPMSS --clamp-mss-to-pmtu 2>/dev/null || true
```

### ৫.৩ V2bX-BD (Secondary Instance) তৈরি করা
**পূর্বশর্ত:** xBoard-এ নতুন একটি Node তৈরি করবেন (যেমন Node ID: 7)। 

এরপর টার্মিনালে নিচের পুরো ব্লকটি রান করুন:
```bash
# ১. অরিজিনাল V2bX-এর কপি তৈরি করা
cp -r /etc/V2bX /etc/V2bX-BD

# ২. কনফিগারেশনে Node ID 7 করা এবং পাথ আপডেট করা
sed -i 's/"NodeID": [0-9]*/"NodeID": 7/g' /etc/V2bX-BD/config.json
sed -i 's|/etc/V2bX/|/etc/V2bX-BD/|g' /etc/V2bX-BD/config.json

# ৩. Custom Outbound তৈরি (Mark 255 এবং IPv4 Strict)
cat <<EOF > /etc/V2bX-BD/custom_outbound.json
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

# ৪. Custom Route তৈরি
cat <<EOF > /etc/V2bX-BD/custom_route.json
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

# ৫. কনফিগার ফাইলে রুট লিংক আপডেট করা
sed -i -E 's|"RouteConfigPath":.*|"RouteConfigPath": "/etc/V2bX-BD/custom_route.json"|g' /etc/V2bX-BD/config.json

# ৬. নতুন সার্ভিস তৈরি করা (অরিজিনাল executable path-এর সাথে মিল রেখে)
cat <<EOF > /etc/systemd/system/V2bX-BD.service
[Unit]
Description=V2bX BD Service
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

# ৭. সার্ভিস চালু করা
systemctl daemon-reload
systemctl enable V2bX-BD
systemctl restart V2bX-BD
systemctl status V2bX-BD --no-pager
```

এই গাইড ফলো করলে নিখুঁতভাবে **Mikhmon এবং xBoard**—উভয় কাজই কোনো ঝামেলা ছাড়া করা যাবে।
