# fabric v2.0 Raft多机部署

## 前期准备

1. 物理环境：3台可通过网络进行通信的服务器，此处操作系统为Ubuntu 18.04

2. 在每一台服务器上都需要安装**<u>Fabric v2.0</u>**

3. 各机器上service部署情况如下表所示：

   | 机器IP        | service列表                                                  |
   | ------------- | ------------------------------------------------------------ |
   | 192.168.70.20 | `ca-org1`、`orderer1.example.com`、`peer0.org1.example.com`、`couchdb0` |
   | 192.168.70.21 | `ca-org2`、`orderer2.example.com`、`peer0.org2.example.com`、`couchdb0` |
   | 192.168.70.22 | `ca-orderer`、`orderer3.example.com`                         |

## 搭建fabric区块链网络

### Org(以Org1为例)

1. 为Org1及其组件注册并生成证书

   - 启用`ca-org1`节点：`docker-compose -f Org1/docker-compose-ca.yaml`

   - 切换到工作目录：`cd Org1`

   - 为`ca-org1`管理员生成证书

     ```bash
     mkdir -p crypto-config/peerOrganizations/org1.example.com
     export FABRIC_CA_CLIENT_HOME=${PWD}/crypto-config/peerOrganizations/org1.example.com
     export FABRIC_CA_CLIENT_TLS_CERTFILES=${PWD}/fabric-ca-server/tls-cert.pem
     fabric-ca-client enroll -u https://admin:adminpw@192.168.70.20:7054 --caname ca-org1
     ```

   - 为节点及用户注册(**register**)并生成(**enroll**)证书

     - 确保已设置环境变量：`FABRIC_CA_CLIENT_HOME`、`FABRIC_CA_CLIENT_TLS_CERTFILES`

     - 为节点注册并生成证书(以`peer0.org1.example.com`为例)

       ```bash
       # 01注册证书
       fabric-ca-client register --caname ca-org1 --id.name peer0.org1.example.com --id.secret peer0pw --id.type peer --id.attrs '"hf.Registrar.Roles=peer"'
       
       # 02生成证书(mspz证书和tls证书)
       # msp证书
       fabric-ca-client enroll -u https://peer0.org1.example.com:peer0pw@192.168.70.20:7054 --caname ca-org1 -M ${PWD}/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/msp --csr.hosts peer0.org1.example.com
       # tls证书
       fabric-ca-client enroll -u https://peer0.org1.example.com:peer0pw@192.168.70.20:7054 --caname ca-org1 -M ${PWD}/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls --enrollment.profile tls --csr.hosts peer0.org1.example.com
       # 修改部分证书名称及路径，从而使其与docker-compose相对应
       cd crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com
       cp tls/signcerts/* tls/server.crt
       cp tls/keystore/* tls/server.key
       cp tls/tlscacerts/* tls/ca.crt
       
       # 以下代码仅需执行一次
       mkdir msp/tlscacerts
       cp tls/tlscacerts/* msp/tlscacerts/tlsca.org1.example.com-cert.pem
       mkdir ../../msp/tlscacerts
       cp tls/tlscacerts/* ../../msp/tlscacerts/tlsca.org1.example.com-cert.pem
       # 切换回工作目录
       cd  -
       ```
       
     - 为用户注册并生成证书(以`Admin@org1.example.com`和 `User@org1.example.com` 为例)
     
       ```bash
       # 01注册证书
       # 为Admin@org1.example.com注册证书
       fabric-ca-client register --caname ca-org1 --id.name org1admin --id.secret org1adminpw --id.type admin --id.attrs '"hf.Registrar.Roles=admin"'
       # 为User@org1.example.com注册证书
       fabric-ca-client register --caname ca-org1 --id.name user1 --id.secret user1pw --id.type client --id.attrs '"hf.Registrar.Roles=client"'
       
       # 02生成证书(仅msp证书)
       # 为Admin@org1.example.com生成msp证书
       fabric-ca-client enroll -u https://org1admin:org1adminpw@192.168.70.20:7054 --caname ca-org1 -M ${PWD}/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
       # 为User@org1.example.com生成msp证书
       fabric-ca-client enroll -u https://user1:user1pw@192.168.70.20:7054 --caname ca-org1 -M ${PWD}/crypto-config/peerOrganizations/org1.example.com/users/User@org1.example.com/msp
       ```
     
     - 向上述所有的**<u>msp</u>**目录拷贝`config.yaml`文件。`config.yaml`文件内容如下：
     
         ```yaml
         NodeOUs:
           Enable: true
           ClientOUIdentifier:
             Certificate: cacerts/192-168-70-20-7054-ca-org1.pem
             OrganizationalUnitIdentifier: client
           PeerOUIdentifier:
             Certificate: cacerts/192-168-70-20-7054-ca-org1.pem
             OrganizationalUnitIdentifier: peer
           AdminOUIdentifier:
             Certificate: cacerts/192-168-70-20-7054-ca-org1.pem
             OrganizationalUnitIdentifier: admin
           OrdererOUIdentifier:
             Certificate: cacerts/192-168-70-20-7054-ca-org1.pem
             OrganizationalUnitIdentifier: orderer
         ```

