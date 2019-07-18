# 从零开始构建Fabric网络（solo）

## 前期准备

1. 安装Fabric v1.4.0
2. 安装Fabric组织及联盟管理工具**FabricMgr-1.0-py3-none-any.whl**

## 构建初始网络（仅有一个orderer节点）

1. 基于crypto-config.yaml为orderer生成证书

   ```bash
   cryptogen generate --config=./crypto-config.yaml
   ```

2. 基于configtx.yaml生成创世区块

   ```bash
   configtxgen -profile OrdererGenesis -channelID byfn-sys-channel -outputBlock ./channel-artifacts/genesis.block
   ```

3. 启动初始网络

   ```bash
   docker-compose -f docker-compose-orderer.yaml up -d
   ```

4. 新建联盟（以新建名称为**TestConsortium**的联盟为例，其中包含两个组织：Org1和Org2）

   - （可选）若联盟中包含的组织尚不存在，则可通过下列方式新建组织（以新建Org1为例）

     ```bash
     fabric-org-mgr neworg Org1 org1.example.com -vvv
     fabric-org-mgr gcompose Or1 org1.example.com net byfn -vvv
     ```

   - 创建联盟

     ```bash
     fabric-consortia-mgr newconsortia TestConsortium byfn-sys-channel -ojf Org1/channel-artifacts/Org1.json -ojf Org2/channel-artifacts/Org2.json -vvv
     ```

5. 在特定机器启动特定Org的docker-compose-peer*.yaml（以启动Org1的docker-compose-peer0.yaml为例）

   ```bash
   # 将orderer节点的证书及密钥文件夹拷贝到Org1/crypto-config下(假定当前工作目录为fabric项目根目录、Org1位于first-network的直接子目录)
   cp -r crypto-config/ordererOrganizations Org1/crypto-config
   # 启动docker-compose-peer0.yaml
   cd Org1
   docker-compose -f docker-compose-peer0.yaml up -d
   ```

6. 创建通道配置Transaction（以新建mychannel为例，假定其所属联盟为TestConsortium）

   - 在`configtx.yaml`的`Section: Organizations`中添加名称为TestConsortium的联盟中包含的组织（即将`fabric-org-mgr neworg Org org.example.com -vvv`生成的Org目录下的configtx.yaml中的内容放到网络的configtx.yaml中），详情参考`configtx-mychannel.yaml` 。

     **注意：此时新建Org的`MSPDir`必须是其相对于网络的configtx.yaml的相对路径**。

   - 向`configtx.yaml`的`Section: Profile`中创建Orderer创世区块的配置profile中添加TestConsortium联盟

     ```yaml
     OrdererGenesis:
     	<<: *ChannelDefaults
     	Orderer:
         	<<: *OrdererDefaults
             Organizations:
             	- *OrdererOrg
             Capabilities:
             	<<: *OrdererCapabilities
         Consortiums:
         	SampleConsortium:
             	Organizations:
                 	- *OrdererOrg
             TestConsortium:
             	Organizations:
                 	- *Org1
                     - *Org2
     ```

   - 向`configtx.yaml`的`Section: Profile`中添加新建通道的配置profile

     ```yaml
     TwoOrgsChannel:
     	Consortium: TestConsortium
         Application:
         	<<: *ApplicationDefaults
             Organizations:
             	- *Org1
             	- *Org2
             Capabilities:
             	<<: *ApplicationCapabilities
     ```

   - 创建通道配置Transaction

     ```bash
     # 生成mychannel.tx
     configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./Org1/channel-artifacts/mychannel.tx -channelID mychannel
     
     # 为我们正在创建的channel中的Org定义锚节点
     # 为Org1定义锚节点
     configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./Org1/channel-artifacts/Org1MSPanchors.tx -channelID mychannel -asOrg Org1MSP
     # 为Org2定义锚节点
     configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./Org2/channel-artifacts/Org2MSPanchors.tx -channelID mychannel -asOrg Org2MSP
     ```

7. 新建通道并将peer节点加入通道中

   ```bash
   # 进入cli容器
   docker exec -it Org1cli bash
   
   # 设置环境变量以方便后面的使用
   export CHANNEL_NAME=mychannel
   export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
   
   # 创建Channel
   # 生成的mychannel.block是peer节点加入Channel的许可
   peer channel create -o orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/mychannel.tx --tls --cafile $ORDERER_CA
   
   # 将peer0.org1.example.com加入到mychannel通道中
   peer channel join -b mychannel.block
   ```