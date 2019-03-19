# first-network多机部署（kafka共识）

## 前期准备

1. 物理环境：5台可通过网络进行通信的服务器，此处操作系统为Ubuntu 18.04
2. 在每一台服务器上都需要**<u>安装Fabric v1.4</u>**
3. 各机器上service部署情况如下表所示：
	|    机器IP     |                         service列表                          |
| :-----------: | :----------------------------------------------------------: |
| 192.168.70.20 |        `zookeeper0`、`kafka0`、`orderer0.example.com`        |
| 192.168.70.21 |       `orderer1.example.com`、`peer0.org1.example.com`       |
| 192.168.70.22 | `zookeeper2`、`kafka2`、`orderer2.example.com`、`peer1.org1.example.com` |
| 192.168.70.43 |              `kafka3`、`peer0.org2.example.com`              |
| 192.168.70.44 |       `zookeeper1`、`kafka1`、`peer2.org2.example.com`       |

## 证书及通道配置文件

### `crypto-config.yaml`



### `configtx.yaml`



## Docker-compose配置文件

### `zookeeper`和`kafka`节点配置



### `orderer`节点配置



### `peer`节点配置



## 启动多节点集群

|    机器IP     | 执行的命令                                                   |
| :-----------: | :----------------------------------------------------------- |
| 192.168.70.20 | `docker-compose -f zookeeper0.yaml -f kafka0.yaml -f orderer0.yaml up -d` |
| 192.168.70.21 | `docker-compose -f orderer1.yaml -f peer0.yaml up -d`        |
| 192.168.70.22 | `docker-compose -f zookeeper2.yaml -f kafka2.yaml -f orderer2.yaml -f peer1.yaml up -d` |
| 192.168.70.43 | `docker-compose -f kafka3.yaml -f peer2.yaml up -d`          |
| 192.168.70.44 | `docker-compose -f zookeeper1.yaml -f kafka1.yaml -f peer3.yaml up -d` |