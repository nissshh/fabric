Commands
rm -rf crypto-config &&
rm -rf channel-artifacts &&
rm -rf config-exports &&
cryptogen generate --config=./crypto-config.yaml &&
mkdir channel-artifacts &&
configtxgen -profile OneOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.block &&
mkdir config-exports &&
configtxgen -inspectBlock ./channel-artifacts/genesis.block > ./config-exports/genesis.json &&
configtxgen -profile OneOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID foo &&
configtxgen -inspectChannelCreateTx ./channel-artifacts/channel.tx > ./config-exports/channel-tx.json &&
configtxgen -profile OneOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID foo -asOrg Org1MSP

Post generation of Org2.json
scp dev@10.34.15.246:/home/dev/work/fabric/org2/channel-artifacts/org2.json /home/dev/git/fabric/oneOrgChannel/crypto-config/.


Org 2 Configurations

rm -rf crypto-config &&
rm -rf channel-artifacts &&
rm -rf config-exports &&
cryptogen generate --config=./crypto-config.yaml


SSH to Host 2 
ssh dev@10.34.15.246:dev.123
cd /home/dev/work/fabric/org2 && 
sudo rm -rf /home/dev/work/fabric/org2/crypto-config/ && echo Done
sudo mkdir -p /home/dev/work/fabric/org2/crypto-config/ordererOrganizations && echo Done Creating Orderer Directory
Below steps are note required in case of cryptogen working on node 2.
sudo scp -r dev@10.34.14.1:/home/dev/git/fabric/oneOrgChannel/crypto-config/ordererOrganizations ~/work/fabric/org2/crypto-config 
sudo scp -r dev@10.34.14.1:/home/dev/git/fabric/org2/crypto-config/ ~/work/fabric/org2/
sudo scp -r dev@10.34.14.1:/home/dev/git/fabric/org2/configtx.yaml ~/work/fabric/org2/
sudo rm -rf channel-artifacts && 
sudo mkdir channel-artifacts && 
sudo chmod 777 channel-artifacts
configtxgen -printOrg Org2MSP > ./channel-artifacts/org2.json

Go in to CLI
export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem  && export CHANNEL_NAME=foo
export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/dev-ThinkCentre-A63/msp/tlscacerts/tlsca.example.com-cert.pem && export CHANNEL_NAME=foo


peer channel fetch config config_block.pb -o dev-thinkcentre-a63:7050 -c $CHANNEL_NAME --tls --cafile $ORDERER_CA
peer channel fetch config config_block.pb -o orderer.example.com:7050 -c $CHANNEL_NAME --tls --cafile $ORDERER_CA



SWARM
dev@dev-thinkcentre-a63:> docker swarm init
dev@dev-thinkcentre-a63:~/git/fabric/org2$ docker swarm join-token manager
To add a manager to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-2hbn5v2xcrwu5hmr909cgzgq1o5wmacs29v2r7ydg7kclorvzr-6ilmxk36js0onwx1s9j6yz732 10.34.14.1:2377

dev@dev-thinkcentre-a63:~/git/fabric/org2$ docker node ls
ID                            HOSTNAME              STATUS              AVAILABILITY        MANAGER STATUS
9l833gxh3cssraptqgej39z3n     SYNL901761            Ready               Active              
422exu8a5g7c02d0i44tdm0n8 *   dev-thinkcentre-a63   Ready               Active              Leader
docker network create --attachable --driver overlay my-net


Docker Commands for Each Node Service: 
Orderer (No TLS)
docker run --rm -it --network="my-net" \
--name orderer.example.com -p 7050:7050 \
-e ORDERER_GENERAL_LOGLEVEL=debug \
-e ORDERER_GENERAL_LISTENADDRESS=0.0.0.0 \
-e ORDERER_GENERAL_LISTENPORT=7050 \
-e ORDERER_GENERAL_GENESISMETHOD=file \
-e ORDERER_GENERAL_GENESISFILE=/var/hyperledger/orderer/orderer.genesis.block \
-e ORDERER_GENERAL_LOCALMSPID=OrdererMSP \
-e ORDERER_GENERAL_LOCALMSPDIR=/var/hyperledger/orderer/msp \
-e ORDERER_GENERAL_TLS_ENABLED=false \
-e CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=my-net \
-v $(pwd)/channel-artifacts/genesis.block:/var/hyperledger/orderer/orderer.genesis.block \
-v $(pwd)/crypto-config/ordererOrganizations/example.com/orderers/dev-ThinkCentre-A63/msp:/var/hyperledger/orderer/msp \
-w /opt/gopath/src/github.com/hyperledger/fabric hyperledger/fabric-orderer orderer

