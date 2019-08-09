# 搭建fabcar网络

## 1. clean the keystore

```shell
# 进入fabcar目录
rm -rf ./hfc-key-store
```

## 启动网络

```shell
# 清理docker环境（在first-network目录内）
docker rm -f $(docker ps --filter "name=example.com" --filter "name=cli" --filter "name=ca" --filter "name=couchdb" -aq)
docker rmi -f $(docker images --filter=reference='*example.com*' -q)
rm -rf channel-artifacts/*
rm -rf crypto-config/*
docker network prune # 可以在报错的时候使用（If you see an error stating that you still have “active endpoints”）

# 生成工件
cryptogen generate --config=crypto-config.yaml
export FABRIC_CFG_PATH=$PWD
configtxgen -profile TwoOrgsOrdererGenesis -channelID byfn-sys-channel -outputBlock ./channel-artifacts/genesis.block
export CHANNEL_NAME=mychannel
configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID $CHANNEL_NAME
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org1MSP
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org2MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org2MSP

# 也可以直接执行生成工件
./byfn.sh generate -a -n -s couchdb

# 运行docker容器
docker-compose -f docker-compose-cli.yaml -f docker-compose-ca.yaml -f docker-compose-couch.yaml up -d

# 创建channel
docker exec cli scripts/script.sh mychannel 3 golang 10 false true

```

## 安装 & 初始化 smart contract

```shell
# 进入cli
docker exec -it cli bash

# 设置环境变量
export CONFIG_ROOT=/opt/gopath/src/github.com/hyperledger/fabric/peer
export ORG1_MSPCONFIGPATH=${CONFIG_ROOT}/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export ORG1_TLS_ROOTCERT_FILE=${CONFIG_ROOT}/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export ORG2_MSPCONFIGPATH=${CONFIG_ROOT}/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export ORG2_TLS_ROOTCERT_FILE=${CONFIG_ROOT}/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export ORDERER_TLS_ROOTCERT_FILE=${CONFIG_ROOT}/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export CC_SRC_LANGUAGE=javascript
export CC_RUNTIME_LANGUAGE=node
export CC_SRC_PATH=/opt/gopath/src/github.com/chaincode/fabcar/javascript

# Installing smart contract on peer0.org1.example.com
CORE_PEER_LOCALMSPID=Org1MSP CORE_PEER_ADDRESS=peer0.org1.example.com:7051 CORE_PEER_MSPCONFIGPATH=${ORG1_MSPCONFIGPATH} CORE_PEER_TLS_ROOTCERT_FILE=${ORG1_TLS_ROOTCERT_FILE} peer chaincode install -n fabcar -v 1.0 -p "$CC_SRC_PATH" -l "$CC_RUNTIME_LANGUAGE"

# Installing smart contract on peer1.org1.example.com
CORE_PEER_LOCALMSPID=Org1MSP CORE_PEER_ADDRESS=peer1.org1.example.com:8051 CORE_PEER_MSPCONFIGPATH=${ORG1_MSPCONFIGPATH} CORE_PEER_TLS_ROOTCERT_FILE=${ORG1_TLS_ROOTCERT_FILE} peer chaincode install -n fabcar -v 1.0 -p "$CC_SRC_PATH" -l "$CC_RUNTIME_LANGUAGE"

# Installing smart contract on peer0.org2.example.com
CORE_PEER_LOCALMSPID=Org2MSP CORE_PEER_ADDRESS=peer0.org2.example.com:9051 CORE_PEER_MSPCONFIGPATH=${ORG2_MSPCONFIGPATH} CORE_PEER_TLS_ROOTCERT_FILE=${ORG2_TLS_ROOTCERT_FILE} peer chaincode install -n fabcar -v 1.0 -p "$CC_SRC_PATH" -l "$CC_RUNTIME_LANGUAGE"

# Installing smart contract on peer1.org2.example.com
CORE_PEER_LOCALMSPID=Org2MSP CORE_PEER_ADDRESS=peer1.org2.example.com:10051 CORE_PEER_MSPCONFIGPATH=${ORG2_MSPCONFIGPATH} CORE_PEER_TLS_ROOTCERT_FILE=${ORG2_TLS_ROOTCERT_FILE} peer chaincode install -n fabcar -v 1.0 -p "$CC_SRC_PATH" -l "$CC_RUNTIME_LANGUAGE"

# Instantiating smart contract on mychannel
CORE_PEER_LOCALMSPID=Org1MSP CORE_PEER_MSPCONFIGPATH=${ORG1_MSPCONFIGPATH} peer chaincode instantiate -o orderer.example.com:7050 -C mychannel -n fabcar -l "$CC_RUNTIME_LANGUAGE" -v 1.0 -c '{"Args":[]}' -P "AND('Org1MSP.member','Org2MSP.member')" --tls --cafile ${ORDERER_TLS_ROOTCERT_FILE} --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles ${ORG1_TLS_ROOTCERT_FILE}

# 等待10s

# Submitting initLedger transaction to smart contract on mychannel
# The transaction is sent to all of the peers so that chaincode is built before receiving the following requests
CORE_PEER_LOCALMSPID=Org1MSP CORE_PEER_MSPCONFIGPATH=${ORG1_MSPCONFIGPATH} peer chaincode invoke -o orderer.example.com:7050 -C mychannel -n fabcar -c '{"function":"initLedger","Args":[]}' --waitForEvent --tls --cafile ${ORDERER_TLS_ROOTCERT_FILE} --peerAddresses peer0.org1.example.com:7051 --peerAddresses peer1.org1.example.com:8051 --peerAddresses peer0.org2.example.com:9051 --peerAddresses peer1.org2.example.com:10051 --tlsRootCertFiles ${ORG1_TLS_ROOTCERT_FILE} --tlsRootCertFiles ${ORG1_TLS_ROOTCERT_FILE} --tlsRootCertFiles ${ORG2_TLS_ROOTCERT_FILE} --tlsRootCertFiles ${ORG2_TLS_ROOTCERT_FILE}
```
