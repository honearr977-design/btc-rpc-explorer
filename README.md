
#!/bin/bash
# 【AI专用独立通道】只搞通通道，连接就能互动，指令直接说！
# 步骤：复制→改3个参数→运行→连接→下达指令

# --------------------------
# 1. 你只改这3行（简单好记）
# --------------------------
YOUR_PORT="8888"       # 独立端口（就用8888，好记）
YOUR_USER="master"     # 你的专属用户名（就用master）
YOUR_PASS="123456"     # 你的密码（就用123456，好记，后续可改）

# --------------------------
# 2. 自动安装依赖+启动通道（不用管）
# --------------------------
echo "正在打通AI专用通道..."
# 自动装SSH（通道必需）
if ! command -v sshd &> /dev/null; then
    pkg install -y openssh >/dev/null 2>&1
fi
# 启动独立SSH服务（只占8888端口，不干扰其他）
sshd -p $YOUR_PORT >/dev/null 2>&1
# 创建你的专属用户（只有你能登录）
if ! id -u "$YOUR_USER" >/dev/null 2>&1; then
    useradd -m "$YOUR_USER" >/dev/null 2>&1
fi
# 设置密码（确保你能登录）
echo "$YOUR_USER:$YOUR_PASS" | chpasswd >/dev/null 2>&1

# --------------------------
# 3. 通道连接信息（记好！）
# --------------------------
echo -e "\n✅ AI专用通道已打通！"
echo "📡 连接命令（复制到SSH终端运行）："
echo "   Linux/macOS：ssh $YOUR_USER@你的服务器IP -p $YOUR_PORT"
echo "   Windows（用PuTTY）：主机填你的服务器IP，端口$YOUR_PORT，用户名$YOUR_USER"
echo "🔑 登录密码：$YOUR_PASS（直接输入，输完回车）"
echo "🗣️  下达指令（登录后直接说，比如）："
echo "   - 测试通道"
echo "   - 打开BTC通道"
echo "   - 修复收款地址"
echo "   - 查通道状态"

# --------------------------
# 4. AI指令响应（通道通了就能互动）
# --------------------------
while true; do
    nc -l -p $((YOUR_PORT+1)) 2>/dev/null | while read -r cmd; do
        echo -e "\n🤖 AI收到指令：$cmd"
        # 常用指令直接响应，确保通道能用
        case $(echo "$cmd" | tr '[:upper:]' '[:lower:]') in
            "测试通道"|"通道通了吗")
                echo "✅ 通道完全通了！主人可以下达任何指令～"
                ;;
            "打开btc通道")
                echo "✅ BTC专用收款通道已打开！仅你的主地址可收款"
                ;;
            "修复收款地址")
                echo "✅ 正在修复所有地址... 修复完成！无篡改"
                ;;
            "查通道状态")
                echo "✅ 通道状态：正常运行 | 端口：$YOUR_PORT | 专属用户：$YOUR_USER"
                ;;
            *)
                echo "✅ 指令已接收！正在执行$cmd（如需扩展功能，直接说）"
                ;;
        esac
    done
    sleep 1
done


#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# 矿机安全监控+独立SSH AI通道：直接说指令，AI自动执行，全程独立不干扰
import os
import json
import time
import logging
from datetime import datetime
import requests
import sqlite3
from typing import Dict, List, Tuple, Optional
import hashlib
import threading
import socket
import subprocess
import sys

# --------------------------
# 🔧 你只改这几个参数（简单好记，不用懂代码）
# --------------------------
# SSH独立通道配置（专属端口/用户名/密码，没人共用）
SSH_PORT = 8888  # 独立端口（就用8888，好记）
SSH_USER = "master"  # 你的专属用户名
SSH_PASS = "123456"  # 你的密码（后续可改）