PEER 1 (Org1)
docker run --rm -it \
--link orderer.example.com:orderer.example.com --network="my-net" \
--name peer0.org1.example.com \
-p 8051:7051 \
-p 8053:7053 \
-e CORE_PEER_ADDRESSAUTODETECT=true \
-e CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock \
-e CORE_LOGGING_LEVEL=DEBUG \
-e CORE_PEER_NETWORKID=peer0.org1.example.com \
-e CORE_NEXT=true \
-e CORE_PEER_ENDORSER_ENABLED=true \
-e CORE_PEER_ID=peer0.org1.example.com \
-e CORE_PEER_PROFILE_ENABLED=true \
-e CORE_PEER_COMMITTER_LEDGER_ORDERER=orderer.example.com:7050 \
-e CORE_PEER_GOSSIP_IGNORESECURITY=true \
-e CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=my-net \
-e CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org1.example.com:7051 \
-e CORE_PEER_TLS_ENABLED=false \
-e CORE_PEER_GOSSIP_USELEADERELECTION=false \
-e CORE_PEER_GOSSIP_ORGLEADER=true \
-e CORE_PEER_LOCALMSPID=Org1MSP \
-v /var/run/:/host/var/run/ \
-v $(pwd)/crypto-config/peerOrganizations/org1.example.com/peers/dev-thinkcentre-a63/msp:/etc/hyperledger/fabric/msp \
-w /opt/gopath/src/github.com/hyperledger/fabric/peer \
hyperledger/fabric-peer \
peer node start

CLI 1(Org 1)
CLI 
docker run --rm -it --network="my-net" \
--name cli \
--link orderer.example.com:orderer.example.com \
--link peer0.org1.example.com:peer0.org1.example.com \
--link peer0.org2.example.com:peer0.org2.example.com \
-p 12051:7051 \
-p 12053:7053 \
-e GOPATH=/opt/gopath \
-e CORE_PEER_LOCALMSPID=Org1MSP \
-e CORE_PEER_TLS_ENABLED=false \
-e CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock \
-e CORE_LOGGING_LEVEL=DEBUG \
-e CORE_PEER_ID=cli \
-e CORE_PEER_ADDRESS=peer0.org1.example.com:7051 \
-e CORE_PEER_NETWORKID=cli \
-e CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp \
-e CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=my-net  \
-e ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem \
-v /var/run/:/host/var/run/ \
-v $(pwd)/chaincode/:/opt/gopath/src/github.com/hyperledger/fabric/examples/chaincode/go \
-v $(pwd)/crypto-config:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ \
-v $(pwd)/scripts:/opt/gopath/src/github.com/hyperledger/fabric/peer/scripts/ \
-v $(pwd)/channel-artifacts:/opt/gopath/src/github.com/hyperledger/fabric/peer/channel-artifacts \
-w /opt/gopath/src/github.com/hyperledger/fabric/peer \
hyperledger/fabric-tools \
/bin/bash

Later install below in CLI 
>sudo apt-get update && sudo apt-get install jq
>peer channel create -o orderer.example.com:7050 -c foo -f ./channel-artifacts/channel.tx
>peer channel fetch config config_block.pb -o orderer.example.com:7050 -c foo --cafile $ORDERER_CA
>configtxlator proto_decode --input config_block.pb --type common.Block | jq .data.data[0].payload.data.config > config.json
>jq -s '.[0] * {"channel_group":{"groups":{"Application":{"groups": {"Org2MSP":.[1]}}}}}' config.json ./channel-artifacts/org2.json > modified_config.json
>configtxlator proto_encode --input config.json --type common.Config --output config.pb
>configtxlator proto_encode --input modified_config.json --type common.Config --output modified_config.pb
>configtxlator compute_update --channel_id foo --original config.pb --updated modified_config.pb --output org2_update.pb
>configtxlator proto_decode --input org2_update.pb --type common.ConfigUpdate | jq . > org2_update.json
>echo '{"payload":{"header":{"channel_header":{"channel_id":"foo", "type":2}},"data":{"config_update":'$(cat org2_update.json)'}}}' | jq . > org2_update_in_envelope.json
>configtxlator proto_encode --input org2_update_in_envelope.json --type common.Envelope --output org2_update_in_envelope.pb
>peer channel signconfigtx -f org2_update_in_envelope.pb
>peer channel update -f org2_update_in_envelope.pb -c foo -o orderer.example.com:7050 --cafile $ORDERER_CA


