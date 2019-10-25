# 建立网络

## 1. 生成certs和keys（cryptogen）

`cryptogen generate --config=crypto-config.yaml`

运行之后，在当前目录的cryto-config目录下生成相关工件(certs and keys),生成的目录见crypro-config-dir.md

核心目录解释
    - ca
    - msp
        - admincerts
        - cacerts
        - keystore
        - signcerts
        - tlscacerts
        - config.yml
    - tlsca
    - tls
    - users
    - peers
    - orders

Users
    - Admin@example.com
    - Admin@org1.example.com
    - User1@org1.example.com
    - Admin@org2.example.com
    - User2@org2.example.com

Nodes
    - orderer中的example.com组织
      - ca.example.com
      - orderer.example.com
      - ordererX.example.com
      - tlsca.example.com
    - peer中的org1.example.com组织
      - ca.org1.example.com
      - tlsca.org1.example.com
      - peer0.org1.example.com
      - peer1.org1.example.com
    - peer中的org2.example.com组织
      - ca.org2.example.com
      - tlsca.org2.example.com
      - peer0.org2.example.com
      - peer1.org2.example.com

### 替换私钥

OPTS="-i"
cp docker-compose-e2e-template.yaml docker-compose-e2e.yaml
CURRENT_DIR=$PWD
cd crypto-config/peerOrganizations/org1.example.com/ca/
PRIV_KEY=$(ls *_sk)
cd "$CURRENT_DIR"
sed $OPTS "s/CA1_PRIVATE_KEY/${PRIV_KEY}/g" docker-compose-e2e.yaml
cd crypto-config/peerOrganizations/org2.example.com/ca/
PRIV_KEY=$(ls *_sk)
cd "$CURRENT_DIR"
sed $OPTS "s/CA2_PRIVATE_KEY/${PRIV_KEY}/g" docker-compose-e2e.yaml

## 2. 生成genesis block和初始化transaction(configtxgen)

此处生成的genesis block为系统级别的genesis block
除了系统级别的genesis block之外还有channel级别的genesis block（通过创建channel的transaction生成）

```shell
export FABRIC_CFG_PATH=$PWD # configtx.yml所在的目录

# 生成genesis block
configtxgen -profile TwoOrgsOrdererGenesis -channelID byfn-sys-channel -outputBlock ./channel-artifacts/genesis.block
configtxgen -profile SampleMultiNodeEtcdRaft -channelID byfn-sys-channel -outputBlock ./channel-artifacts/genesis.block # Raft ordering service
configtxgen -profile SampleDevModeKafka -channelID byfn-sys-channel -outputBlock ./channel-artifacts/genesis.block # Kafka ordering service

此处的channelID是系统channel的名字

# 生成Channel Configuration Transaction
export CHANNEL_NAME=mychannel  && configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID $CHANNEL_NAME # 此命令也适用于Raft ordering service 和 Kafka ordering service

# 生成transaction定义anchor peer
export CHANNEL_NAME=mychannel
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org1MSP
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org2MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org2MSP
```

上面命令运行后，相应的工件会保存在channel-artifacts

## 3. 启动网络

```shell
docker-compose -f docker-compose-cli.yaml up -d
```

如果想看到日志的话可以省略-d参数

## 4. 创建 & 加入Channel

```shell
# 进入cli容器
docker exec -it cli bash

# 每次执行命令前先设置环境变量
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt

# 创建channel，会返回mychannel.block,保存在当前目录下
export CORE_PEER_LOCALMSPID="OrdererMSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/users/Admin@example.com/msp


export CHANNEL_NAME=mychannel
peer channel create -o orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/channel.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

# 将peer0.org1加入channel
peer channel join -b mychannel.block

# 将peer0.org2加入channel
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp CORE_PEER_ADDRESS=peer0.org2.example.com:9051 CORE_PEER_LOCALMSPID="Org2MSP" CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt peer channel join -b mychannel.block
```