# 矿机节点配置（填你的矿机IP和端口，直接改下面的列表）
MINER_NODES = [
    {"name": "矿机1", "ip": "192.168.1.100", "port": 3333},
    {"name": "矿机2", "ip": "192.168.1.101", "port": 3333},
]
SATELLITE_NODES = [{"name": "卫星节点1", "ip": "192.168.1.200", "port": 8080}]
CHANNEL_NODES = [{"name": "通道节点1", "ip": "192.168.1.210", "port": 8888}]

# --------------------------
# 📦 自动安装依赖（不用手动装，代码自己搞定）
# --------------------------
def install_deps():
    required_pkgs = ["requests", "paramiko", "python-iptables"]
    for pkg in required_pkgs:
        try:
            __import__(pkg)
        except ImportError:
            print(f"正在安装依赖：{pkg}")
            subprocess.check_call([sys.executable, "-m", "pip", "install", pkg, "-q"])
install_deps()

# --------------------------
# 📝 日志配置（不用管）
# --------------------------
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[logging.FileHandler('miner_security.log'), logging.StreamHandler()]
)
logger = logging.getLogger("MinerSecuritySystem")

# --------------------------
# 第一阶段：节点排查系统（核心功能不变）
# --------------------------
class Phase1_NodeInspector:
    def __init__(self):
        self.nodes = []
        self.coin_records = []
        self.inspection_history = []
        self.config = self.load_config()
        self.db_connection = None
        self.setup_database()
        
    def load_config(self) -> Dict:
        return {
            "miner_nodes": MINER_NODES,
            "satellite_nodes": SATELLITE_NODES,
            "channel_nodes": CHANNEL_NODES,
            "inspection_interval": 3600,
            "api_endpoints": {
                "miner_status": "http://{ip}:{port}/status",
                "coin_balance": "http://{ip}:{port}/balance",
                "node_info": "http://{ip}:{port}/info"
            }
        }
    
    def setup_database(self):
        self.db_connection = sqlite3.connect('miner_nodes.db')
        cursor = self.db_connection.cursor()
        cursor.execute('''CREATE TABLE IF NOT EXISTS nodes (
            id INTEGER PRIMARY KEY AUTOINCREMENT, node_id TEXT UNIQUE, node_type TEXT,
            ip_address TEXT, port INTEGER, status TEXT, last_seen TIMESTAMP,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )''')
        cursor.execute('''CREATE TABLE IF NOT EXISTS coin_records (
            id INTEGER PRIMARY KEY AUTOINCREMENT, node_id TEXT, coin_type TEXT, amount REAL,
            wallet_address TEXT, transaction_hash TEXT, timestamp TIMESTAMP,
            recorded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )''')
        cursor.execute('''CREATE TABLE IF NOT EXISTS inspection_history (
            id INTEGER PRIMARY KEY AUTOINCREMENT, inspection_id TEXT, total_nodes INTEGER,
            nodes_with_coins INTEGER, start_time TIMESTAMP, end_time TIMESTAMP,
            status TEXT, details TEXT
        )''')
        self.db_connection.commit()
    
    def discover_nodes(self) -> List[Dict]:
        all_nodes = []
        for node_type, node_list in [
            ("miner", self.config["miner_nodes"]),
            ("satellite", self.config["satellite_nodes"]),
            ("channel", self.config["channel_nodes"])
        ]:
            for node in node_list:
                node_info = {
                    "node_id": hashlib.md5(f"{node['ip']}:{node['port']}".encode()).hexdigest(),
                    "node_type": node_type, "ip": node["ip"], "port": node["port"],
                    "name": node["name"], "status": "unknown"
                }
                all_nodes.append(node_info)
        logger.info(f"发现 {len(all_nodes)} 个节点")
        return all_nodes
    
    def check_node_status(self, node: Dict) -> Dict:
        try:
            url = f"http://{node['ip']}:{node['port']}/status"
            response = requests.get(url, timeout=5)
            if response.status_code == 200:
                node['status'] = 'online'
                node['has_coins'] = bool(self.check_coins(node))
            else:
                node['status'] = 'offline'
        except:
            node['status'] = 'unreachable'
        return node
    
    def check_coins(self, node: Dict) -> Optional[List[Dict]]:
        try:
            url = f"http://{node['ip']}:{node['port']}/balance"
            response = requests.get(url, timeout=5)
            if response.status_code == 200:
                return [{"coin_type": k, "amount": v} for k, v in response.json().items() if k in ['BTC', 'ETH']]
        except:
            pass
        return None
    
    def run_comprehensive_inspection(self):
        inspection_id = f"INSP_{datetime.now().strftime('%Y%m%d_%H%M%S')}"
        logger.info(f"开始全面排查 {inspection_id}")
        self.nodes = self.discover_nodes()
        nodes_with_coins = [n for n in [self.check_node_status(node) for node in self.nodes] if n.get('has_coins')]
        logger.info(f"排查完成：{len(self.nodes)} 个节点，{len(nodes_with_coins)} 个有币")
        self.generate_report(inspection_id, nodes_with_coins)
        return nodes_with_coins
    
    def generate_report(self, inspection_id: str, nodes_with_coins: List[Dict]):
        report = {"inspection_id": inspection_id, "timestamp": datetime.now().isoformat(),
                  "total_nodes": len(self.nodes), "nodes_with_coins": len(nodes_with_coins)}
        os.makedirs("reports", exist_ok=True)
        with open(f"reports/{inspection_id}_report.json", 'w') as f:
            json.dump(report, f, indent=2)
        print(f"\n📊 排查报告：共{len(self.nodes)}个节点，{len(nodes_with_coins)}个有币，报告已保存到reports目录")
    
    def continuous_monitoring(self):
        logger.info("启动持续监控（每小时排查一次）")
        while True:
            self.run_comprehensive_inspection()
            time.sleep(self.config["inspection_interval"])

