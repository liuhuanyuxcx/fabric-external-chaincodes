# 使用k8s的pod启动Hyperledger fabric的链码容器

## 一、环境要求
- 本实验环境使用的是centos7
- 本实验环境使用的是1.16.8版本的k8s
- 本实验环境使用的是18.06.3-ce版本的docker
- 本实验环境使用的是1.14.4版本的golang

## 二、镜像准备：
### Hyperledger fabric相关镜像：
- hyperledger/fabric-tools:amd64-2.1.0
- hyperledger/fabric-peer:amd64-2.1.0
- hyperledger/fabric-orderer:amd64-2.1.0
### 链码编译和运行相关镜像：
- golang:1.14.4-alpine3.12
- alpine:3.12

## 三、安装fabric的证书生成器cryptogen和通道生成器configtxgen
### 方法一：下载（由于网络原因，本实验采取的是方法二）
```sh
wget https://github.com/hyperledger/fabric/releases/download/v2.1.0/hyperledger-fabric-linux-amd64-2.1.0.tar.gz

tar -xzf hyperledger-fabric-linux-amd64-2.1.0.tar.gz

# Move to the bin path
mv bin/* /bin

# Check that you have successfully installed the tools by executing
configtxgen --version

# Should print the following output:
# configtxgen:
#  Version: 2.1.0
#  Commit SHA: 1bdf975
#  Go version: go1.14.1
#  OS/Arch: linux/amd64
```
### 方法二：源码编译
```
# 下载fabric源码
mkdir -p $GOPATH/src/github.com/hyperledger
cd $GOPATH/src/github.com/hyperledger
git clone https://github.com/hyperledger/fabric.git
cd fabric
# 编译
make native
# 移动到GOPATH下的bin目录，供fabricOps.sh使用
mv $GOPATH/src/github.com/hyperledger/fabric/build/bin $GOPATH/

```

## 四、启动Hyperledger Fabric网络
### 1.生成证书和创始区块，需要在本项目目录下执行

```sh
./fabricOps.sh start
```

### 2.创建一个文件夹来保存Hyperledger Fabric工作负载的持久化数据：
```sh
mkdir ~/storage
```

### 3.启动fabric网络
```
kubectl create ns hyperledger
kubectl create -f orderer-service/
kubectl create -f org1/
kubectl create -f org2/
```

### 4.检查Fabric网络情况：
```
kubectl get pods -n hyperledger

### 正常情况应该按如下显示：
NAME                          READY   STATUS    RESTARTS   AGE
cli-org1-bc9f895f6-zmmdc      1/1     Running   0          55m
cli-org2-7779cc8788-8q4ns     1/1     Running   0          2m20s
orderer0-5866b6bd7-pflf7      1/1     Running   0          131m
orderer1-c4fd65c7d-c27ll      1/1     Running   0          131m
orderer2-557cb7865-wlcmh      1/1     Running   0          131m
peer0-org1-798b974467-vv4zz   1/1     Running   0          71m
peer0-org2-5849c55fcd-mbn5h   1/1     Running   0          2m19s
```
### 5.设置Fabric网络的通道

```
### 进入org1的cli操作
kubectl exec -it cli_org1_pod_name sh -n hyperledger
peer channel create -o orderer0:7050 -c mychannel -f ./scripts/channel-artifacts/channel.tx --tls true --cafile $ORDERER_CA
peer channel join -b mychannel.block
### 进入org2的cli操作
kubectl exec -it cli_org2_pod_name sh -n hyperledger
peer channel fetch 0 mychannel.block -c mychannel -o orderer0:7050 --tls --cafile $ORDERER_CA
peer channel join -b mychannel.block
```

### 6.在Fabric节点安装外部链码

