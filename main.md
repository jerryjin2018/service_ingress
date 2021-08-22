# Check Service and Ingress in EKS  

在使用EKS/Kubernets的过程，暴露我们的服务(各种类型的协议),经常会碰到Service和Ingress。那细节上它们具体是如何实现的了？对一些应用的某些部分（如前端），可能希望将其暴露给 Kubernetes 集群外部的IP地址。Kubernetes ServiceTypes允许指定你所需要的Service类型，默认是ClusterIP。
ServiceTypes的取值以及行为如下：

**ClusterIP:** 通过集群的内部IP暴露服务，选择该值时服务只能够在集群内部访问。 这也是默认的ServiceType。  
**NodePort:** 通过每个节点上的IP和静态端口（NodePort）暴露服务。 NodePort服务会路由到自动创建的ClusterIP服务。 通过请求<节点 IP>:<节点端口>，你可以从集群的外部访问一个NodePort服务.  
**LoadBalancer:** 使用云提供商的负载均衡器向外部暴露服务。 外部负载均衡器可以将流量路由到自动创建的NodePort服务和 ClusterIP 服务上。  

ExternalName：通过返回CNAME和对应值，可以将服务映射到externalName字段的内容（例如，foo.bar.example.com）。 无需创建任何类型代理。  
说明：你需要使用kube-dns 1.7及以上版本或者 oreDNS 0.0.8 及以上版本才能使用 ExternalName 类型。  
另外，你也可以使用 Ingress 来暴露自己的服务。 Ingress 不是一种服务类型，但它充当集群的入口点。 它可以将路由规则整合到一个资源中，因为它可以在同一IP地址下公开多个服务。  

在探索Service and Ingress之前，让我们先温顾一下ISO OSI 7层模型和TCP/IP 4层模型。另外还有就是Linux的network stack以及iptables。因为这些都是我们来探索Service和Ingress的前提知识。

