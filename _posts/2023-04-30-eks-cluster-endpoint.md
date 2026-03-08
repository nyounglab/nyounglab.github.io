---
layout: post
title: aews) eks cluster endpoint
category:
  - study
  - aws
tags:
  - study
  - aws
---

가시다님의 eks 스터디에 참여하게되어 첫번째 시간에 eks의 architecture와 배포하는 것을 공부하고, 과제로 cluster endpoint에 대해 알아보았다.


# eks architecture


![1-0](/assets/img/aews/1-0.png)

aws의 관리형 쿠버네티스의 eks는 controlplane이 사용자의 vpc가 아닌 aws가 관리하는 vpc에 위치하게 된다. 실제 어플리케이션이 올라가는 dataplane이 사용자의vpc에 올라가게 되는데, 이때 controleplane에서 dataplane으로 내부 통신 할 수 있도록 eni가 연결된다. 이때 외,내부와 통신 방식에 따라 세가지 방식으로 클러스터가 달라진다. 

테스트는 eksctl을 사용해서 managed node group이 있는 cluster 생성하며, ap-northeast-2리전의 a,c존 두개의 subnet을 이용했다.


# eks cluster endpoint

## public

public이 기본값이기 때문에, 다른 설정없이 cluster를 생성했다. 생성하기 전 —dry-run 옵션을 붙여서 yaml을 확인해보면 vpc.clusterEndpoints에 privateAccess: false, publicAccess: true 를 확인 할 수 있다.

```

(eunyoung@myeks:default) [root@myeks-host ~]# eksctl create cluster --name $CLUSTER_NAME --region=ap-northeast-2 \
 --nodegroup-name=mynodegroup --node-type=t3.medium \
 --node-volume-size=30 --vpc-public-subnets "$subnet1,$subnet2" \
 --version 1.24 --ssh-access --external-dns-access --verbose 4

```

콘솔에서 생성된 클러스터 확인
![1-1](/assets/img/aews/1-1.png)


```

(eunyoung@myeks:default) [root@myeks-host ~]# eksctl get cluster
NAME	REGION		EKSCTL CREATED
myeks	ap-northeast-2	True

# cluster endpoint
(eunyoung@myeks:default) [root@myeks-host ~]# aws eks describe-cluster --name $CLUSTER_NAME --query cluster.endpoint
"https://4EF9B1C9A5F24DCBD324557BF7056DC0.gr7.ap-northeast-2.eks.amazonaws.com"

# public endpoint이기 때문에 외부에서 호출시에도 접근이 가능하다.
❯ curl -k -s https://4EF9B1C9A5F24DCBD324557BF7056DC0.gr7.ap-northeast-2.eks.amazonaws.com
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {},
  "code": 403
}%
                                                                                       13:43:03
❯ curl -k -s https://4EF9B1C9A5F24DCBD324557BF7056DC0.gr7.ap-northeast-2.eks.amazonaws.com/version
{
  "major": "1",
  "minor": "24+",
  "gitVersion": "v1.24.12-eks-ec5523e",
  "gitCommit": "3939bb9475d7f05c8b7b058eadbe679e6c9b5e2e",
  "gitTreeState": "clean",
  "buildDate": "2023-03-20T21:30:46Z",
  "goVersion": "go1.19.7",
  "compiler": "gc",
  "platform": "linux/amd64"
}%



# 외부에서 dig 명령어를 통해 endpoint를 확인해보면 두개의 외부 ip를 확인할 수 있다.
❯ dig +short 4EF9B1C9A5F24DCBD324557BF7056DC0.gr7.ap-northeast-2.eks.amazonaws.com
43.201.141.73
3.37.234.25

# 위 ip를 찾으러 떠나보자. ..
# 노드 안에서 tcp로 연결된 소켓 상태를 확인해보면,
# kube-proxy와 kubelet의 peer ip와 동일한 것을 통해 controlplane의 외부 ip로 생각할 수 있다.
(eunyoung@myeks:default) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$NODE1 sudo ss -tnp
State Recv-Q Send-Q Local Address:Port   Peer Address:Port Process
ESTAB 0      0       192.168.2.94:59986 43.201.141.73:443   users:(("kube-proxy",pid=3093,fd=11))
ESTAB 0      0       192.168.2.94:34456   3.37.234.25:443   users:(("kubelet",pid=2842,fd=42))
...

(eunyoung@myeks:default) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$NODE2 sudo ss -tnp
State Recv-Q Send-Q Local Address:Port   Peer Address:Port Process
ESTAB 0      0      192.168.1.149:41074   3.37.234.25:443   users:(("kube-proxy",pid=3088,fd=11))
ESTAB 0      0      192.168.1.149:42790 43.201.141.73:443   users:(("kubelet",pid=2840,fd=12))
..


# 노드 안에서 endpoint를 확인해보면, 외부와 동일하게 외부 ip가 반환된다.
(eunyoung@myeks:default) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$NODE1 dig +short 4EF9B1C9A5F24DCBD324557BF7056DC0.gr7.ap-northeast-2.eks.amazonaws.com
43.201.141.73
3.37.234.25
(eunyoung@myeks:default) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$NODE2 dig +short 4EF9B1C9A5F24DCBD324557BF7056DC0.gr7.ap-northeast-2.eks.amazonaws.com
3.37.234.25
43.201.141.73

```