```
### 进入org1的cli操作
kubectl exec -it cli_org1_pod_name sh -n hyperledger

### 创建connection.json，其中包含连接外部链码服务器的信息，例如地址、TLS证书、连接超时配置等。
cat <<EOF >  connection.json
{
    "address": "chaincode-e2e-org1.hyperledger:7052",
    "dial_timeout": "10s",
    "tls_required": false,
    "client_auth_required": false,
    "client_key": "-----BEGIN EC PRIVATE KEY----- ... -----END EC PRIVATE KEY-----",
    "client_cert": "-----BEGIN CERTIFICATE----- ... -----END CERTIFICATE-----",
    "root_cert": "-----BEGIN CERTIFICATE---- ... -----END CERTIFICATE-----"
}
EOF

### 创建metadata.json，其中包含了链码类型、路径、标签信息：
cat <<EOF > metadata.json
{"path":"","type":"external","label":"e2e"}
EOF

### 打包
tar cfz code.tar.gz connection.json
tar cfz e2e-org1.tgz code.tar.gz metadata.json
peer lifecycle chaincode install e2e-org1.tgz

### 记下来上面的链码包标识符，我们接下来会用到，不过你也可以随时用下面的命令查询链码包的标识符：
peer lifecycle chaincode queryinstalled

### 进入org2的cli操作
kubectl exec -it cli_org2_pod_name sh -n hyperledger

### 创建connection.json，其中包含连接外部链码服务器的信息，例如地址、TLS证书、连接超时配置等。
cat <<EOF >  connection.json
{
    "address": "chaincode-e2e-org2.hyperledger:7052",
    "dial_timeout": "10s",
    "tls_required": false,
    "client_auth_required": false,
    "client_key": "-----BEGIN EC PRIVATE KEY----- ... -----END EC PRIVATE KEY-----",
    "client_cert": "-----BEGIN CERTIFICATE----- ... -----END CERTIFICATE-----",
    "root_cert": "-----BEGIN CERTIFICATE---- ... -----END CERTIFICATE-----"
}
EOF

### 创建metadata.json，其中包含了链码类型、路径、标签信息：
cat <<EOF > metadata.json
{"path":"","type":"external","label":"e2e"}
EOF

### 打包
tar cfz code.tar.gz connection.json
tar cfz e2e-org2.tgz code.tar.gz metadata.json
peer lifecycle chaincode install e2e-org2.tgz
```

## 五、Fabric外部链码的构建与部署一：e2e

### 1.编译链码
```
cd e2e
docker build -t chaincode/e2e:1.0 .
```
### 2.修改链码部署文件k8s/org1-chaincode-deployment.yaml和k8s/org2-chaincode-deployment.yaml中的CHAINCODE_CCID变量为你需要安装的链码。
```
#---------------- Chaincode Deployment ---------------------
apiVersion: apps/v1 # for versions before 1.8.0 use apps/v1beta1
kind: Deployment
metadata:
  name: chaincode-e2e-org1
  namespace: hyperledger
  labels:
    app: chaincode-e2e-org1
spec:
  selector:
    matchLabels:
      app: chaincode-e2e-org1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: chaincode-e2e-org1
    spec:
      containers:
        - image: chaincode/e2e:1.0
          name: chaincode-e2e-org1
          imagePullPolicy: IfNotPresent
          env:
            - name: CHAINCODE_CCID
              value: "e2e:0057a2c8e2731b2c09a75f1a6eb79ca078c7936f21fc8c0995b51df3a18f853e"
            - name: CHAINCODE_ADDRESS
              value: "0.0.0.0:7052"
          ports:
            - containerPort: 7052

```
### 3.部署和查看结果
```
kubectl create -f k8s
kubectl get pods -n hyperledger

NAME                                  READY   STATUS    RESTARTS   AGE
chaincode-e2e-org1-fd5c66d4d-b5gm4    1/1     Running   0          7m15s
chaincode-e2e-org2-6f9c8bcd48-mf62k   1/1     Running   0          7m15s
cli-org1-5cffd76cc7-xtllh             1/1     Running   0          14m
cli-org2-55968b6d77-d7s6b             1/1     Running   0          14m
orderer0-7b4bfd577-xj7fw              1/1     Running   0          14m
orderer1-7654ff5fcb-g7brn             1/1     Running   0          14m
orderer2-6746c797f7-74f5r             1/1     Running   0          14m
peer0-org1-69bfcf8564-xtk6w           1/1     Running   0          14m
peer0-org2-cd784948c-btmck            1/1     Running   0          14m
```