## 1. Prepare the environment
#### 1.1) check ISO OSI 7 layers模型，TCP/IP 4 layers模型 和 iptalbes的文档
a)ISO OSI 7 layers models
[https://www.cisco.com/c/dam/global/fi_fi/assets/docs/SMB_University_120307_Networking_Fundamentals.pdf](https://www.cisco.com/c/dam/global/fi_fi/assets/docs/SMB_University_120307_Networking_Fundamentals.pdf)   
[https://en.wikipedia.org/wiki/OSI_model](https://en.wikipedia.org/wiki/OSI_model)

b)TCP/IP 4 layers models  
[https://www.cisco.com/E-Learning/bulk/public/tac/cim/cib/using_cisco_ios_software/linked/tcpip.htm](https://www.cisco.com/E-Learning/bulk/public/tac/cim/cib/using_cisco_ios_software/linked/tcpip.htm)  
[https://www.cisco.com/c/en/us/support/docs/ip/routing-information-protocol-rip/13769-5.html](https://www.cisco.com/c/en/us/support/docs/ip/routing-information-protocol-rip/13769-5.html)  

c.iptables  
[https://www.netfilter.org/](https://www.netfilter.org/)  
[https://www.netfilter.org/projects/iptables/index.html](https://www.netfilter.org/projects/iptables/index.html)  
[https://www.zsythink.net/archives/tag/iptables/](https://www.zsythink.net/archives/tag/iptables/)  
  

#### 1.2) create test EKS cluster  
使用如下的yaml文件，通过eksctl创建EKS集群  
eksgo05-cluster.yaml
```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksgo05
  region: cn-northwest-1 
  version: "1.19"

managedNodeGroups:
  - name: ng-eksgo05-01
    instanceType: t3.xlarge
    instanceName: ng-eksgo05
    desiredCapacity: 3
    minSize: 1
    maxSize: 8 
    volumeSize: 100
    ssh:
      publicKeyName: your_key_pair
      allow: true
      enableSsm: true 
    iam:
      withAddonPolicies:
        ebs: true
        fsx: true
        efs: true
```

## 2. check iptables 4 tables (filter,nat,raw, mangle) on one of worker node
我们会发现每个worker node
#### 2.1) filter table
```
[root@ip-192-168-27-117 ~]# iptables -t filter -vnL | grep ^Chain
Chain INPUT (policy ACCEPT 7168 packets, 1482K bytes)
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
Chain OUTPUT (policy ACCEPT 7055 packets, 722K bytes)
Chain DOCKER (0 references)
Chain DOCKER-ISOLATION-STAGE-1 (0 references)
Chain DOCKER-ISOLATION-STAGE-2 (0 references)
Chain KUBE-EXTERNAL-SERVICES (1 references)
Chain KUBE-FIREWALL (2 references)
Chain KUBE-FORWARD (1 references)
Chain KUBE-KUBELET-CANARY (0 references)
Chain KUBE-PROXY-CANARY (0 references)
Chain KUBE-SERVICES (3 references)
[root@ip-192-168-27-117 ~]# iptables -t filter -vnL | grep ^Chain | wc -l
12
```
#### 2.2) nat table
[root@ip-192-168-27-117 ~]# iptables -t nat -vnL | grep ^Chain 
```
Chain PREROUTING (policy ACCEPT 15 packets, 880 bytes)
Chain INPUT (policy ACCEPT 15 packets, 880 bytes)
Chain OUTPUT (policy ACCEPT 975 packets, 61801 bytes)
Chain POSTROUTING (policy ACCEPT 224 packets, 16037 bytes)
Chain AWS-SNAT-CHAIN-0 (1 references)
Chain AWS-SNAT-CHAIN-1 (1 references)
Chain DOCKER (0 references)
Chain KUBE-KUBELET-CANARY (0 references)
Chain KUBE-MARK-DROP (0 references)
Chain KUBE-MARK-MASQ (12 references)
Chain KUBE-NODEPORTS (1 references)
Chain KUBE-POSTROUTING (1 references)
Chain KUBE-PROXY-CANARY (0 references)
Chain KUBE-SEP-37LH6U3HQBURX6XM (1 references)
Chain KUBE-SEP-AK3FEQAJMIBNMIHI (1 references)
Chain KUBE-SEP-C3ZAOHB5D5ADMT4S (1 references)
Chain KUBE-SEP-CC7K7QFF24BHFFIL (1 references)
Chain KUBE-SEP-G3UEEUSSNLD2VBOT (1 references)
Chain KUBE-SEP-GMOY7AGLKA4JLDBZ (1 references)
Chain KUBE-SEP-IGS4Q7GX7RBS5WTI (1 references)
Chain KUBE-SEP-JP4JNXIBO2FIEGVF (1 references)
Chain KUBE-SEP-RV2UIE37VZRNCGHI (1 references)
Chain KUBE-SEP-UWSOO7WHHDYD3OYA (1 references)
Chain KUBE-SEP-V47ZUOAGKF5Z7WLU (1 references)
Chain KUBE-SEP-ZUTEKL2NHHI2R3ED (1 references)
Chain KUBE-SERVICES (2 references)
Chain KUBE-SVC-ERIFXISQEP7F7OF4 (1 references)
Chain KUBE-SVC-JLDWYN2OQOZJWGQM (1 references)
Chain KUBE-SVC-NPX46M4PTMTKRN6Y (1 references)
Chain KUBE-SVC-TCOU7JCQXEZGVUNU (1 references)
Chain KUBE-SVC-WVB4SFQYY2BFP54D (1 references)
Chain KUBE-SVC-XS62VUIMGR5RELHB (1 references)
Chain KUBE-SVC-ZUD4L6KQKCHD52W4 (1 references)
[root@ip-192-168-27-117 ~]# iptables -t nat -vnL | grep ^Chain | wc -l
33
```
#### 2.3) raw table
```
[root@ip-192-168-27-117 ~]# iptables -t raw -vnL | grep ^Chain 
Chain PREROUTING (policy ACCEPT 1239K packets, 267M bytes)
Chain OUTPUT (policy ACCEPT 1245K packets, 128M bytes)
[root@ip-192-168-27-117 ~]# iptables -t raw -vnL | grep ^Chain | wc -l
2
```
#### 2.4) mangle table
```
[root@ip-192-168-27-117 ~]# iptables -t mangle -vnL | grep ^Chain 
Chain PREROUTING (policy ACCEPT 14M packets, 3670M bytes)
Chain INPUT (policy ACCEPT 13M packets, 2931M bytes)
Chain FORWARD (policy ACCEPT 1644K packets, 739M bytes)
Chain OUTPUT (policy ACCEPT 13M packets, 1319M bytes)
Chain POSTROUTING (policy ACCEPT 15M packets, 2058M bytes)
Chain KUBE-KUBELET-CANARY (0 references)
Chain KUBE-PROXY-CANARY (0 references)
[root@ip-192-168-27-117 ~]# iptables -t mangle -vnL | grep ^Chain | wc -l
7
```
#### 2.5) add the another node to node group
a)检查node group中节点的数量(3)
```
[ec2-user@ip-172-31-1-111 ~]$ kubectl get nodes -o wide
NAME                                                STATUS   ROLES    AGE    VERSION              INTERNAL-IP      EXTERNAL-IP      OS-IMAGE         KERNEL-VERSION                CONTAINER-RUNTIME
ip-192-168-27-117.cn-northwest-1.compute.internal   Ready    <none>   93d    v1.19.6-eks-49a6c0   192.168.27.117   52.83.21.129     Amazon Linux 2   5.4.129-63.229.amzn2.x86_64   docker://20.10.4
ip-192-168-38-4.cn-northwest-1.compute.internal     Ready    <none>   148d   v1.19.6-eks-49a6c0   192.168.38.4     52.82.124.129    Amazon Linux 2   5.4.129-63.229.amzn2.x86_64   docker://20.10.4
ip-192-168-84-63.cn-northwest-1.compute.internal    Ready    <none>   148d   v1.19.6-eks-49a6c0   192.168.84.63    161.189.103.25   Amazon Linux 2   5.4.129-63.229.amzn2.x86_64   docker://20.10.4
```
a)向node group中增加节点
```
[ec2-user@ip-172-31-1-111 ~]$ aws eks list-clusters 
{
    "clusters": [
        "eksgo03",
        "eksgo05",
        "KubeflowOnEKS",
        "cicdeks-cluster-d4aC"
    ]
}
aws eks list-nodegroups  --cluster-name eksgo05
{
    "nodegroups": [
        "ng-eksgo05-01"
    ]
}
[ec2-user@ip-172-31-1-111 ~]$ aws eks describe-nodegroup  --cluster-name eksgo05 --nodegroup-name ng-eksgo05-01 | head -n 20
{
    "nodegroup": {
        "nodegroupName": "ng-eksgo05-01",
        "nodegroupArn": "arn:aws-cn:eks:cn-northwest-1:**********:nodegroup/eksgo05/ng-eksgo05-01/babc2709-5bd0-719e-e7f5-7b2ca4dd9555",
        "clusterName": "eksgo05",
        "version": "1.19",
        "releaseVersion": "1.19.6-20210310",
        "createdAt": "2021-03-20T05:19:15.530000+00:00",
        "modifiedAt": "2021-08-15T14:19:24.690000+00:00",
        "status": "ACTIVE",
        "capacityType": "ON_DEMAND",
        "scalingConfig": {
            "minSize": 1,
            "maxSize": 8,
            "desiredSize": 3
        },
        "instanceTypes": [
            "t3.xlarge"
        ],
        "subnets": [
[ec2-user@ip-172-31-1-111 ~]$ aws eks describe-nodegroup  --cluster-name eksgo05 --nodegroup-name ng-eksgo05-01 | jq .nodegroup.scalingConfig
{
  "minSize": 1,
  "maxSize": 8,
  "desiredSize": 3
}
```    

```
[ec2-user@ip-172-31-1-111 ~]$ aws eks update-nodegroup-config --cluster-name eksgo05 --nodegroup-name ng-eksgo05-01 --scaling-config minSize=1,maxSize=8,desiredSize=4
{
    "update": {
        "id": "6b56dd21-5615-39d2-a504-86aeaf2c64a1",
        "status": "InProgress",
        "type": "ConfigUpdate",
        "params": [
            {
                "type": "MinSize",
                "value": "1"
            },
            {
                "type": "MaxSize",
                "value": "8"
            },
            {
                "type": "DesiredSize",
                "value": "4"
            }
        ],
        "createdAt": "2021-08-15T14:32:02.243000+00:00",
        "errors": []
    }
}
[ec2-user@ip-172-31-1-111 ~]$ aws eks describe-nodegroup  --cluster-name eksgo05 --nodegroup-name ng-eksgo05-01 | jq .nodegroup.scalingConfig
{
  "minSize": 1,
  "maxSize": 8,
  "desiredSize": 4
} 
[ec2-user@ip-172-31-1-111 ~]$ kubectl get nodes -o wide
NAME                                                STATUS   ROLES    AGE    VERSION              INTERNAL-IP      EXTERNAL-IP      OS-IMAGE         KERNEL-VERSION                CONTAINER-RUNTIME
ip-192-168-27-117.cn-northwest-1.compute.internal   Ready    <none>   93d    v1.19.6-eks-49a6c0   192.168.27.117   52.83.21.129     Amazon Linux 2   5.4.129-63.229.amzn2.x86_64   docker://20.10.4
ip-192-168-38-4.cn-northwest-1.compute.internal     Ready    <none>   148d   v1.19.6-eks-49a6c0   192.168.38.4     52.82.124.129    Amazon Linux 2   5.4.129-63.229.amzn2.x86_64   docker://20.10.4
ip-192-168-84-63.cn-northwest-1.compute.internal    Ready    <none>   148d   v1.19.6-eks-49a6c0   192.168.84.63    161.189.103.25   Amazon Linux 2   5.4.129-63.229.amzn2.x86_64   docker://20.10.4
ip-192-168-87-23.cn-northwest-1.compute.internal    Ready    <none>   28s    v1.19.6-eks-49a6c0   192.168.87.23    52.82.32.182     Amazon Linux 2   5.4.95-42.163.amzn2.x86_64    docker://19.3.13
```   
通过新增一个worker node后，我们会发现iptables的4张tables中链，包括自定义链的数量和名称都是一样的.同版本的EKS中4张tables中链，包括自定义链,有部分名称是相同的

## 3. 检查Cluster IP

### 3.1) create test service(ClusterIP) and test pod  
```
---
apiVersion: v1
kind: Namespace
metadata:
  name: net-test
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: net-test
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-net
  template:
    metadata:
      labels:
        app: nginx-net
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-clusterip
  namespace: net-test
spec:
  type: ClusterIP
  selector:
    app: nginx-net
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
```

### 3.2) 检查Service -- Cluster IP和对应的nat表中的链
nat表中的链有iptable默认定义的链(PREROUTING,INPUT,OUTPUT,POSTROUTING)  
在测试机器上执行kubectl,我们可以看到我们刚刚创建的名为nginx-service 的service，目前为ClusterIP的类型(TYPE)，而且它的CLUSTER-IP地址为10.100.246.132，端口为80/TCP，也就是通过 nginx-service(10.100.246.132)的80端口可以访问到target(pod的80端口）
```
[ec2-user@ip-172-31-1-111 ~]$ kubectl get service -n net-test -o wide
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE   SELECTOR
nginx-service   ClusterIP   10.100.246.132   <none>        80/TCP    23h   app=nginx-net
```
在worker node上查看iptables 规则，其实在4个worker node上的iptables -t nat -vnL结果基本一样，只是Chain KUBE-SERVICES中具体的规则的顺序不一样
```
[root@ip-192-168-27-117 ~]# iptables -t nat -vnL
Chain PREROUTING (policy ACCEPT 180 packets, 9568 bytes)
 pkts bytes target     prot opt in     out     source               destination         
 321K   19M KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */

Chain INPUT (policy ACCEPT 180 packets, 9568 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 2795 packets, 183K bytes)
 pkts bytes target     prot opt in     out     source               destination         
1262K   80M KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */

Chain POSTROUTING (policy ACCEPT 1216 packets, 87087 bytes)
 pkts bytes target     prot opt in     out     source               destination         
1412K   89M KUBE-POSTROUTING  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes postrouting rules */
1262K   80M AWS-SNAT-CHAIN-0  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* AWS SNAT CHAIN */

Chain AWS-SNAT-CHAIN-0 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
1129K   69M AWS-SNAT-CHAIN-1  all  --  *      *       0.0.0.0/0           !192.168.0.0/16       /* AWS SNAT CHAIN */

Chain AWS-SNAT-CHAIN-1 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
 962K   59M SNAT       all  --  *      !vlan+  0.0.0.0/0            0.0.0.0/0            /* AWS, SNAT */ ADDRTYPE match dst-type !LOCAL to:192.168.27.117 random-fully

Chain DOCKER (0 references)
 pkts bytes target     prot opt in     out     source               destination         

Chain KUBE-KUBELET-CANARY (0 references)
 pkts bytes target     prot opt in     out     source               destination         

Chain KUBE-MARK-DROP (0 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            MARK or 0x8000

Chain KUBE-MARK-MASQ (12 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            MARK or 0x4000

Chain KUBE-NODEPORTS (1 references)
 pkts bytes target     prot opt in     out     source               destination         

Chain KUBE-POSTROUTING (1 references)
 pkts bytes target     prot opt in     out     source               destination         
 2795  183K RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            mark match ! 0x4000/0x4000
    0     0 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            MARK xor 0x4000
    0     0 MASQUERADE  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service traffic requiring SNAT */ random-fully

Chain KUBE-PROXY-CANARY (0 references)
 pkts bytes target     prot opt in     out     source               destination         

Chain KUBE-SEP-37LH6U3HQBURX6XM (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       192.168.24.176       0.0.0.0/0            /* net-test/nginx-service */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */ tcp to:192.168.24.176:80

Chain KUBE-SEP-AK3FEQAJMIBNMIHI (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       192.168.153.35       0.0.0.0/0            /* default/kubernetes:https */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/kubernetes:https */ tcp to:192.168.153.35:443

Chain KUBE-SEP-C3ZAOHB5D5ADMT4S (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       192.168.41.64        0.0.0.0/0            /* kube-system/kube-dns:dns-tcp */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns-tcp */ tcp to:192.168.41.64:53

Chain KUBE-SEP-CC7K7QFF24BHFFIL (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       192.168.74.98        0.0.0.0/0            /* kube-system/kube-dns:dns-tcp */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns-tcp */ tcp to:192.168.74.98:53

Chain KUBE-SEP-G3UEEUSSNLD2VBOT (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       192.168.108.110      0.0.0.0/0            /* default/kubernetes:https */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/kubernetes:https */ tcp to:192.168.108.110:443

Chain KUBE-SEP-GMOY7AGLKA4JLDBZ (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       192.168.86.49        0.0.0.0/0            /* net-test/nginx-service */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */ tcp to:192.168.86.49:80

Chain KUBE-SEP-IGS4Q7GX7RBS5WTI (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       192.168.50.114       0.0.0.0/0            /* net-test/nginx-service */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */ tcp to:192.168.50.114:80

Chain KUBE-SEP-JP4JNXIBO2FIEGVF (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       192.168.34.33        0.0.0.0/0            /* cert-manager/cert-manager */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cert-manager/cert-manager */ tcp to:192.168.34.33:9402

Chain KUBE-SEP-RV2UIE37VZRNCGHI (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       192.168.74.98        0.0.0.0/0            /* kube-system/kube-dns:dns */
    0     0 DNAT       udp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns */ udp to:192.168.74.98:53

Chain KUBE-SEP-UWSOO7WHHDYD3OYA (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       192.168.34.70        0.0.0.0/0            /* kube-system/aws-load-balancer-webhook-service */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/aws-load-balancer-webhook-service */ tcp to:192.168.34.70:9443

Chain KUBE-SEP-V47ZUOAGKF5Z7WLU (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       192.168.41.64        0.0.0.0/0            /* kube-system/kube-dns:dns */
    0     0 DNAT       udp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns */ udp to:192.168.41.64:53

Chain KUBE-SEP-ZUTEKL2NHHI2R3ED (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       192.168.69.246       0.0.0.0/0            /* cert-manager/cert-manager-webhook:https */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cert-manager/cert-manager-webhook:https */ tcp to:192.168.69.246:10250

Chain KUBE-SERVICES (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SVC-WVB4SFQYY2BFP54D  tcp  --  *      *       0.0.0.0/0            10.100.235.233       /* cert-manager/cert-manager cluster IP */ tcp dpt:9402
    0     0 KUBE-SVC-NPX46M4PTMTKRN6Y  tcp  --  *      *       0.0.0.0/0            10.100.0.1           /* default/kubernetes:https cluster IP */ tcp dpt:443
    0     0 KUBE-SVC-XS62VUIMGR5RELHB  tcp  --  *      *       0.0.0.0/0            10.100.32.228        /* kube-system/aws-load-balancer-webhook-service cluster IP */ tcp dpt:443
    0     0 KUBE-SVC-ERIFXISQEP7F7OF4  tcp  --  *      *       0.0.0.0/0            10.100.0.10          /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:53
    0     0 KUBE-SVC-ZUD4L6KQKCHD52W4  tcp  --  *      *       0.0.0.0/0            10.100.105.44        /* cert-manager/cert-manager-webhook:https cluster IP */ tcp dpt:443
    0     0 KUBE-SVC-TCOU7JCQXEZGVUNU  udp  --  *      *       0.0.0.0/0            10.100.0.10          /* kube-system/kube-dns:dns cluster IP */ udp dpt:53
    0     0 KUBE-SVC-JLDWYN2OQOZJWGQM  tcp  --  *      *       0.0.0.0/0            10.100.246.132       /* net-test/nginx-service cluster IP */ tcp dpt:80
  454 26008 KUBE-NODEPORTS  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL

Chain KUBE-SVC-ERIFXISQEP7F7OF4 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SEP-C3ZAOHB5D5ADMT4S  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns-tcp */ statistic mode random probability 0.50000000000
    0     0 KUBE-SEP-CC7K7QFF24BHFFIL  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns-tcp */

Chain KUBE-SVC-JLDWYN2OQOZJWGQM (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SEP-37LH6U3HQBURX6XM  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */ statistic mode random probability 0.33333333349
    0     0 KUBE-SEP-IGS4Q7GX7RBS5WTI  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */ statistic mode random probability 0.50000000000
    0     0 KUBE-SEP-GMOY7AGLKA4JLDBZ  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */

Chain KUBE-SVC-NPX46M4PTMTKRN6Y (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SEP-G3UEEUSSNLD2VBOT  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/kubernetes:https */ statistic mode random probability 0.50000000000
    0     0 KUBE-SEP-AK3FEQAJMIBNMIHI  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/kubernetes:https */

Chain KUBE-SVC-TCOU7JCQXEZGVUNU (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SEP-V47ZUOAGKF5Z7WLU  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns */ statistic mode random probability 0.50000000000
    0     0 KUBE-SEP-RV2UIE37VZRNCGHI  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns */

Chain KUBE-SVC-WVB4SFQYY2BFP54D (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SEP-JP4JNXIBO2FIEGVF  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cert-manager/cert-manager */

Chain KUBE-SVC-XS62VUIMGR5RELHB (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SEP-UWSOO7WHHDYD3OYA  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/aws-load-balancer-webhook-service */

Chain KUBE-SVC-ZUD4L6KQKCHD52W4 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SEP-ZUTEKL2NHHI2R3ED  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cert-manager/cert-manager-webhook:https */
```

nat表中的链有iptable默认定义的链(PREROUTING,INPUT,OUTPUT,POSTROUTING)，来检查CLUSTER IP -- 10.100.246.132。我们会发现10.100.246.132 在自定义链 KUBE-SERVICES中定义一条规则，具体如下，并且应用的target/action是KUBE-SVC-JLDWYN2OQOZJWGQM
```
[root@ip-192-168-27-117 ~]# iptables -t nat -vnL  | grep -i "10.100.246.132" -B 8
Chain KUBE-SERVICES (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SVC-WVB4SFQYY2BFP54D  tcp  --  *      *       0.0.0.0/0            10.100.235.233       /* cert-manager/cert-manager cluster IP */ tcp dpt:9402
    0     0 KUBE-SVC-NPX46M4PTMTKRN6Y  tcp  --  *      *       0.0.0.0/0            10.100.0.1           /* default/kubernetes:https cluster IP */ tcp dpt:443
    0     0 KUBE-SVC-XS62VUIMGR5RELHB  tcp  --  *      *       0.0.0.0/0            10.100.32.228        /* kube-system/aws-load-balancer-webhook-service cluster IP */ tcp dpt:443
    0     0 KUBE-SVC-ERIFXISQEP7F7OF4  tcp  --  *      *       0.0.0.0/0            10.100.0.10          /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:53
    0     0 KUBE-SVC-ZUD4L6KQKCHD52W4  tcp  --  *      *       0.0.0.0/0            10.100.105.44        /* cert-manager/cert-manager-webhook:https cluster IP */ tcp dpt:443
    0     0 KUBE-SVC-TCOU7JCQXEZGVUNU  udp  --  *      *       0.0.0.0/0            10.100.0.10          /* kube-system/kube-dns:dns cluster IP */ udp dpt:53
    0     0 KUBE-SVC-JLDWYN2OQOZJWGQM  tcp  --  *      *       0.0.0.0/0            10.100.246.132       /* net-test/nginx-service cluster IP */ tcp dpt:80
```

那我们来检查自定义链 KUBE-SERVICES 和 KUBE-SVC-JLDWYN2OQOZJWGQM
自定义链 KUBE-SERVICES 主要在 nat 表的 PREROUTING 和 OUTPUT 链中出现,
>		PREROUTING
>		OUTPUT
也就是说，nat 表的 PREROUTING 和 OUTPUT 链的 target/action 调用了 KUBE-SERVICES 自定义链
>		KUBE-SERVICES
而 KUBE-SERVICES 自定义链调用了 KUBE-SVC-JLDWYN2OQOZJWGQM
>		KUBE-SVC-JLDWYN2OQOZJWGQM
而自定义链 KUBE-SVC-JLDWYN2OQOZJWGQM 又引用了如下3个自定义链  
>		1) KUBE-SEP-37LH6U3HQBURX6XM  
>		2) KUBE-SEP-IGS4Q7GX7RBS5WTI  
>		3) KUBE-SEP-GMOY7AGLKA4JLDBZ  

1)自定义链 KUBE-SEP-37LH6U3HQBURX6XM 又引用了如下1个自定义链和target/action DNAT
>		KUBE-MARK-MASQ
>		DNAT

自定义链 KUBE-MARK-MASQ 的target/action MARK
>		MARK

2)自定义链 KUBE-SEP-IGS4Q7GX7RBS5WTI 又引用了如下1个自定义链和target/action DNAT
>		KUBE-MARK-MASQ
>		DNAT

