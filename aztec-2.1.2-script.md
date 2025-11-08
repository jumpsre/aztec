# aztec 2.1.2 升级步骤

## 0、准备工作

**找到你的旧的序列私钥地址:**  

**sepolia rpc 地址：** 

**Beacon API地址： ** 

## 1、安装依赖环境

```bash
bash -i <(curl -s https://install.aztec.network) && curl -L https://foundry.paradigm.xyz | bash && source /root/.bashrc && foundryup && aztec-up 2.1.2
aztec --version
cast --version
```

## 2、执行节点注册脚本

```bash
#!/bin/bash
set -e  # 出现错误时立即退出

clear
echo "读取旧的私钥地址."
read -sp "   请输入旧的序列其私钥 : " OLD_PRIVATE_KEY && echo
read -p "   请输入sepolia rpc地址: " ETH_RPC
echo "开始启动.." && echo " "

# 校验输入
if [ -z "$OLD_PRIVATE_KEY" ]; then
  echo "❌ 错误：旧的私钥不能为空。"
  exit 1
fi
if [ -z "$ETH_RPC" ]; then
  echo "❌ 错误：RPC 地址不能为空。"
  exit 1
fi

echo ":请准备好记录下你的以太坊私钥、BLS私钥以及以太坊地址."

# 默认keystone生成路径
KEYSTORE_FILE=~/.aztec/keystore/key1.json

if [ -f "$KEYSTORE_FILE" ]; then
  echo "检测到密钥文件已存在: $KEYSTORE_FILE"
  echo "跳过新密钥生成，继续使用现有密钥。"
else
  read -p "   按 [Enter] 键以生成新的密钥..." 
  aztec validator-keys new --fee-recipient 0x0000000000000000000000000000000000000000000000000000000000000000 && echo " " 
fi

# 新序列器私钥
NEW_ETH_PRIVATE_KEY=$(jq -r '.validators[0].attester.eth' $KEYSTORE_FILE)
# 新BLS私钥
NEW_BLS_PRIVATE_KEY=$(jq -r '.validators[0].attester.bls' $KEYSTORE_FILE)
# 新序列器地址
NEW_PUBLIC_ADDRESS=$(cast wallet address $NEW_ETH_PRIVATE_KEY)

echo "很好！你的新密钥如下。请妥善保存这些信息！"
echo "   - 新的以太坊私钥: $NEW_ETH_PRIVATE_KEY"
echo "   - 新的BLS私钥:  $NEW_BLS_PRIVATE_KEY"
echo "   - 新的公钥地址:   $NEW_PUBLIC_ADDRESS"
echo " "

echo "你需要向该新地址转入 0.2 到 0.5 Sepolia ETH："
echo "   $NEW_PUBLIC_ADDRESS"
read -p "   转账确认完成后，按 [Enter] 键继续.." && echo " "

TOKEN_ADDRESS="0x139d2a7a0881e16332d7D1F8DB383A4507E1Ea7A"
SPENDER_ADDRESS="0xebd99ff0ff6677205509ae73f93d0ca52ac85d67"
APPROVE_AMOUNT="200000ether"
ETHERSCAN_PREFIX="https://sepolia.etherscan.io/tx/"

echo "🔍 检查当前授权额度..."
ALLOWANCE=$(cast call "$TOKEN_ADDRESS" "allowance(address,address)(uint256)" "$(cast wallet address "$OLD_PRIVATE_KEY")" "$SPENDER_ADDRESS" --rpc-url "$ETH_RPC" || echo 0)


if [[ "$ALLOWANCE" != "0" && "$ALLOWANCE" != "0x0" ]]; then
  echo "✅ 检测到已有授权额度（跳过 approve）。"
else
  echo "执行 STAKE 代币授权..."
  TX_HASH=$(cast send "$TOKEN_ADDRESS" "approve(address,uint256)" "$SPENDER_ADDRESS" "$APPROVE_AMOUNT" \
    --private-key "$OLD_PRIVATE_KEY" --rpc-url "$ETH_RPC" | grep -oE '0x[a-fA-F0-9]{64}' | head -n 1 || true)

  if [ -z "$TX_HASH" ]; then
    echo "❌ 授权交易失败，请检查 RPC 或账户余额。"
    exit 1
  fi

  echo "✅ 授权交易已发送！"
  echo "   📜 交易哈希: $TX_HASH"
  echo "   🔗 Etherscan: ${ETHERSCAN_PREFIX}${TX_HASH}"
  echo "请等待交易确认后继续..." && echo " "
fi


# === 检查是否已注册验证者 ===
echo "🔎 检查该地址是否已经注册为验证者..."
IS_REGISTERED=$(cast call "$SPENDER_ADDRESS" "isValidator(address)(bool)" "$NEW_PUBLIC_ADDRESS" --rpc-url "$ETH_RPC" 2>/dev/null || echo "false")

if [ "$IS_REGISTERED" == "true" ]; then
  echo "✅ 检测到该地址已经是验证者，跳过注册步骤。"
else
  echo "🚀 正在加入测试网..."
  REG_TX=$(aztec add-l1-validator \
    --l1-rpc-urls "$ETH_RPC" \
    --network testnet \
    --private-key "$OLD_PRIVATE_KEY" \
    --attester "$NEW_PUBLIC_ADDRESS" \
    --withdrawer "$NEW_PUBLIC_ADDRESS" \
    --bls-secret-key "$NEW_BLS_PRIVATE_KEY" \
    --rollup "$SPENDER_ADDRESS" 2>&1 | tee /tmp/aztec_join.log | grep -oE '0x[a-fA-F0-9]{64}' | head -n 1 || true)

  if [ -n "$REG_TX" ]; then
    echo "✅ 已发送注册交易："
    echo "   📜 交易哈希: $REG_TX"
    echo "   🔗 Etherscan: ${ETHERSCAN_PREFIX}${REG_TX}"
  else
    echo "⚠️ 未检测到交易哈希，请手动确认是否注册成功。"
  fi
fi

echo "🎉 全部完成！你已成功加入（或已在）新的测试网."
echo "请使用你的新私钥和新地址重新运行节点."

echo "✅ 全部完成！你已成功加入新的测试网，现在请使用你的新私钥和新地址重新运行节点."
```