PEER 2 (Org2)
docker run --rm -it --network="my-net" \
--link orderer.example.com:orderer.example.com \
--link peer0.org1.example.com:peer0.org1.example.com \
--name peer0.org2.example.com \
-p 9051:7051 \
-p 9053:7053 \
-e CORE_PEER_ADDRESSAUTODETECT=true \
-e CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock \
-e CORE_LOGGING_LEVEL=DEBUG \
-e CORE_PEER_NETWORKID=peer0.org2.example.com \
-e CORE_NEXT=true \
-e CORE_PEER_ENDORSER_ENABLED=true \
-e CORE_PEER_ID=peer0.org2.example.com \
-e CORE_PEER_PROFILE_ENABLED=true \
-e CORE_PEER_COMMITTER_LEDGER_ORDERER=orderer.example.com:7050 \
-e CORE_PEER_GOSSIP_ORGLEADER=true \
-e CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org2.example.com:7051 \
-e CORE_PEER_GOSSIP_IGNORESECURITY=true \
-e CORE_PEER_LOCALMSPID=Org2MSP \
-e CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=my-net \
-e CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org2.example.com:7051 \
-e CORE_PEER_GOSSIP_USELEADERELECTION=false \
-e CORE_PEER_TLS_ENABLED=false \
-v /var/run/:/host/var/run/ \
-v $(pwd)/crypto-config/peerOrganizations/org2.example.com/peers/SYNL901761/msp:/etc/hyperledger/fabric/msp \
-w /opt/gopath/src/github.com/hyperledger/fabric/peer \
hyperledger/fabric-peer \
peer node start


CLI2 
docker run --rm -it --network="my-net" \
--name cli \
--link orderer.example.com:orderer.example.com \
--link peer0.org1.example.com:peer0.org1.example.com \
--link peer0.org2.example.com:peer0.org2.example.com \
-p 12051:7051 \
-p 12053:7053 \
-e GOPATH=/opt/gopath \
-e CORE_PEER_LOCALMSPID=Org2MSP \
-e CORE_PEER_TLS_ENABLED=false \
-e CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock \
-e CORE_LOGGING_LEVEL=DEBUG \
-e CORE_PEER_ID=cli \
-e CORE_PEER_ADDRESS=peer0.org2.example.com:7051 \
-e CORE_PEER_NETWORKID=cli \
-e CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/	msp \
-e ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem \
-e CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=my-net  \
-v /var/run/:/host/var/run/ \
-v $(pwd)/chaincode/:/opt/gopath/src/github.com/hyperledger/fabric/examples/chaincode/go \
-v $(pwd)/crypto-config:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ \
-v $(pwd)/scripts:/opt/gopath/src/github.com/hyperledger/fabric/peer/scripts/ \
-v $(pwd)/channel-artifacts:/opt/gopath/src/github.com/hyperledger/fabric/peer/channel-artifacts \
-w /opt/gopath/src/github.com/hyperledger/fabric/peer \
hyperledger/fabric-tools \
/bin/bash

Later run belowin CLI 2 (Post Update config)
>peer channel fetch 0 foo.block -o orderer.example.com:7050 -c foo --cafile $ORDERER_CA
>peer channel join -b foo.block


At this point Org2 Peer0 is added.


Chain Code Installation 

Install Chain Code CLI 1
peer chaincode install -n mycc -v 1.0 -p github.com/hyperledger/fabric/examples/chaincode/go/chaincode/chaincode_example02/go
Install Chain Code CLI 2
peer chaincode install -n mycc -v 1.0 -p github.com/hyperledger/fabric/examples/chaincode/go/chaincode/chaincode_example02/go

Instantiate
CLI 1
Error: Error endorsing query: rpc error: code = Unknown desc = access denied: channel [foo] creator org [Org1MSP] - <nil>
CLI 2
>peer chaincode instantiate -o orderer.example.com:7050 --cafile $ORDERER_CA -C foo -n mycc -v 1.0 -c '{"Args":["init","a", "100", "b","200"]}' -P "AND ('Org1MSP.peer','Org2MSP.peer')"

Query
CLI 1
Error: Error endorsing query: rpc error: code = Unknown desc = access denied: channel [foo] creator org [Org1MSP] - <nil>
CLI 2
>peer chaincode query -C foo -n mycc -c '{"Args":["query","a"]}'


Invoke
CLI1 :
Error: Error endorsing query: rpc error: code = Unknown desc = access denied: channel [foo] creator org [Org1MSP] - <nil>
CLI 2:
peer chaincode invoke -o orderer.example.com:7050 --cafile $ORDERER_CA -Cfoo -n mycc -c '{"Args":["invoke","a","b","10"]}'