自定义链 KUBE-MARK-MASQ 的target/action MARK
>		MARK

3)自定义链 KUBE-SEP-GMOY7AGLKA4JLDBZ 又引用了如下1个自定义链和target/action DNAT
>		KUBE-MARK-MASQ
>		DNAT

自定义链 KUBE-MARK-MASQ 的target/action MARK
>		MARK


```
[root@ip-192-168-27-117 ~]# iptables -t nat -vnL  | grep -i "KUBE-SERVICES" -B 5
Chain PREROUTING (policy ACCEPT 361 packets, 19212 bytes)
 pkts bytes target     prot opt in     out     source               destination         
 322K   19M KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
--
Chain INPUT (policy ACCEPT 361 packets, 19212 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 5845 packets, 383K bytes)
 pkts bytes target     prot opt in     out     source               destination         
1266K   80M KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
--
Chain KUBE-SEP-ZUTEKL2NHHI2R3ED (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       192.168.69.246       0.0.0.0/0            /* cert-manager/cert-manager-webhook:https */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cert-manager/cert-manager-webhook:https */ tcp to:192.168.69.246:10250

Chain KUBE-SERVICES (2 references)
```

```
[root@ip-192-168-27-117 ~]# iptables -t nat -vnL  | grep -i "Chain KUBE-SVC-JLDWYN2OQOZJWGQM " -A 5
Chain KUBE-SVC-JLDWYN2OQOZJWGQM (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SEP-37LH6U3HQBURX6XM  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */ statistic mode random probability 0.33333333349
    0     0 KUBE-SEP-IGS4Q7GX7RBS5WTI  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */ statistic mode random probability 0.50000000000
    0     0 KUBE-SEP-GMOY7AGLKA4JLDBZ  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */
```