## 5. 更新anchor peers

```shell
# Update the channel definition to define the anchor peer for Org1 as peer0.org1.example.com
peer channel update -o orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/Org1MSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

# update the channel definition to define the anchor peer for Org2 as peer0.org2.example.com
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp CORE_PEER_ADDRESS=peer0.org2.example.com:9051 CORE_PEER_LOCALMSPID="Org2MSP" CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt peer channel update -o orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/Org2MSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

## 6. 安装 & 实例化Chaincode

```shell
# 在peer0.org1上安装chaincode
peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/chaincode_example02/go/ # Golang
peer chaincode install -n mycc -v 1.0 -l node -p /opt/gopath/src/github.com/chaincode/chaincode_example02/node/ # Node.js
peer chaincode install -n mycc -v 1.0 -l java -p /opt/gopath/src/github.com/chaincode/chaincode_example02/java/ # Java

# 在peer0.org2上安装chaincode
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp CORE_PEER_ADDRESS=peer0.org2.example.com:9051 CORE_PEER_LOCALMSPID="Org2MSP" CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/chaincode_example02/go/

# 实例化chaincode（会启动一个新的docker来运行对应peer的chaincode，只需要由一个peer实例化一次，其他peer会在执行chaincode时进行新建docker的操作）
# a chaincode container is not started for a peer until an init or traditional transaction - read/write - is performed against that chaincode (e.g. query for the value of “a”)
peer chaincode instantiate -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C $CHANNEL_NAME -n mycc -v 1.0 -c '{"Args":["init","a", "100", "b","200"]}' -P "AND ('Org1MSP.peer','Org2MSP.peer')"

# 在peer1.org2上安装chaincode
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp CORE_PEER_ADDRESS=peer1.org2.example.com:10051 CORE_PEER_LOCALMSPID="Org2MSP" CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/tls/ca.crt peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/chaincode_example02/go/

# 将peer1.org2加入channel
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp CORE_PEER_ADDRESS=peer1.org2.example.com:10051 CORE_PEER_LOCALMSPID="Org2MSP" CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/tls/ca.crt peer channel join -b mychannel.block
```

## 7. 查询

```shell
peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
```

## 8. 调用

```shell
peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C $CHANNEL_NAME -n mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"Args":["invoke","a","b","10"]}'
```

## 9. 其他

常用命令

```shell
# 查看日志
docker logs -f cli

# 在chaincode容器上查看transaction日志
docker logs dev-peer0.org2.example.com-mycc-1.0

# 清理环境
docker rm -f $(docker ps --filter "name=example.com" --filter "name=cli" -aq)
docker rmi -f $(docker images --filter=reference='*example.com*' -q)
rm -rf channel-artifacts/*
rm -rf crypto-config/*
docker network prune # 可以在报错的时候使用（If you see an error stating that you still have “active endpoints”）

# 启动网络后执行脚本，构建channel
docker exec cli scripts/script.sh $CHANNEL_NAME $CLI_DELAY $LANGUAGE $CLI_TIMEOUT $VERBOSE $NO_CHAINCODE
```

### docker-compose-e2e.yaml 与 docker-compose-cli.yaml

- docker-compose-cli.yaml ：提供了一个cli容器来执行操作
- docker-compose-e2e.yaml ：用于通过node.js SDK执行 end-to-end tests，同时还包含了 fabric-ca servers(提供restful接口)
  - 使用的时候注意设置环境变量FABRIC_CA_SERVER_TLS_KEYFILE 

### 状态DB

- 默认为goleveldb
- 可以切换为CouchDB ： docker-compose -f docker-compose-cli.yaml -f docker-compose-couch.yaml up -d
  - 可以提供更多的query功能
  - 每个peer有独立的container，且可以通过映射容器端口，通过web应用查看db内容
  - 持久化 ： 通过挂载db容器的volume