2. 启用`peer0.org1.example.com`节点：`docker-compose -f Org1/docker-compose-org1.yaml -f Org1/docker-compose-couch.yaml up -d`


### Orderer

1. 为Orderer及其组件注册并生成证书

   - 启用`ca-orderer`节点： `docker-compose -f OrdererOrg/docker-compose-ca.yaml up -d`

   - 切换到工作目录：`cd OrdererOrg`

   - 为`ca-orderer`管理员生成证书并将其分发给其他机器

     ```bash
     mkdir -p crypto-config/ordererOrganizations/example.com
     export FABRIC_CA_CLIENT_HOME=${PWD}/crypto-config/ordererOrganizations/example.com
     export FABRIC_CA_CLIENT_TLS_CERTFILES=${PWD}/fabric-ca-server/tls-cert.pem
     fabric-ca-client enroll -u https://admin:adminpw@192.168.70.22:7054 --caname ca-orderer
     
     scp -P 8888 -r crypto-config ubuntu@192.168.70.20:$PWD
     scp -P 8888 -r crypto-config ubuntu@192.168.70.21:$PWD
     ```

   - 为节点及用户注册(**register**)并生成(**enroll**)证书

     - 确保已设置环境变量：`FABRIC_CA_CLIENT_HOME`、`FABRIC_CA_CLIENT_TLS_CERTFILES`

     - 为节点注册并生成证书(以`orderer1.example.com`为例)

       ```bash
       # 01注册证书(可以在任意一台机器上执行)
       fabric-ca-client register --caname ca-orderer --id.name orderer1.example.com --id.secret orderer1pw --id.type orderer --id.attrs '"hf.Registrar.Roles=orderer"'
       
       # 02生成证书(msp证书和tls证书)
       # 在节点所在机器上各自执行(以orderer1.example.com节点为例，以下操作应该在192.168.70.20上执行)
       # msp证书
       fabric-ca-client enroll -u https://orderer1.example.com:orderer1pw@192.168.70.22:7054 --caname ca-orderer -M ${PWD}/crypto-config/ordererOrganizations/example.com/orderers/orderer1.example.com/msp --csr.hosts orderer1.example.com
       # tls证书
       fabric-ca-client enroll -u https://orderer1.example.com:orderer1pw@192.168.70.22:7054 --caname ca-orderer -M ${PWD}/crypto-config/ordererOrganizations/example.com/orderers/orderer1.example.com/tls --enrollment.profile tls --csr.hosts orderer1.example.com
       # 修改部分证书名称及路径，从而使其与docker-compose相对应
       cd crypto-config/ordererOrganizations/example.com/orderers/orderer1.example.com
       cp tls/signcerts/* tls/server.crt
       cp tls/keystore/* tls/server.key
       cp tls/tlscacerts/* tls/ca.crt
       mkdir msp/tlscacerts
       cp tls/tlscacerts/* msp/tlscacerts/tlsca.example.com-cert.pem
       # 以下代码仅需执行一次
       mkdir ../../msp/tlscacerts
       cp tls/tlscacerts/* ../../msp/tlscacerts/tlsca.example.com-cert.pem
       # 切换回工作目录
       cd  -
       ```
       
     - 为用户注册并生成证书(以`Admin@example.com`为例)
     
       ```bash
       # 01注册证书
       fabric-ca-client register --caname ca-orderer --id.name ordererAdmin --id.secret ordererAdminpw --id.type admin --id.attrs '"hf.Registrar.Roles=admin"'
       # 02生成证书(仅msp证书)
       fabric-ca-client enroll -u https://ordererAdmin:ordererAdminpw@192.168.70.22:7054 --caname ca-orderer -M ${PWD}/crypto-config/ordererOrganizations/example.com/users/Admin@example.com/msp
       ```