#### ####### 插播一下 此处需要介绍一下 iptables statistic 模块:  
[https://ipset.netfilter.org/iptables-extensions.man.html](https://ipset.netfilter.org/iptables-extensions.man.html)  
```
statistic
This module matches packets based on some statistic condition. It supports two distinct modes settable with the --mode option.
Supported options:

--mode mode
Set the matching mode of the matching rule, supported modes are random and nth.
[!] --probability p
Set the probability for a packet to be randomly matched. It only works with the random mode. p must be within 0.0 and 1.0. The supported granularity is in 1/2147483648th increments.
[!] --every n
Match one packet every nth packet. It works only with the nth mode (see also the --packet option).
--packet p
Set the initial counter value (0 <= p <= n-1, default 0) for the nth mode.
```
很显然，通过 statistic 模块达到负载均衡的效果，但是这里是3个worker node，如果是4个或是10个pod，情况会是如何的了？那我们先将pod扩展到4个。

```
[ec2-user@ip-172-31-1-111 ~]$ kubectl get pod  -n net-test -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE                                                NOMINATED NODE   READINESS GATES
nginx-deployment-868b96499f-fq8t5   1/1     Running   0          45h   192.168.24.176   ip-192-168-27-117.cn-northwest-1.compute.internal   <none>           <none>
nginx-deployment-868b96499f-j7dwm   1/1     Running   0          45h   192.168.50.114   ip-192-168-38-4.cn-northwest-1.compute.internal     <none>           <none>
nginx-deployment-868b96499f-lkz8b   1/1     Running   0          45h   192.168.86.49    ip-192-168-84-63.cn-northwest-1.compute.internal    <none>           <none>
test-server                         1/1     Running   0          45h   192.168.8.217    ip-192-168-27-117.cn-northwest-1.compute.internal   <none>           <none>
[ec2-user@ip-172-31-1-111 ~]$ kubectl get nodes -o wide
NAME                                                STATUS   ROLES    AGE    VERSION              INTERNAL-IP      EXTERNAL-IP      OS-IMAGE         KERNEL-VERSION                CONTAINER-RUNTIME
ip-192-168-27-117.cn-northwest-1.compute.internal   Ready    <none>   94d    v1.19.6-eks-49a6c0   192.168.27.117   52.83.21.129     Amazon Linux 2   5.4.129-63.229.amzn2.x86_64   docker://20.10.4
ip-192-168-38-4.cn-northwest-1.compute.internal     Ready    <none>   149d   v1.19.6-eks-49a6c0   192.168.38.4     52.82.124.129    Amazon Linux 2   5.4.129-63.229.amzn2.x86_64   docker://20.10.4
ip-192-168-84-63.cn-northwest-1.compute.internal    Ready    <none>   149d   v1.19.6-eks-49a6c0   192.168.84.63    161.189.103.25   Amazon Linux 2   5.4.129-63.229.amzn2.x86_64   docker://20.10.4
ip-192-168-87-23.cn-northwest-1.compute.internal    Ready    <none>   22h    v1.19.6-eks-49a6c0   192.168.87.23    52.82.32.182     Amazon Linux 2   5.4.129-63.229.amzn2.x86_64   docker://20.10.4
[ec2-user@ip-172-31-1-111 ~]$ 
[ec2-user@ip-172-31-1-111 ~]$ kubectl get deployments.apps/nginx-deployment -n net-test 
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           45h
[ec2-user@ip-172-31-1-111 ~]$ 

### 将deployments中pod副本的数量由 3 个扩展到 4 
[ec2-user@ip-172-31-1-111 ~]$ kubectl scale deployments.apps/nginx-deployment --replicas=4 -n net-test 
deployment.apps/nginx-deployment scaled


[ec2-user@ip-172-31-1-111 ~]$ kubectl get deployments.apps/nginx-deployment -n net-test 
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   4/4     4            4           45h
[ec2-user@ip-172-31-1-111 ~]$ kubectl get pod  -n net-test -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE                                                NOMINATED NODE   READINESS GATES
nginx-deployment-868b96499f-cjlwc   1/1     Running   0          20s   192.168.82.60    ip-192-168-87-23.cn-northwest-1.compute.internal    <none>           <none>
nginx-deployment-868b96499f-fq8t5   1/1     Running   0          45h   192.168.24.176   ip-192-168-27-117.cn-northwest-1.compute.internal   <none>           <none>
nginx-deployment-868b96499f-j7dwm   1/1     Running   0          45h   192.168.50.114   ip-192-168-38-4.cn-northwest-1.compute.internal     <none>           <none>
nginx-deployment-868b96499f-lkz8b   1/1     Running   0          45h   192.168.86.49    ip-192-168-84-63.cn-northwest-1.compute.internal    <none>           <none>
test-server                         1/1     Running   0          45h   192.168.8.217    ip-192-168-27-117.cn-northwest-1.compute.internal   <none>           <none>
```
再来看看iptables 自定义链 KUBE-SVC-JLDWYN2OQOZJWGQM。正如你所见到的，自定义链 KUBE-SVC-JLDWYN2OQOZJWGQM 的条目发生了变化，它根据增加的pod，而调整了statistic模块 mode random probability后面的值。
```
[root@ip-192-168-27-117 ~]# iptables -t nat -vnL  | grep -i "Chain KUBE-SVC-JLDWYN2OQOZJWGQM " -A 5
Chain KUBE-SVC-JLDWYN2OQOZJWGQM (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SEP-37LH6U3HQBURX6XM  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */ statistic mode random probability 0.25000000000
    0     0 KUBE-SEP-IGS4Q7GX7RBS5WTI  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */ statistic mode random probability 0.33333333349
    0     0 KUBE-SEP-C234U7CCB7GHKOZ6  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */ statistic mode random probability 0.50000000000
    0     0 KUBE-SEP-GMOY7AGLKA4JLDBZ  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */
```
Pod的扩展到10个后的情况
```
[root@ip-192-168-27-117 ~]# iptables -t nat -vnL  | grep -i "Chain KUBE-SVC-JLDWYN2OQOZJWGQM " -A 10
Chain KUBE-SVC-JLDWYN2OQOZJWGQM (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SEP-UQ72TRR5EWTR5KBK  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */ statistic mode random probability 0.10000000009
    0     0 KUBE-SEP-37LH6U3HQBURX6XM  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */ statistic mode random probability 0.11111111101
    0     0 KUBE-SEP-CMZXDP2AFR2KKK3W  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */ statistic mode random probability 0.12500000000
    0     0 KUBE-SEP-FEWA4SW2W3WQMCE7  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */ statistic mode random probability 0.14285714272
    0     0 KUBE-SEP-IGS4Q7GX7RBS5WTI  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */ statistic mode random probability 0.16666666651
    0     0 KUBE-SEP-GGNUNOJFOKAIE5V3  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */ statistic mode random probability 0.20000000019
    0     0 KUBE-SEP-C234U7CCB7GHKOZ6  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */ statistic mode random probability 0.25000000000
    0     0 KUBE-SEP-GMOY7AGLKA4JLDBZ  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */ statistic mode random probability 0.33333333349
    0     0 KUBE-SEP-NPXTA66666E7ZXBU  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */ statistic mode random probability 0.50000000000
```
总结下来，当你扩展pod副本的数量后，EKS/K8S 会在自定义链 KUBE-SVC-JLDWYN2OQOZJWGQM 中定义n条额外的自定义链，n的数量等于pod replicas的梳理，每一条自定义链后面的statistic模块 mode random probability后面的值的规律是:  
1/n  
1/(n-1)  
1/(n-2)  
...
1/3  
1/2

#### ####### 插播结束，继续分析自定义链  

```
[root@ip-192-168-27-117 ~]# iptables -t nat -vnL  | grep -i "Chain KUBE-SEP-37LH6U3HQBURX6XM" -A 4
Chain KUBE-SEP-37LH6U3HQBURX6XM (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       192.168.24.176       0.0.0.0/0            /* net-test/nginx-service */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */ tcp to:192.168.24.176:80
```

```
[root@ip-192-168-27-117 ~]# iptables -t nat -vnL  | grep -i "Chain KUBE-SEP-IGS4Q7GX7RBS5WTI" -A 4
Chain KUBE-SEP-IGS4Q7GX7RBS5WTI (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       192.168.50.114       0.0.0.0/0            /* net-test/nginx-service */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */ tcp to:192.168.50.114:80
```

```
[root@ip-192-168-27-117 ~]# iptables -t nat -vnL  | grep -i "Chain KUBE-SEP-GMOY7AGLKA4JLDBZ" -A 4
Chain KUBE-SEP-GMOY7AGLKA4JLDBZ (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       192.168.86.49        0.0.0.0/0            /* net-test/nginx-service */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service */ tcp to:192.168.86.49:80
```

```
[root@ip-192-168-27-117 ~]# iptables -t nat -vnL  | grep -i "Chain KUBE-MARK-MASQ" -A 2
Chain KUBE-MARK-MASQ (19 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            MARK or 0x4000
```
那DNAT和MARK target/action意义是什么了？如下的官方介绍。DNAT就是对数据包中的目的地址做NAT(地址转换),而MARK则是给指定的数据包做标识
[https://ipset.netfilter.org/iptables-extensions.man.html](https://ipset.netfilter.org/iptables-extensions.man.html)  

```
DNAT
A virtual state, matching if the original destination differs from the reply source.
mark
This module matches the netfilter mark field associated with a packet (which can be set using the MARK target below).
[!] --mark value[/mask]
Matches packets with the given unsigned mark value (if a mask is specified, this is logically ANDed with the mask before the comparison).
```

## 4. 检查Node Port
### 4.1) create test service(NodePort) and test pod  
```
---
apiVersion: v1
kind: Namespace
metadata:
  name: net-test
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: net-test
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-net
  template:
    metadata:
      labels:
        app: nginx-net
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-nodeport
  namespace: net-test
spec:
  type: NodePort
  selector:
    app: nginx-net
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
```  

### 4.2) 检查Service -- Node Port和对应的nat表中的链
nat表中的链有iptable默认定义的链(PREROUTING,INPUT,OUTPUT,POSTROUTING)  
在测试机器上执行kubectl,我们可以看到我们刚刚创建的名为nginx-service-nodeport 的service，目前为NodePort的类型(TYPE)，而且它的CLUSTER-IP地址为10.100.246.132，对的它也有CLUSTER-IP，它的端口是8080:30559/TCP。也就是通过 nginx-service-nodeport(worker node)的31766端口可以访问到target(pod的80端口）
```
[ec2-user@ip-172-31-1-111 explore-network]$ kubectl get service -n net-test
NAME                     TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
nginx-service-nodeport   NodePort   10.100.118.236   <none>        8080:30559/TCP   12s
```
大家很容易想到使用Node Port，为了确保能访问到pod，一定是发生了nat(网络地址翻译)，所以我们来查看nat表。nat表中的链有iptable默认定义的链(PREROUTING,INPUT,OUTPUT,POSTROUTING)，逻辑上数据包进入到network stack，是先到PREROUTING链，再结合 iptables -t nat -vnL 输出中在4条链(PREROUTING,INPUT,OUTPUT,POSTROUTING)内容的观察，也会直接查看PREROUTING链的规则。如下：  

```
[root@ip-192-168-27-117 ~]# iptables -t nat -vnL | grep PREROUTING -A 3
Chain PREROUTING (policy ACCEPT 16 packets, 960 bytes)
 pkts bytes target     prot opt in     out     source               destination         
 328K   20M KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
```

PREROUTING链的target/action还是自定义链 KUBE-SERVICES。接下来还是查看自定义链 KUBE-SERVICES中定义一条规则，具体如下:

```
[root@ip-192-168-27-117 ~]# iptables -t nat -vnL | grep "Chain KUBE-SERVICES" -A 10
Chain KUBE-SERVICES (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SVC-WVB4SFQYY2BFP54D  tcp  --  *      *       0.0.0.0/0            10.100.235.233       /* cert-manager/cert-manager cluster IP */ tcp dpt:9402
    0     0 KUBE-SVC-NPX46M4PTMTKRN6Y  tcp  --  *      *       0.0.0.0/0            10.100.0.1           /* default/kubernetes:https cluster IP */ tcp dpt:443
    0     0 KUBE-SVC-XS62VUIMGR5RELHB  tcp  --  *      *       0.0.0.0/0            10.100.32.228        /* kube-system/aws-load-balancer-webhook-service cluster IP */ tcp dpt:443
    0     0 KUBE-SVC-ERIFXISQEP7F7OF4  tcp  --  *      *       0.0.0.0/0            10.100.0.10          /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:53
    0     0 KUBE-SVC-ZUD4L6KQKCHD52W4  tcp  --  *      *       0.0.0.0/0            10.100.105.44        /* cert-manager/cert-manager-webhook:https cluster IP */ tcp dpt:443
    0     0 KUBE-SVC-TCOU7JCQXEZGVUNU  udp  --  *      *       0.0.0.0/0            10.100.0.10          /* kube-system/kube-dns:dns cluster IP */ udp dpt:53
    0     0 KUBE-SVC-NUUJACZKM4EFI6UZ  tcp  --  *      *       0.0.0.0/0            10.100.118.236       /* net-test/nginx-service-nodeport cluster IP */ tcp dpt:8080
  113  6780 KUBE-NODEPORTS  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL
```
尽管其中有一个条规则，它的destination 是10.100.118.236，但是我们现在创建的service的类型是nodeport，也就意味一定是通过worker node的端口(port)进入到worker node，也就是一定通过了worker node的ip地址,不论是primary ip,还是secondary ip.
我们可以看到worker node的端口30559是被kube-proxy监听，而且是监听了worker node上所有的ip地址。所以进入到worker node的端口30559的数据包匹配的是 KUBE-SERVICES 中 KUBE-NODEPORTS 自定义链。

```
[root@ip-192-168-27-117 ~]# netstat -ntupl| head -n 2 && netstat -ntupl | grep -i 30559
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:30559           0.0.0.0:*               LISTEN      5848/kube-proxy  
```
那KUBE-NODEPORTS 自定义链包含的具体规则是什么了？如大家可以看到的，自定义链 KUBE-MARK-MASQ 对数据包mark。另外的自定义链 KUBE-SVC-NUUJACZKM4EFI6UZ ，让我们记住这个自定义链 KUBE-SVC-NUUJACZKM4EFI6UZ

```
[root@ip-192-168-27-117 ~]# iptables -t nat -vnL | grep "Chain KUBE-NODEPORTS" -A 4
Chain KUBE-NODEPORTS (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service-nodeport */ tcp dpt:30559
    0     0 KUBE-SVC-NUUJACZKM4EFI6UZ  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0    
       /* net-test/nginx-service-nodeport */ tcp dpt:30559
``` 

到这里，大家又看到如ClusterIP又做了负载均衡。

```
[root@ip-192-168-27-117 ~]# iptables -t nat -vnL | grep "Chain KUBE-SVC-NUUJACZKM4EFI6UZ" -A 5
Chain KUBE-SVC-NUUJACZKM4EFI6UZ (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SEP-ZYSM2J22H5HIHOCU  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service-nodeport */ statistic mode random probability 0.33333333349
    0     0 KUBE-SEP-LSO5PYSXOURJXISR  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service-nodeport */ statistic mode random probability 0.50000000000
    0     0 KUBE-SEP-32G3EIVK5YFXJQY7  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* net-test/nginx-service-nodeport */
```

我们来看看自定义链 KUBE-SERVICES 中destination 是10.100.118.236的规则, 它的target/action也是自定义链 KUBE-SVC-NUUJACZKM4EFI6UZ。所以在iptables 的规则的角度，NodePort变相的使用了ClusterIP(的规则)，而不是直接使用ClusterIP的ip地址。
```
[root@ip-192-168-27-117 ~]# iptables -t nat -vnL | grep "Chain KUBE-SERVICES" -A 10 | grep 10.100.118.236
    0     0 KUBE-SVC-NUUJACZKM4EFI6UZ  tcp  --  *      *       0.0.0.0/0            10.100.118.236       /* net-test/nginx-service-nodeport cluster IP */ tcp dpt:8080

```

## 5. 检查LoadBalancer
### 5.1) create test service(LoadBalancer) and test pod  

```
---
apiVersion: v1
kind: Namespace
metadata:
  name: net-test
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: net-test
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-net
  template:
    metadata:
      labels:
        app: nginx-net
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-lb
  namespace: net-test
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  type: LoadBalancer
  selector:
    app: nginx-net
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
```


### 5.2) 检查Service -- LoadBalancer(Instance tyep) 和对应的nat表中的链
在测试机器上执行kubectl,我们可以看到我们刚刚创建的名为nginx-service-lb 的service，目前为LoadBalancer的类型(TYPE)，而且它的CLUSTER-IP地址为10.100.91.22，对的它也有CLUSTER-IP，它的端口是8080:30607/TCP。也就是通过 nginx-service-lb(也就是AWS的NLB)将数据包分发到 worker node的30607端口可以访问到target(pod的80端口）

```
[ec2-user@ip-172-31-1-111 explore-network]$ kubectl get service -n net-test
NAME               TYPE           CLUSTER-IP     EXTERNAL-IP                                                                             PORT(S)          AGE
nginx-service-lb   LoadBalancer   10.100.91.22   a15b8d4be32604ad2b1e87dfd0c19b9f-5d2603cc314d2674.elb.cn-northwest-1.amazonaws.com.cn   8080:30607/TCP   4m23s
```

其实当数据包由AWS的NLB转发到worker node的30607端口，数据包处理的逻辑和过程和NodePort没有什么本质的差别。此处不再赘述。

### 5.3) 检查Service -- LoadBalancer(IP tyep) 
重新部署测试环境，此处省略创建AWS Load Balancer Controllerd的步骤，因为LoadBalancer(IP tyep)需要AWS Load Balancer Controllerd的支持。

```
---
apiVersion: v1
kind: Namespace
metadata:
  name: net-test
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: net-test
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-net
  template:
    metadata:
      labels:
        app: nginx-net
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-lb
  namespace: net-test
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb-ip
spec:
  type: LoadBalancer
  selector:
    app: nginx-net
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
```

```
[ec2-user@ip-172-31-1-111 ~]$ kubectl get service -n net-test
NAME               TYPE           CLUSTER-IP     EXTERNAL-IP                                                                            PORT(S)          AGE
nginx-service-lb   LoadBalancer   10.100.44.39   k8s-nettest-nginxser-d5319fcd93-4a1fae250b6ffc1e.elb.cn-northwest-1.amazonaws.com.cn   8080:32398/TCP   13h

[ec2-user@ip-172-31-1-111 ~]$ kubectl describe service -n net-test
Name:                     nginx-service-lb
Namespace:                net-test
Labels:                   <none>
Annotations:              service.beta.kubernetes.io/aws-load-balancer-type: nlb-ip
Selector:                 app=nginx-net
Type:                     LoadBalancer
IP:                       10.100.44.39
LoadBalancer Ingress:     k8s-nettest-nginxser-d5319fcd93-4a1fae250b6ffc1e.elb.cn-northwest-1.amazonaws.com.cn
Port:                     <unset>  8080/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  32398/TCP
Endpoints:                192.168.29.170:80,192.168.46.67:80,192.168.77.200:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```
 
在测试机器上执行kubectl,我们可以看到我们刚刚创建的名为nginx-service-lb 的service，目前为LoadBalancer的类型(TYPE)，而且它的CLUSTER-IP地址为10.100.44.39，它的端口是8080:32398/TCP。也就是通过 nginx-service-lb(也就是AWS的NLB)将数据包分发到 worker node的32398端口可以访问到target(pod的80端口），但是这里是直接将数据包发到pod的ip地址，因为此处EKS的service的类型为LoadBalancer(IP tyep)。而且pod也具有vpc cidr range内分配的IP 地址，这里没有iptables中的具体规则。这里可以通过查看 NLB ( k8s-nettest-nginxser-d5319fcd93-4a1fae250b6ffc1e.elb.cn-northwest-1.amazonaws.com.cn )的target group。

## 6. 检查Ingress
Ingress 公开了从集群外部到集群内服务的 HTTP 和 HTTPS 路由。 流量路由由 Ingress 资源上定义的规则控制。  
Ingress 规则   
每个 HTTP 规则都包含以下信息：  
**可选的 host:**  该规则适用于通过指定 IP 地址的所有入站 HTTP 通信。 如果提供了 host（例如 foo.bar.com），则 rules 适用于该 host。  
**路径列表 paths:** （例如，/testpath）,每个路径都有一个由 serviceName 和 servicePort 定义的关联后端。 在负载均衡器将流量定向到引用的服务之前，主机和路径都必须匹配传入请求的内容。  
**backend（后端）:** 是 Service 文档中所述的服务和端口名称的组合。 与规则的 host 和 path 匹配的对 Ingress 的 HTTP（和 HTTPS ）请求将发送到列出的 backend。  
通常在 Ingress 控制器中会配置 defaultBackend（默认后端），以服务于任何不符合规约中 path 的请求。  

### 6.1) create test ingress(LoadBalancer) and test pod  
```
---
apiVersion: v1
kind: Namespace
metadata:
  name: net-test
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: net-test
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-net
  template:
    metadata:
      labels:
        app: nginx-net
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-nodeport
  namespace: net-test
spec:
  type: NodePort
  selector:
    app: nginx-net
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  namespace: net-test
  name: nginx-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - http:
        paths:
          - path: /*
            backend:
              serviceName: nginx-service-nodeport
              servicePort: 8080
```

```
[ec2-user@ip-172-31-1-111 explore-network]$ kubectl get ingress -n net-test
NAME            CLASS    HOSTS   ADDRESS                                                                          PORTS   AGE
nginx-ingress   <none>   *       k8s-nettest-nginxing-2893a9a207-1682869329.cn-northwest-1.elb.amazonaws.com.cn   80      34m

[ec2-user@ip-172-31-1-111 explore-network]$ kubectl describe ingress -n net-test
Name:             nginx-ingress
Namespace:        net-test
Address:          k8s-nettest-nginxing-2893a9a207-1682869329.cn-northwest-1.elb.amazonaws.com.cn
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host        Path  Backends
  ----        ----  --------
  *           
              /*   nginx-service-nodeport:8080 (192.168.14.134:80,192.168.38.166:80,192.168.83.238:80)
Annotations:  alb.ingress.kubernetes.io/scheme: internet-facing
              alb.ingress.kubernetes.io/target-type: ip
              kubernetes.io/ingress.class: alb
Events:
  Type    Reason                  Age   From     Message
  ----    ------                  ----  ----     -------
  Normal  SuccessfullyReconciled  34m   ingress  Successfully reconciled

[ec2-user@ip-172-31-1-111 explore-network]$ kubectl get service -n net-test
NAME                     TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
nginx-service-nodeport   NodePort   10.100.194.22   <none>        8080:32436/TCP   40m
```
正如你在kubernetes官方页面( [https://kubernetes.io/docs/concepts/services-networking/ingress/](https://kubernetes.io/docs/concepts/services-networking/ingress/) )中看到的那样, Ingress就扮演一个连接器的角色，它将流量引入到Service.但我们可以看到为这个Ingress创建的ALB仍然使用的IP模式。也就是意味着Ingress也会如Service的LoadBalancer类型中的IP模式，流量直接引入到pod的ip地址,因为pod也具有vpc cidr range内分配的IP地址。同样这里也没有iptables中的具体规则。
另外，尽管此处为NodePort,其实如果是ClusterIP还是LoadBalancer,效果也一样，这一点完全体现了Ingress就扮演一个连接器的角色，它将流量直接引入到pod，借助了ALB中的IP模式的优势。
这里可以通过查看 ALB ( k8s-nettest-nginxing-2893a9a207-1682869329.cn-northwest-1.elb.amazonaws.com.cn )的target group。

iptables from [https://kubernetes.io/docs/concepts/services-networking/service/](https://kubernetes.io/docs/concepts/services-networking/service/)  
> Again, consider the image processing application described above. When the backend Service is created, the Kubernetes control plane assigns a virtual IP address, for example 10.0.0.1. Assuming the Service port is 1234, the Service is observed by all of the kube-proxy instances in the cluster. When a proxy sees a new Service, it installs a series of iptables rules which redirect from the virtual IP address to per-Service rules. The per-Service rules link to per-Endpoint rules which redirect traffic (using destination NAT) to the backends.  

> When a client connects to the Service's virtual IP address the iptables rule kicks in. A backend is chosen (either based on session affinity or randomly) and packets are redirected to the backend. Unlike the userspace proxy, packets are never copied to userspace, the kube-proxy does not have to be running for the virtual IP address to work, and Nodes see traffic arriving from the unaltered client IP address.
> This same basic flow executes when traffic comes in through a node-port or through a load-balancer, though in those cases the client IP does get altered.



### Reference:
[https://www.cisco.com/c/dam/global/fi_fi/assets/docs/SMB_University_120307_Networking_Fundamentals.pdf](https://www.cisco.com/c/dam/global/fi_fi/assets/docs/SMB_University_120307_Networking_Fundamentals.pdf)   
[https://en.wikipedia.org/wiki/OSI_model](https://en.wikipedia.org/wiki/OSI_model)  
[https://www.cisco.com/E-Learning/bulk/public/tac/cim/cib/using_cisco_ios_software/linked/tcpip.htm](https://www.cisco.com/E-Learning/bulk/public/tac/cim/cib/using_cisco_ios_software/linked/tcpip.htm)  
[https://www.cisco.com/c/en/us/support/docs/ip/routing-information-protocol-rip/13769-5.html](https://www.cisco.com/c/en/us/support/docs/ip/routing-information-protocol-rip/13769-5.html)  
[https://www.netfilter.org/](https://www.netfilter.org/)  
[https://www.netfilter.org/projects/iptables/index.html](https://www.netfilter.org/projects/iptables/index.html)  
[https://www.zsythink.net/archives/tag/iptables/](https://www.zsythink.net/archives/tag/iptables/)  
[https://ipset.netfilter.org/iptables-extensions.man.html](https://ipset.netfilter.org/iptables-extensions.man.html)  
[https://aws.amazon.com/premiumsupport/knowledge-center/eks-kubernetes-services-cluster/](https://aws.amazon.com/premiumsupport/knowledge-center/eks-kubernetes-services-cluster/)  
[https://kubernetes.io/docs/concepts/services-networking/service/](https://kubernetes.io/docs/concepts/services-networking/service/)  
[https://docs.aws.amazon.com/eks/latest/userguide/network-load-balancing.html](https://docs.aws.amazon.com/eks/latest/userguide/network-load-balancing.html)




