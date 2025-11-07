# Aztec 2.1.2 最新版测试网

[toc]

## 1、安装软件包

### 1.1 安装 aztec 2.1.2

```bash
bash -i <(curl -s https://install.aztec.network)
```

### 1.2 安装foundry

```bash
curl -L https://foundry.paradigm.xyz | bash
source /root/.bashrc
foundryup
```

### 1.3 查看版本：

```bash
aztec --version
cast --version
```

> 确保 aztec 是 2.1.2

## 2、创建keystore

**方法1: 没有运行序列器的新用户**


```bash
aztec validator-keys new \
  --fee-recipient 0x0000000000000000000000000000000000000000000000000000000000000000
```

**方法二：运行序列器的用户**

```bash
aztec validator-keys new \
  --fee-recipient 0x0000000000000000000000000000000000000000000000000000000000000000 \
  --mnemonic "your twelve word mnemonic phrase here"
```

> 参数说明：
>
> * --fee-recipient：接收L2奖励的地址，如果没有建议用默认 0x0000000000000000000000000000000000000000000000000000000000000000
> * --mnemonic: 这个地方很关键，建议用旧序列器的助记词，我用的是旧地址的助记词


>说明：
>
>* 默认keystone存储在 ` ~/.aztec/keystore/key1.json` 一定要保存好该文件不要泄露！！！

## 3. 运行Docker Compose

### 3.1 创建目录

```bash
mkdir -p aztec-sequencer/keys aztec-sequencer/data
cd aztec-sequencer
touch .env
```

### 3.2 复制keystone到指定目录

```bash
# Option 1: 如果已经创建了则复制
cp ~/.aztec/keystore/key1.json aztec-sequencer/keys/keystore.json

# Option 2: 如果没有创建的可以用这个命令创建
aztec validator-keys new \
  --fee-recipient [YOUR_AZTEC_FEE_RECIPIENT_ADDRESS] \
  --mnemonic "your twelve word mnemonic phrase here" \
  --data-dir aztec-sequencer/keys \
  --file keystore.json
```

> 说明：
>
> * 需要保证账号里面有足够的ETH，建议>0.3E，余额过低会导致区块发布失败

### 3.3 配置环境变量

创建 `aztec-sequencer/.evn` 文件

```bash
DATA_DIRECTORY=./data
KEY_STORE_DIRECTORY=./keys
LOG_LEVEL=info
ETHEREUM_HOSTS=[your L1 execution endpoint, or a comma separated list if you have multiple]
L1_CONSENSUS_HOST_URLS=[your L1 consensus endpoint, or a comma separated list if you have multiple]
P2P_IP=[your external IP address]
P2P_PORT=40400
AZTEC_PORT=8080
AZTEC_ADMIN_PORT=8880
```

**参数说明:**

* ETHEREUM_HOSTS: 换成ETH RPC地址
* L1_CONSENSUS_HOST_URLS： 换成 CONSENSUS 地址
* P2P_IP：换成你的公网IP

### 3.4 创建docker-compose文件

```bash
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
```

### 3.5 启动

```bash
docker compose up -d
```

## 4、验证节点

### 4.1 检测同步状态

```bash
curl -s -X POST -H 'Content-Type: application/json' \
-d '{"jsonrpc":"2.0","method":"node_getL2Tips","params":[],"id":67}' \
http://localhost:8080 | jq -r ".result.proven.number"
```

> 提示：
>
> 可以在 https://devnet.aztecscan.xyz/ 或 https://aztecexplorer.xyz/ 确定高度

### 4.2 检测节点状态

```bash
curl http://localhost:8080/status
```

### 4.3 查看日志

```bash
docker compose logs -f aztec-sequencer
```

## 5、注册序列器

序列器节点设置并运行后，必须将其注册到网络才能加入序列器集。

### 5.1 授权 Aztec Rollup地址

```bash
cast send 0x139d2a7a0881e16332d7D1F8DB383A4507E1Ea7A "approve(address,uint256)" 0xebd99ff0ff6677205509ae73f93d0ca52ac85d67 200000ether --private-key "$PRIVATE_KEY_OF_OLD_SEQUENCER" --rpc-url $ETH_RPC
```

> 提示：
>
> * `PRIVATE_KEY_OF_OLD_SEQUENCER` 换成你接收 `STAKE` 的地址，一般是老的序列器私钥

### 5.2 注册你的节点

```bash
aztec add-l1-validator \
  --l1-rpc-urls $ETH_RPC \
  --network testnet \
  --private-key $PRIVATE_KEY_OF_OLD_SEQUENCER \
  --attester $ETH_ATTESTER_ADDRESS \
  --withdrawer $ANY_ETH_ADDRESS \
  --bls-secret-key $BLS_ATTESTER_PRIV_KEY \
  --rollup 0xebd99ff0ff6677205509ae73f93d0ca52ac85d67
```

**参数说明：**

* --l1-rpc-urls: 以太坊RPC
* **`--private-key: 用于支付 gas 费用的以太坊账户私钥（这不是您的序列器密钥）千万不要填错`**
* --attester：您的序列器在密钥库中的认证地址
* --withdrawer: 保持与attester一致
* --bls-secret-key: bls地址的私钥
* --rollup: 默认即可

> 提示：
>
> 一定要注意 --private-key 这里是支付ETH的地址，官方文档提示不要跟序列器一个地址

交易成功后，你的序列器会被提交上链。