2. 生成创世区块

   - 将各组织的msp目录拷贝到`./configtx/MSPs`下，并通过scp命令将其发送到orderer节点所在机器。

     ```bash
     # 在192.168.70.20上执行下列操作
     cp -r Org1/crypto-config/peerOrganizations/org1.example.com/msp/* configtx/MSPs/Org1
     cd configtx/MSPs
     scp -P 8888 -r Org1 ubuntu@192.168.70.22:$PWD
     
     # 在192.168.70.21上执行下列操作
     cp -r Org2/crypto-config/peerOrganizations/org2.example.com/msp/* configtx/MSPs/Org2
     cd configtx/MSPs
     scp -P 8888 -r Org2 ubuntu@192.168.70.22:$PWD
     
     # 在192.168.70.22上执行下列操作
     cp -r OrdererOrg/crypto-config/ordererOrganizations/example.com/msp/* configtx/MSPs/OrdererOrg
     ```

   - orderer节点上修改`configtx/configtx.yaml`文件

     - 修改**<u>*Organization*</u>**节每个组织的`MSPDir`，此处需要按实际情况进行更改，实例如下所示：

       ```yaml
       Organizations:
           - &OrdererOrg
               ...
               MSPDir: MSPs/OrdererOrg
               ...
           - &Org1
               ...
               MSPDir: MSPs/Org1
               ...
               AnchorPeers:
                   - Host: peer0.org1.example.com
                     Port: 7051
           - &org2
               ...
               MSPDir: MSPs/Org2
               ...
               AnchorPeers:
                   - Host: peer0.org2.example.com
                     Port: 7051
       ```

     - 修改**<u>*Orderer*</u>**节的`Addresses`如下所示：

       ```yaml
       Orderer: &OrdererDefaults	# 按实际情况进行更改(非常重要)
           ...
           Addresses:
               - orderer1.example.com:7050
           ...
       ```

     - 在`Profiles`节添加下列内容：

       ```yaml
       Profiles:	# 按实际情况进行更改(非常重要)
           # 添加多节点EtcdRaft创世区块配置
           SampleMultiNodeEtcdRaft:
               <<: *ChannelDefaults
               Capabilities:
                   <<: *ChannelCapabilities
               Orderer:
                   <<: *OrdererDefaults
                   OrdererType: etcdraft
                   EtcdRaft:
                       Consenters:
                       - Host: orderer1.example.com
                         Port: 7050
                         ClientTLSCert: ../OrdererOrg/crypto-config/ordererOrganizations/example.com/orderers/orderer1.example.com/tls/server.crt
                         ServerTLSCert: ../OrdererOrg/crypto-config/ordererOrganizations/example.com/orderers/orderer1.example.com/tls/server.crt
                       - Host: orderer2.example.com
                         Port: 7050
                         ClientTLSCert: ../OrdererOrg/crypto-config/ordererOrganizations/example.com/orderers/orderer2.example.com/tls/server.crt
                         ServerTLSCert: ../OrdererOrg/crypto-config/ordererOrganizations/example.com/orderers/orderer2.example.com/tls/server.crt
                       - Host: orderer3.example.com
                         Port: 7050
                         ClientTLSCert: ../OrdererOrg/crypto-config/ordererOrganizations/example.com/orderers/orderer3.example.com/tls/server.crt
                         ServerTLSCert: ../OrdererOrg/crypto-config/ordererOrganizations/example.com/orderers/orderer3.example.com/tls/server.crt
                   Addresses:
                       - orderer1.example.com:7050
                       - orderer2.example.com:7050
                       - orderer3.example.com:7050
                   Organizations:
                   - *OrdererOrg
                   Capabilities:
                       <<: *OrdererCapabilities
               Application:
                   <<: *ApplicationDefaults
                   Organizations:
                   - <<: *OrdererOrg
               Consortiums:
                   SampleConsortium:
                       Organizations:
                       - *Org1
                       - *Org2
       ```

   - 生成创世区块并将其分发给其他机器：

     ```bash
     # 生成创世区块
     FABRIC_CFG_PATH=${PWD}/configtx configtxgen -profile SampleMultiNodeEtcdRaft -channelID system-channel -outputBlock OrdererOrg/genesis.block
     
     # 分发给其他机器(192.168.70.20、192.168.70.21)
     cd OrdererOrg
     scp -P 8888 genesis.block ubuntu@192.168.70.20:$PWD
     scp -P 8888 genesis.block ubuntu@192.168.70.21:$PWD
     cd -
     ```

   - 启动所有的orderer节点

     ```bash
     # 在192.168.70.20所在机器上执行
     docker-compose -f OrdererOrg/docker-compose-orderer1.yaml up -d
     
     # 在192.168.70.21所在机器上执行
     docker-compose -f OrdererOrg/docker-compose-orderer2.yaml up -d
     
     # 在192.168.70.22所在机器上执行
     docker-compose -f OrdererOrg/docker-compose-orderer3.yaml up -d
     ```

   - 生成通道配置tx及锚节点tx并将其分发给各自机器：

     ```bash
     # 生成通道配置tx
     FABRIC_CFG_PATH=${PWD}/configtx configtxgen -profile TwoOrgsChannel -channelID mychannel -outputCreateChannelTx Org1/channel-artifacts/mychannel.tx
     # 为Org1生成锚节点tx
     FABRIC_CFG_PATH=${PWD}/configtx configtxgen -profile TwoOrgsChannel -channelID mychannel -asOrg Org1MSP -outputAnchorPeersUpdate Org1/channel-artifacts/Org1MSPanchors.tx
     # 分发给Org1所在机器(192.168.70.20)
     cd Org1
     scp -P 8888 -r channel-artifacts ubuntu@192.168.70.20:$PWD
     cd -
     
     # 为Org2生成锚节点tx
     FABRIC_CFG_PATH=${PWD}/configtx configtxgen -profile TwoOrgsChannel -channelID mychannel -asOrg Org2MSP -outputAnchorPeersUpdate Org2/channel-artifacts/Org2MSPanchors.tx
     # 分发给Org2所在机器(192.168.70.21)
     cd Org2
     scp -P 8888 -r channel-artifacts ubuntu@192.168.70.21:$PWD
     cd -
     ```

   - 

3. 启用orderer节点：`docker-compose -f org1/docker/docker-compose-orderer.yaml up -d`