![1-2](/assets/img/aews/1-2.png)



### controlplane의 eni

다른 endpoint를 확인 하기 전에, controlplane의 eni를 알아야한다.


```

# contolplane에서 dataplane에 접근을 할 때에는 어떻게되는지 확인하기 위해 aws-node에 접근 후, 노드에서 다시 소켓연결을 확인해봤다.
(eunyoung@myeks:default) [root@myeks-host ~]# kubectl exec daemonsets/aws-node -it -n kube-system -c aws-node -- bash
bash-4.2# 

(eunyoung@myeks:default) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$NODE1 sudo ss -tnp
State Recv-Q Send-Q         Local Address:Port           Peer Address:Port Process
ESTAB 0      0      [::ffff:192.168.2.94]:10250 [::ffff:192.168.2.10]:46156 users:(("kubelet",pid=2842,fd=7))
...
(eunyoung@myeks:default) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$NODE2 sudo ss -tnp
State Recv-Q Send-Q          Local Address:Port            Peer Address:Port Process
STAB 0      0      [::ffff:192.168.1.149]:10250 [::ffff:192.168.1.121]:55260 users:(("kubelet",pid=2840,fd=7))
...

```

새롭게 생성된 연결을 보면 peer ip가 192.168.2.10과 192.168.1.121 인 것을 볼 수 있다.
'ec2 > 네트워크 및 보안 > 네트워크 인터페이스'에 가서 ip에 해당하는 네트워크 인터페이스를 확인해보자


![1-3](/assets/img/aews/1-3.png)

보면 인스턴스id가 없어서 할당되지 않은 것처럼 보이지만, 자세히 살펴보면 다른 네트워크 인터페이스와 다른 점을 확인 할 수 있다. 해당 네트워크 인터페이스는 내가 소유하고 있지만,  요청자id와 인스턴스 소유자id를 통해 이는 다른 곳에서 사용되고 있는 것으로 확인할 수 있다. 이것이 그림들에 있는 controlplane의 eni다.

