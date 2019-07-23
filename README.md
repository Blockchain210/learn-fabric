# Hyperledger Fabric学习



该项目主要包含以下分支，以下是对各分支的简单介绍

- **kafka**：将Hyperledger Fabric v1.4.0中fabric-samples的first-network单机版的kafka改造为多机部署，在改造初期，由于对zookeeper和kafka了解有限，为了简化部署、方便学习，将zookeeper、kafka和orderer全部放到同一台机器。

  > 注：此部分为first-network多机部署（kafka模式）的过渡产品，将zookeeper、kafka和orderer全部放到同一台机器，目的只是为了简化部署、方便学习，实际环境中不建议采用该方式。

- **kafka-multihost**：Hyperledger Fabric v1.4.0中fabric-samples的<u>first-networ多机部署（kafka模式）</u>，是kafka分支的升级版，实际环境可以参考该部署方式。

- **solo-from-orderer**：从零开始构建Fabric网络（solo模式），由于实际开发过程中，一开始就设计好Fabric网络的概率比较小（个人认为可忽略），因此一个最简单的fabric网络实际上只包含orderer节点，其他Org都是在随后进行逐步添加的，该分支旨在完成从零开始构建Fabric网络的过程，当前仅测试了solo模式，kafka模式后续测试完成后会更新相关内容。