切记一定要保存新的私钥！！！切记一定要保存新的私钥！！！切记一定要保存新的私钥！！！

## 3、运行节点

删除历史数据

```bash
rm -rf /root/aztec/testnet/date/*
```

```bash
# 创建工作目录
mkdir -p /root/aztec-sequencer/keys /root/aztec-sequencer/data
# 进入目录
cd /root/aztec-sequencer
# 复制key1文件

cp ~/.aztec/keystore/key1.json /root/aztec-sequencer/keys/keystore.json

ETHEREUM_HOSTS="http://195.88.87.198:8545"
L1_CONSENSUS_HOST_URLS="http://195.88.87.198:3500"

# 生成.evn 文件
cat > /root/aztec-sequencer/.env <<EOF
DATA_DIRECTORY=./data
KEY_STORE_DIRECTORY=./keys
LOG_LEVEL=info
ETHEREUM_HOSTS=${ETHEREUM_HOSTS}
L1_CONSENSUS_HOST_URLS=${L1_CONSENSUS_HOST_URLS}
P2P_IP=$(curl -s ipv4.icanhazip.com)
P2P_PORT=40400
AZTEC_PORT=8080
AZTEC_ADMIN_PORT=8880
EOF
```

创建docker-compose地址

```bash
cat > /root/aztec-sequencer/docker-compose.yml <<'EOF'
services:
  aztec-sequencer:
    image: "aztecprotocol/aztec:2.1.2"
    container_name: "aztec-sequencer"
    ports:
      - ${AZTEC_PORT}:${AZTEC_PORT}
      - ${AZTEC_ADMIN_PORT}:${AZTEC_ADMIN_PORT}
      - ${P2P_PORT}:${P2P_PORT}
      - ${P2P_PORT}:${P2P_PORT}/udp
    volumes:
      - ${DATA_DIRECTORY}:/var/lib/data
      - ${KEY_STORE_DIRECTORY}:/var/lib/keystore
    environment:
      KEY_STORE_DIRECTORY: /var/lib/keystore
      DATA_DIRECTORY: /var/lib/data
      LOG_LEVEL: ${LOG_LEVEL}
      ETHEREUM_HOSTS: ${ETHEREUM_HOSTS}
      L1_CONSENSUS_HOST_URLS: ${L1_CONSENSUS_HOST_URLS}
      P2P_IP: ${P2P_IP}
      P2P_PORT: ${P2P_PORT}
      AZTEC_PORT: ${AZTEC_PORT}
      AZTEC_ADMIN_PORT: ${AZTEC_ADMIN_PORT}
    entrypoint: >-
      node
      --no-warnings
      /usr/src/yarn-project/aztec/dest/bin/index.js
      start
      --node
      --archiver
      --sequencer
      --network testnet
    networks:
      - aztec
    restart: always

networks:
  aztec:
    name: aztec
EOF
```

启动节点

```bash
docker compose up -d
```

查看日志

```bash
docker compose logs -f --tail 10
```

查看同步状态

```bash
curl -s -X POST -H 'Content-Type: application/json' \
-d '{"jsonrpc":"2.0","method":"node_getL2Tips","params":[],"id":67}' \
http://localhost:8080 | jq -r ".result.proven.number"
```

查看节点状态

```bash
curl http://localhost:8080/status
```

检查新节点是否注册

https://dashtec.xyz/queue?search={换成自己新的地址}