dataplane 노드의 eni정보를 확인하고 싶다면 (물론 콘솔에서도 가능하지만..) 아래와 같이 메타데이터를 호출하는 방식으로도 확인가능하다. [🔗](https://nyounglab.github.io/study/aws/2023/04/06/aws-%EA%B6%8C%ED%95%9C%ED%9B%94%EC%B9%98%EA%B8%B0/)


```

# 노드의 owner id 확인
(eunyoung@myeks:default) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$NODE2 curl -s http://169.254.169.254/latest/meta-data/acs/$MACNUM/owner-id
643127389001
# 노드에서 사용하는 eni
(eunyoung@myeks:default) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$NODE2 curl -s http://169.254.169.254/latest/meta-data/network/interfaces/macs/$MACNUM/interface-id/
eni-0e0e1e15c769414a9

```


## public + private

```

(eunyoung@myeks:default) [root@myeks-host ~]# eksctl utils update-cluster-endpoints --name=$CLUSTER_NAME --private-access=true --public-access=true --verbose 4 --approve

```

명령어로 업데이트 후, 해당 vpc 대역대에 대해 443포트를 열어 api호출이 가능하도록 아래와 같이 보안그룹 설정을 해주었다.

![1-4](/assets/img/aews/1-4.png)


```

# 외부에서 cluster endpoint를 확인해보면 여전히 외부 ip가 반환된다.
❯ dig +short 4EF9B1C9A5F24DCBD324557BF7056DC0.gr7.ap-northeast-2.eks.amazonaws.com
43.201.141.73
3.37.234.25

# 노드에서 cluster endpoint를 확인해보면, public이였을 때와 다르게 eni ip가 반환된다.
(eunyoung@myeks:default) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$NODE1 dig +short 4EF9B1C9A5F24DCBD324557BF7056DC0.gr7.ap-northeast-2.eks.amazonaws.com
192.168.1.121
192.168.2.10
(eunyoung@myeks:default) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$NODE2 dig +short 4EF9B1C9A5F24DCBD324557BF7056DC0.gr7.ap-northeast-2.eks.amazonaws.com
192.168.1.121
192.168.2.10

```

![1-5](/assets/img/aews/1-5.png)

그림과 같이 외부에서는 여전히 외부 ip로 접근되지만, 노드에서는 eni를 통해 접근 되는 것을 알 수 있다. 



## private

```

# update cluste endpoint 
(eunyoung@myeks:default) [root@myeks-host ~]# eksctl utils update-cluster-endpoints --name=$CLUSTER_NAME --private-access=true --public-access=false --verbose 4 --approve


# 외부에서 cluster endpoint를 확인해보면 eni ip가 반환된다.
❯ dig +short 4EF9B1C9A5F24DCBD324557BF7056DC0.gr7.ap-northeast-2.eks.amazonaws.com
192.168.2.10
192.168.1.121

# 더이상 외부에서 cluster에 접근이 불가능히다.
❯ curl -k -s https://4EF9B1C9A5F24DCBD324557BF7056DC0.gr7.ap-northeast-2.eks.amazonaws.com/version


# 노드에서 확인 시
(eunyoung@myeks:default) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$NODE1 dig +short 4EF9B1C9A5F24DCBD324557BF7056DC0.gr7.ap-northeast-2.eks.amazonaws.com
192.168.2.10
192.168.1.121
(eunyoung@myeks:default) [root@myeks-host ~]# ssh -i ~/.ssh/id_rsa ec2-user@$NODE2 dig +short 4EF9B1C9A5F24DCBD324557BF7056DC0.gr7.ap-northeast-2.eks.amazonaws.com
192.168.2.10
192.168.1.121

```

![1-6](/assets/img/aews/1-6.png)

이 경우는 내부에서만 eni를 통해 통신이 가능하고 외부와는 통신이 불가하기 때문에, 외부로 서비스하지않을 경우 사용할 수 있다. 
만약, ecr이나 s3등 aws의 다른 서비스와 연결하려면 vpc endpoint 설정이 필요하다.[🔗](https://docs.aws.amazon.com/eks/latest/userguide/private-clusters.html)


---
참고
- [https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html)
- [https://aws.amazon.com/ko/blogs/containers/de-mystifying-cluster-networking-for-amazon-eks-worker-nodes/](https://aws.amazon.com/ko/blogs/containers/de-mystifying-cluster-networking-for-amazon-eks-worker-nodes/)
- [https://eksctl.io/usage/vpc-cluster-access/](https://eksctl.io/usage/vpc-cluster-access/)
- [https://www.youtube.com/watch?v=aU9G1p85c-k&list=PLApuRlvrZKoielYfY9_7gqS7oB4MTMI9G](https://www.youtube.com/watch?v=aU9G1p85c-k&list=PLApuRlvrZKoielYfY9_7gqS7oB4MTMI9G)