### 4.审批和提交链码

```
### org1的cli pod内执行如下命令，记得修改CHAINCODE_CCID
peer lifecycle chaincode approveformyorg --channelID mychannel --name e2e --version 1.0 --init-required --package-id e2e:e00023b290fe1ffe8d8f9b6d23fdc711433444500d7f3ea0d9edf78bb83ebd8b --sequence 1 -o orderer0:7050 --tls --cafile $ORDERER_CA

### org2的cli pod内执行如下命令，记得修改CHAINCODE_CCID
peer lifecycle chaincode approveformyorg --channelID mychannel --name e2e --version 1.0 --init-required --package-id e2e:f441d58b5e543c895c2cb9109acd798c85194604648bf8ef7fda836a1b4952c7 --sequence 1 -o orderer0:7050 --tls --cafile $ORDERER_CA

### 检查链码审批状态
peer lifecycle chaincode checkcommitreadiness --channelID mychannel --name e2e --version 1.0 --init-required --sequence 1 -o -orderer0:7050 --tls --cafile $ORDERER_CA

### 提交链码
peer lifecycle chaincode commit -o orderer0:7050 --channelID mychannel --name e2e --version 1.0 --sequence 1 --init-required --tls true --cafile $ORDERER_CA --peerAddresses peer0-org1:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses peer0-org2:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2/peers/peer0-org2/tls/ca.crt
```

### 5.测试Fabric外部链码

```
### init
peer chaincode invoke -o orderer0:7050 --tls true --cafile $ORDERER_CA -C mychannel -n e2e --peerAddresses peer0-org1:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses peer0-org2:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2/peers/peer0-org2/tls/ca.crt --isInit -c '{"Args":["Init","a","100","b","200"]}' --waitForEvent

### 查询
peer chaincode query -C mychannel -n e2e -c '{"Args":["query","a"]}'

### 调用交易
peer chaincode invoke -o orderer0:7050 --tls true --cafile $ORDERER_CA -C mychannel -n e2e --peerAddresses peer0-org1:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses peer0-org2:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2/peers/peer0-org2/tls/ca.crt -c '{"Args":["invoke","a","b","10"]}' --waitForEvent

### 再次查询
peer chaincode query -C mychannel -n e2e -c '{"Args":["query","a"]}'
```

## 六、Fabric外部链码的构建与部署二：marbles02

### 1.编译链码（此处省略了“四、6.在Fabric节点安装外部链码”的过程，请参考该节自行修改，把e2e改成marbles即可安装链码）
```
cd marbles02
docker build -t chaincode/marbles:1.0 .
```
### 2.修改链码部署文件k8s/org1-chaincode-deployment.yaml和k8s/org2-chaincode-deployment.yaml中的CHAINCODE_CCID变量为你需要安装的链码。
```
#---------------- Chaincode Deployment ---------------------
apiVersion: apps/v1 # for versions before 1.8.0 use apps/v1beta1
kind: Deployment
metadata:
  name: chaincode-marbles-org1
  namespace: hyperledger
  labels:
    app: chaincode-marbles-org1
spec:
  selector:
    matchLabels:
      app: chaincode-marbles-org1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: chaincode-marbles-org1
    spec:
      containers:
        - image: chaincode/marbles:1.0
          name: chaincode-marbles-org1
          imagePullPolicy: IfNotPresent
          env:
            - name: CHAINCODE_CCID
              value: "marbles:0057a2c8e2731b2c09a75f1a6eb79ca078c7936f21fc8c0995b51df3a18f853e"
            - name: CHAINCODE_ADDRESS
              value: "0.0.0.0:7052"
          ports:
            - containerPort: 7052
```
### 3.部署和查看结果
```
kubectl create -f k8s
kubectl get pods -n hyperledger

chaincode-marbles-org1-6fc8858855-wdz7z   1/1     Running   0          20m
chaincode-marbles-org2-77bf56fdfb-6cdfm   1/1     Running   0          14m
cli-org1-589944999c-cvgbx                 1/1     Running   0          19h
cli-org2-656cf8dd7c-kcxd7                 1/1     Running   0          19h
orderer0-5844bd9bcc-6td8c                 1/1     Running   0          46h
orderer1-75d8df99cd-6vbjl                 1/1     Running   0          46h
orderer2-795cf7c4c-6lsdd                  1/1     Running   0          46h
peer0-org1-5bc579d766-kq2qd               1/1     Running   0          19h
peer0-org2-77f58c87fd-sczp8               1/1     Running   0          19h
```

