# Windows 上的 Hyperledger Fabric开发配置

## Hyperledger Fabric在 windows 机器上的开发环境配置

### 前提条件

* [已经安装好WSL2](../../docs/Productivity/wsl2-dev.md)
* [借鉴资料](https://www.codementor.io/@arvindmaurya/hyperledger-fabric-on-windows-1hjjorw68p)
* 已经安装好 git curl golang docker

### 步骤一 安装 Hyperledger Fabric and Fabric samples

* 下载最新的 fabric samples 和 docker images

    ```cmd
    mkdir -p $HOME/go/src/github.com/
    cd $HOME/go/src/github.com/
    curl -sSL https://bit.ly/2ysbOFE | bash -s
    ```

* 查看已经pull的docker images

    ```cmd
    docker images
    REPOSITORY                   TAG       IMAGE ID       CREATED       SIZE
    hyperledger/fabric-tools     2.4       46e728e02f21   4 weeks ago   489MB
    hyperledger/fabric-tools     2.4.6     46e728e02f21   4 weeks ago   489MB
    hyperledger/fabric-tools     latest    46e728e02f21   4 weeks ago   489MB
    hyperledger/fabric-peer      2.4       d88ae875cc38   4 weeks ago   64.2MB
    hyperledger/fabric-peer      2.4.6     d88ae875cc38   4 weeks ago   64.2MB
    hyperledger/fabric-peer      latest    d88ae875cc38   4 weeks ago   64.2MB
    hyperledger/fabric-orderer   2.4       f4b44e136877   4 weeks ago   36.7MB
    hyperledger/fabric-orderer   2.4.6     f4b44e136877   4 weeks ago   36.7MB
    hyperledger/fabric-orderer   latest    f4b44e136877   4 weeks ago   36.7MB
    hyperledger/fabric-ccenv     2.4       32368d1f15d4   4 weeks ago   520MB
    hyperledger/fabric-ccenv     2.4.6     32368d1f15d4   4 weeks ago   520MB
    hyperledger/fabric-ccenv     latest    32368d1f15d4   4 weeks ago   520MB
    hyperledger/fabric-baseos    2.4       dc5d59da5a8f   4 weeks ago   6.86MB
    hyperledger/fabric-baseos    2.4.6     dc5d59da5a8f   4 weeks ago   6.86MB
    hyperledger/fabric-baseos    latest    dc5d59da5a8f   4 weeks ago   6.86MB
    hyperledger/fabric-ca        1.5       93f19fa873cb   8 weeks ago   76.5MB
    hyperledger/fabric-ca        1.5.5     93f19fa873cb   8 weeks ago   76.5MB
    hyperledger/fabric-ca        latest    93f19fa873cb   8 weeks ago   76.5MB
    ```

* 查看文件结构

    ```cmd
    cd $HOME/go/src/github.com/
    ls ./fabric-samples
    CHANGELOG.md          asset-transfer-events             ci                        test-network
    CODEOWNERS            asset-transfer-ledger-queries     commercial-paper          test-network-k8s
    CODE_OF_CONDUCT.md    asset-transfer-private-data       config                    test-network-nano-bash
    CONTRIBUTING.md       asset-transfer-sbe                fabcar                    token-erc-1155
    LICENSE               asset-transfer-secured-agreement  hardware-security-module  token-erc-20
    MAINTAINERS.md        auction-dutch                     high-throughput           token-erc-721
    README.md             auction-simple                    interest_rate_swaps       token-utxo
    SECURITY.md           bin                               off_chain_data
    asset-transfer-abac   builders                          scripts
    asset-transfer-basic  chaincode                         test-application
    ```

### 步骤二 运行 fabric test network

* 删除现有的任何容器或以前运行的组件

    ```cmd
    cd $HOME/go/src/github.com/fabric-samples/test-network
    ./network.sh down
    ```

* 启动测试网络

    ```cmd
    ./network.sh up
    ```

* 查看生成的容器

    ```cmd
    docker ps -a
    CONTAINER ID   IMAGE                               COMMAND             CREATED              STATUS              PORTS                                                                    NAMES
    90dbff0f8113   hyperledger/fabric-tools:latest     "/bin/bash"         About a minute ago   Up About a minute                                                                            cli
    911f3eb0298f   hyperledger/fabric-peer:latest      "peer node start"   About a minute ago   Up About a minute   0.0.0.0:7051->7051/tcp, 0.0.0.0:9444->9444/tcp                           peer0.org1.example.com
    9ce4ec2ed9b7   hyperledger/fabric-orderer:latest   "orderer"           About a minute ago   Up About a minute   0.0.0.0:7050->7050/tcp, 0.0.0.0:7053->7053/tcp, 0.0.0.0:9443->9443/tcp   orderer.example.com
    4ce063c77751   hyperledger/fabric-peer:latest      "peer node start"   About a minute ago   Up About a minute   0.0.0.0:9051->9051/tcp, 7051/tcp, 0.0.0.0:9445->9445/tcp                 peer0.org2.example.com
    ```

### 总结

**基本上很连贯的过程,有问题建议谷歌**

**剩下的流程建议查看官方文档咯**

**[hyperledger-fabric](https://hyperledger-fabric.readthedocs.io/zh_CN/latest/whatis.html)**
