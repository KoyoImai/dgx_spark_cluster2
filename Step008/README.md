# Step008：

## 設定のリセット
全ての計算nodeで以下のコマンドを実行してください．
```
sudo rm -f /etc/netplan/40-cx7.yaml
sudo rm -f /etc/netplan/41-cx7-sw.yaml
sudo netplan apply
```
```
sudo sed -i '/qsfp/d' /etc/hosts
sudo sed -i '/node15-sw\|node16-sw\|node17-sw\|node18-sw/d' /etc/hosts
```


## 
**[参考](https://build.nvidia.com/spark/multi-sparks-through-switch/multi-sparks)**
node15
```
# Create the netplan configuration file
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f1np1:
      addresses:
        - 192.168.100.10/24
      dhcp4: no
    enP2p1s0f1np1:
      addresses:
        - 192.168.100.11/24
      dhcp4: no
EOF

# Set appropriate permissions
sudo chmod 600 /etc/netplan/40-cx7.yaml

# Apply the configuration
sudo netplan apply
```

node16
```
# Create the netplan configuration file
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f1np1:
      addresses:
        - 192.168.100.12/24
      dhcp4: no
    enP2p1s0f1np1:
      addresses:
        - 192.168.100.13/24
      dhcp4: no
EOF

# Set appropriate permissions
sudo chmod 600 /etc/netplan/40-cx7.yaml

# Apply the configuration
sudo netplan apply
```

node17
```
# Create the netplan configuration file
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f1np1:
      addresses:
        - 192.168.100.14/24
      dhcp4: no
    enP2p1s0f1np1:
      addresses:
        - 192.168.100.15/24
      dhcp4: no
EOF

# Set appropriate permissions
sudo chmod 600 /etc/netplan/40-cx7.yaml

# Apply the configuration
sudo netplan apply
```

node18
```
# Create the netplan configuration file
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f1np1:
      addresses:
        - 192.168.100.16/24
      dhcp4: no
    enP2p1s0f1np1:
      addresses:
        - 192.168.100.17/24
      dhcp4: no
EOF

# Set appropriate permissions
sudo chmod 600 /etc/netplan/40-cx7.yaml

# Apply the configuration
sudo netplan apply

```







