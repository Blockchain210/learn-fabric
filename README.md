# first-network-kafka

Hyperledger Fabric v 1.4中first-network的kafka模式多机部署（将zookeeper、kafka和orderer全部放到同一台机器），各机器及其对应service列表如下图所示：

|    机器IP     |                         service列表                          |
| :-----------: | :----------------------------------------------------------: |
| 192.168.70.20 | `zookeeper0`、` zookeeper1`、`zookeeper2`、`kafka0`、 `kafka1`、`kafka2`、`kafka3`、`orderer0.example.com`、`orderer1.example.com`、`orderer2.example.com` |
| 192.168.70.21 |                 `peer0.org1.example`、`cli`                  |
| 192.168.70.22 |                 `peer1.org1.example`、`cli`                  |
| 192.168.70.43 |                 `peer0.org2.example`、 `cli`                 |
| 192.168.70.44 |                 `peer1.org2.example` 、`cli`                 |

> 注：此部分为first-network多机部署kafka模式的过渡产品，将zookeeper、kafka和orderer全部放到同一台机器，目的只是为了简化部署、方便学习，实际环境中不建议采用此种方式。