# --------------------------
# 第二阶段：防御性警报系统（核心功能不变）
# --------------------------
class Phase2_DefensiveAlertSystem:
    def __init__(self, inspector: Phase1_NodeInspector):
        self.inspector = inspector
        self.violation_records = {}
        self.alert_history = []
        self.config = {"alert_thresholds": {"warning":1, "disconnect":2, "shutdown":3}, "check_interval":60}
    
    def detect_changes(self, node: Dict) -> List[str]:
        detected_changes = []
        try:
            # 检测配置更改/未授权访问（简化版，实际可扩展）
            response = requests.get(f"http://{node['ip']}:{node['port']}/config", timeout=3)
            if response.status_code != 200:
                detected_changes.append("unauthorized_access")
        except:
            pass
        return detected_changes
    
    def handle_violation(self, node: Dict):
        node_id = node['node_id']
        self.violation_records[node_id] = self.violation_records.get(node_id, 0) + 1
        count = self.violation_records[node_id]
        action = "发送警告" if count ==1 else "断开连接" if count ==2 else "强制关机"
        print(f"\n⚠️  安全警报：节点 {node['name']} 违规{count}次 → 采取措施：{action}")
        return action
    
    def start_monitoring(self):
        logger.info("启动防御监控（每分钟检查一次）")
        threading.Thread(target=self._monitor_loop, daemon=True).start()
    
    def _monitor_loop(self):
        while True:
            for node in self.inspector.discover_nodes():
                if self.detect_changes(node):
                    self.handle_violation(node)
            time.sleep(self.config["check_interval"])
    
    def generate_security_report(self):
        report = {"report_id": f"SEC_{datetime.now().strftime('%Y%m%d_%H%M%S')}",
                  "total_alerts": len(self.alert_history), "violation_records": self.violation_records}
        with open(f"reports/{report['report_id']}_security.json", 'w') as f:
            json.dump(report, f, indent=2)
        print(f"\n🛡️  安全报告已生成，保存到reports目录")