### 4.审批和提交链码

```
### org1的cli pod内执行如下命令，记得修改CHAINCODE_CCID
peer lifecycle chaincode approveformyorg --channelID mychannel --name marbles --version 1.0 --init-required --package-id marbles:d8140fbc1a0903bd88611a96c5b0077a2fdeef00a95c05bfe52e207f5f9ab79d --sequence 1 -o orderer0:7050 --tls --cafile $ORDERER_CA

### org2的cli pod内执行如下命令，记得修改CHAINCODE_CCID
peer lifecycle chaincode approveformyorg --channelID mychannel --name marbles --version 1.0 --init-required --package-id marbles:af45e0faa9676dd884a56e34b6a9bc1f7f1df04d6356aa1b2b9f123bd1d9e9e6 --sequence 1 -o orderer0:7050 --tls --cafile $ORDERER_CA

### 检查链码审批状态
peer lifecycle chaincode checkcommitreadiness --channelID mychannel --name marbles --version 1.0 --init-required --sequence 1 -o -orderer0:7050 --tls --cafile $ORDERER_CA

### 提交链码
peer lifecycle chaincode commit -o orderer0:7050 --channelID mychannel --name marbles --version 1.0 --sequence 1 --init-required --tls true --cafile $ORDERER_CA --peerAddresses peer0-org1:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses peer0-org2:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2/peers/peer0-org2/tls/ca.crt
```

### 5.测试Fabric外部链码

```
### init
peer chaincode invoke -o orderer0:7050 --tls true --cafile $ORDERER_CA -C mychannel -n marbles --peerAddresses peer0-org1:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses peer0-org2:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2/peers/peer0-org2/tls/ca.crt --isInit -c '{"Args":["Init"]}' --waitForEvent

### 创建第一个宝石
peer chaincode invoke -o orderer0:7050 --tls true --cafile $ORDERER_CA -C mychannel -n marbles --peerAddresses peer0-org1:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses peer0-org2:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2/peers/peer0-org2/tls/ca.crt -c '{"Args":["initMarble","marble1","blue","35","tom"]}' --waitForEvent

### 创建第二个宝石
peer chaincode invoke -o orderer0:7050 --tls true --cafile $ORDERER_CA -C mychannel -n marbles --peerAddresses peer0-org1:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses peer0-org2:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2/peers/peer0-org2/tls/ca.crt -c '{"Args":["initMarble","marble2","red","50","tom"]}' --waitForEvent

### 查询宝石
peer chaincode query -C mychannel -n marbles -c '{"Args":["readMarble","marble1"]}'
```

## 问题及解决方法：

### 1.如何指定背书策略
在`peer lifecycle chaincode approveformyorg`和`peer lifecycle chaincode commit`时增加`--signature-policy "AND ('Org1MSP.peer','Org2MSP.peer')"`即可

### 2.升级链码
外部链码改变之后，需要重新生成镜像并启动链码pod，然后在审批和提交链码时，修改`version`，`sequence`在原有基础上+1，其他和第一次审批和提交链码一样。

## 参考链接和源码：
- [Hyperledger Fabric 2.0外部链码实战【Kubernetes】](http://blog.hubwiz.com/2020/03/12/fabric-2-external-chaincode/)
- [源码参考](https://github.com/vanitas92/fabric-external-chaincodes)
- [官方文档](https://hyperledger-fabric.readthedocs.io/en/release-2.0/deploy_chaincode.html)