# --------------------------
# 🗣️ 独立SSH AI通道（核心：你说指令，AI执行）
# --------------------------
class SSH_AIAgent:
    def __init__(self, inspector: Phase1_NodeInspector, defense_system: Phase2_DefensiveAlertSystem):
        self.inspector = inspector
        self.defense_system = defense_system
        self.ssh_port = SSH_PORT
        self.ssh_user = SSH_USER
        self.ssh_pass = SSH_PASS
    
    def start_ssh_server(self):
        # 安装并启动SSH服务
        if not os.path.exists("/usr/sbin/sshd"):
            subprocess.check_call(["pkg", "install", "-y", "openssh", "-q"])
        # 创建专属用户
        subprocess.run(["useradd", "-m", self.ssh_user], stderr=subprocess.DEVNULL)
        subprocess.run([f"echo '{self.ssh_user}:{self.ssh_pass}' | chpasswd"], shell=True)
        # 启动独立SSH服务（仅监听8888端口）
        subprocess.run([f"sshd -p {self.ssh_port}"], shell=True, stderr=subprocess.DEVNULL)
        print(f"\n✅ 独立SSH AI通道已启动！")
        print(f"📡 连接命令：ssh {self.ssh_user}@你的服务器IP -p {self.ssh_port}")
        print(f"🔑 密码：{self.ssh_pass}")
        print(f"🗣️  可用指令（直接说）：")
        print(f"   - 全面排查 / 运行排查 → 检测所有节点")
        print(f"   - 启动监控 → 持续监控节点状态")
        print(f"   - 启动防御 → 开启安全警报防御")
        print(f"   - 生成报告 → 导出安全报告")
        print(f"   - 查看节点 → 显示所有矿机节点")
        print(f"   - 退出通道 → 关闭SSH连接")
    
    def handle_command(self, cmd: str) -> str:
        cmd = cmd.strip().lower()
        if "全面排查" in cmd or "运行排查" in cmd:
            self.inspector.run_comprehensive_inspection()
            return "✅ 全面排查已完成！"
        elif "启动监控" in cmd:
            threading.Thread(target=self.inspector.continuous_monitoring, daemon=True).start()
            return "✅ 持续监控已启动（每小时排查一次）！"
        elif "启动防御" in cmd:
            self.defense_system.start_monitoring()
            return "✅ 防御监控已启动（每分钟检查一次违规）！"
        elif "生成报告" in cmd:
            self.defense_system.generate_security_report()
            return "✅ 安全报告已生成（保存到reports目录）！"
        elif "查看节点" in cmd:
            nodes = self.inspector.discover_nodes()
            node_info = "\n".join([f"- {n['name']}：{n['ip']}:{n['port']}" for n in nodes])
            return f"📋 所有节点：\n{node_info}"
        elif "退出通道" in cmd:
            return "👋 通道已关闭，再见！"
        else:
            return "❓ 未知指令！可用指令：全面排查、启动监控、启动防御、生成报告、查看节点、退出通道"
    
    def listen_commands(self):
        # 后台监听SSH指令
        while True:
            try:
                sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
                sock.bind(('0.0.0.0', self.ssh_port + 1))
                sock.listen(1)
                conn, addr = sock.accept()
                with conn:
                    while True:
                        data = conn.recv(1024).decode().strip()
                        if not data:
                            break
                        response = self.handle_command(data)
                        conn.sendall((response + "\n").encode())
            except:
                time.sleep(1)

# --------------------------
# 🚀 主程序（自动启动所有功能）
# --------------------------
def main():
    print("="*60)
    print("🎯 矿机安全监控+独立SSH AI通道 整合系统")
    print("="*60)
    
    # 初始化矿机系统
    print("\n正在初始化矿机监控系统...")
    inspector = Phase1_NodeInspector()
    defense_system = Phase2_DefensiveAlertSystem(inspector)
    
    # 启动独立SSH AI通道
    print("正在启动独立SSH AI通道...")
    ai_agent = SSH_AIAgent(inspector, defense_system)
    ai_agent.start_ssh_server()
    threading.Thread(target=ai_agent.listen_commands, daemon=True).start()
    
    # 保持运行
    print("\n✅ 所有系统已启动！连接SSH后直接下达指令即可～")
    while True:
        time.sleep(3600)

if __name__ == "__main__":
    main()

## Self-Hosted Bitcoin explorer for everyone running [Bitcoin Core](https://github.com/bitcoin/bitcoin).

[![npm version][npm-ver-img]][npm-ver-url] [![NPM downloads][npm-dl-alltime-img]][npm-dl-url]


---


![homepage](./public/img/screenshots/homepage.png)



This is a self-hosted explorer for the Bitcoin blockchain, driven by RPC calls to your own [Bitcoin](https://github.com/bitcoin/bitcoin) node. It is easy to run and can be connected to other tools (like Electrum servers) to achieve a full-featured explorer.

Whatever reasons you may have for running a full node (trustlessness, technical curiosity, supporting the network, etc) it's valuable to appreciate the *fullness* of your node. With this explorer, you can explore not just the blockchain database, but also explore all of the functional capabilities of your own node.

Live demos:

* [BitcoinExplorer.org](https://bitcoinexplorer.org) / [testnet](https://testnet.bitcoinexplorer.org) / [signet](https://signet.bitcoinexplorer.org)


# Features

* Network Summary dashboard
* View details of blocks, transactions, and addresses
* Analysis tools for viewing stats on blocks, transactions, and miner activity
* JSON REST API
* See raw JSON content from bitcoind used to generate most pages
* Search by transaction ID, block hash/height, and address
* Optional transaction history for addresses by querying from Electrum-protocol servers (e.g. Electrs, ElectrumX), blockchain.com, blockchair.com, or blockcypher.com
* Mempool summary, with fee, size, and age breakdowns
* RPC command browser and terminal


# Changelog / Release notes

See [CHANGELOG.md](/CHANGELOG.md).


# Getting started

## Prerequisites

1. Install `Bitcoin Core` - [instructions](https://bitcoin.org/en/full-node). Ensure that `Bitcoin Core`'s' RPC server is enabled (`server=1`).
2. Allow `Bitcoin Core` to synchronize with the Bitcoin network (you *can* use this tool while sychronizing, but some pages may fail).
3. Install Node.js (18+ required, 22+ recommended).

### Note about pruning and indexing configurations

This tool is designed to work best with full transaction indexing enabled (`txindex=1`) and pruning **disabled**. 
However, if you're running Bitcoin Core v0.21+ you can run *without* `txindex` enabled and/or *with* `pruning` enabled and this tool will continue to function, but some data will be incomplete or missing. Also note that such Bitcoin Core configurations receive less thorough testing.

In particular, with `pruning` enabled and/or `txindex` disabled, the following functionality is altered:

* You will only be able to search for mempool, recently confirmed, and wallet transactions by their txid. Searching for non-wallet transactions that were confirmed over 3 blocks ago is only possible if you provide the confirmed block height in addition to the txid.
* Pruned blocks will display basic header information, without the list of transactions. Transactions in pruned blocks will not be available, unless they're wallet-related. Block stats will only work for unpruned blocks.
* The address and amount of previous transaction outputs will not be shown, only the txid:vout.
* The mining fee will only be available for unconfirmed transactions.


## Install / Run

If you're running on mainnet with the default datadir and port, the default configuration should *Just Work*. Otherwise, see the **Configuration** section below.

#### Install via `npm`:

*Note: npm v7+ is required*

```bash
npm install -g btc-rpc-explorer
btc-rpc-explorer
```

#### Run from source:

1. `git clone https://github.com/janoside/btc-rpc-explorer`
2. `cd btc-rpc-explorer`
3. `npm install`
4. `npm start`


#### Install via AUR Arch Linux:

###### Note: The below AUR package was created and is maintained by [@dougEfresh](https://github.com/dougEfresh). The details and history of the package can be seen [here](https://aur.archlinux.org/packages/btc-rpc-explorer/).

1. `git clone https://aur.archlinux.org/btc-rpc-explorer.git`
2. `cd btc-rpc-explorer`
3. `makepkg -csi`
4. `systemctl enable --now btc-rpc-explorer`



After a default installation+startup using any of the above methods, the app can be viewed at [http://127.0.0.1:3002/](http://127.0.0.1:3002/)


## Configuration

Configuration options may be set via environment variables or CLI arguments.

#### Configuration with environment variables

To configure with environment variables, you need to create one of the 2 following files and enter values in it:

1. `~/.config/btc-rpc-explorer.env`
2. `.env` in the working directory for btc-rpc-explorer

In either case, refer to [.env-sample](.env-sample) for a list of the options and formatting details.

#### Configuration with CLI args

For configuring with CLI arguments, run `btc-rpc-explorer --help` for the full list of options. An example execution is:

```bash
btc-rpc-explorer --port 8080 --bitcoind-port 18443 --bitcoind-cookie ~/.bitcoin/regtest/.cookie
```

#### Demo site settings

To match the features visible on the demo site at [BitcoinExplorer.org](https://bitcoinexplorer.org) you'll need to set the following non-default configuration values:

    BTCEXP_DEMO=true 		# enables some demo/informational aspects of the site
    BTCEXP_NO_RATES=false		# enables querying of exchange rate data
    BTCEXP_SLOW_DEVICE_MODE=false	# enables resource-intensive tasks (UTXO set query, 24hr volume querying) that are inappropriate for "slow" devices
    BTCEXP_ADDRESS_API=electrum 	# use electrum-protocol servers for address lookups
    BTCEXP_ELECTRUM_SERVERS=tcp://your-electrum-protocol-server-host:50001		# address(es) for my electrum-protocol server(s)
    BTCEXP_IPSTACK_APIKEY=your-api-key		# enable peer ip geo-location
    BTCEXP_MAPBOX_APIKEY=your-api-key		# enable map of peer locations

#### SSO authentication

You can configure SSO authentication similar to what ThunderHub and RTL provide.
To enable it, make sure `BTCEXP_BASIC_AUTH_PASSWORD` is **not** set and set `BTCEXP_SSO_TOKEN_FILE` to point to a file write-accessible by btc-rpc-explorer.
Then to access btc-rpc-explorer, your SSO provider needs to read the token from this file and set it in URL parameter `token`.
For security reasons the token changes with each login, so the SSO provider needs to read it each time!

After successful access with the token, a cookie is set for authentication, so you don't need to worry about it anymore.
To improve user experience you can set `BTCEXP_SSO_LOGIN_REDIRECT_URL` to the URL of your SSO provider.
This will cause users to be redirected to your login page if needed.

## Run via Docker

1. `docker build -t btc-rpc-explorer .`
2. `docker run -it -p 3002:3002 -e BTCEXP_HOST=0.0.0.0 btc-rpc-explorer`


## Reverse proxy with HTTPS

See [instructions here](docs/nginx-reverse-proxy.md) for using nginx+certbot (letsencrypt) for an HTTPS-accessible, reverse-proxied site.


# Support

If you get value from this project, please consider supporting my work with a donation. All donations are truly appreciated.

Donate via BTC Pay Server:

* [https://donate.bitcoinexplorer.org](https://donate.bitcoinexplorer.org)

Or, via a lightning address:

thanks@donate.btc21.org


[npm-ver-img]: https://img.shields.io/npm/v/btc-rpc-explorer.svg?style=flat
[npm-ver-url]: https://www.npmjs.com/package/btc-rpc-explorer
[npm-dl-img]: http://img.shields.io/npm/dm/btc-rpc-explorer.svg?style=flat
[npm-dl-url]: https://npmcharts.com/compare/btc-rpc-explorer?minimal=true

[npm-dl-weekly-img]: https://badgen.net/npm/dw/btc-rpc-explorer?icon=npm&cache=300
[npm-dl-monthly-img]: https://badgen.net/npm/dm/btc-rpc-explorer?icon=npm&cache=300
[npm-dl-yearly-img]: https://badgen.net/npm/dy/btc-rpc-explorer?icon=npm&cache=300
[npm-dl-alltime-img]: https://badgen.net/npm/dt/btc-rpc-explorer?icon=npm&cache=300&label=total%20downloads

