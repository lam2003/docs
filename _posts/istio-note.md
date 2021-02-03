---
title: Istio安装笔记
date: 2021-02-03 18:09:38
tags: 
  - [k8s]
  - [istio]
categories:
  - k8s
---
- #### 在k8s上安装Istio
    ```
    #下载istio
    export ISTIO_VERSION=1.5.0
    curl -L https://istio.io/downloadIstio | sh -
    mv ./istio-1.5.0 /opt/istio 
    #修改/etc/environment，将/opt/istio/bin加入PATH
    sudo echo PATH=$PATH:/opt/istio/bin > /etc/environment
    source /etc/environment
    #在k8s上安装istio，此处我们使用default模式安装
    #在进行这一步之前，需要确保用户目录中已配置kubeconfig（~/.kube/config）
    istioctl manifest apply --set profile=default --set addonComponents.kiali.enabled=true
    ```
    可在安装时添加 *--set addonComponents.xxx.enabled=true*（xxx为组件名称）进行安装

- #### kiali初始化配置
    新版kiali由于未在docker环境变量中设置登录名与密码，导致初始登录时报错。解决办法：在kiali *Deployment.spec.template.spec.containters.env*中加入以下环境变量：

    ```
    {
        "name": "SERVER_CREDENTIALS_USERNAME",
        "valueFrom": {
            "secretKeyRef": {
                "name": "kiali",
                "key": "username",
                "optional": true
            }
        }
    },
    {
        "name": "SERVER_CREDENTIALS_PASSWORD",
        "valueFrom": {
        "secretKeyRef": {
            "name": "kiali",
            "key": "passphrase",
            "optional": true
            }
        }
    }
    ```
    然后为kiali创建configmap：
    
    ```
    kubectl create secret generic kiali -n istio-system --from-literal=username=admin --from-literal=passphrase=admin
    ```
    用户名：admin，密码：admin

- #### 解决istio中的外部鉴权问题
    istio官方推荐使用jwt进行鉴权，这种鉴权方法需要用户先去鉴权服务请求一个token，然后将此token置于http request的header之中。由于我们使用的token是由yy UDB生成的，此种鉴权方法并不适合我们。在ServiceMesh v1版本我们开发了一个ext-authz服务进行鉴权，要想在istio中使用，只需要编写envoyfilter，在*envoy.route*之前加入*envoy.ext_authz*，把流量导入ext-authz服务。

    ingress-gateway-ef.yaml:
    
    ```
    apiVersion: networking.istio.io/v1alpha3
    kind: EnvoyFilter
    metadata:
      name: ingress-gateway-ef
      namespace: istio-system
    spec:
      workloadSelector:
        labels:
          app: istio-ingressgateway
      configPatches:
      - applyTo: HTTP_FILTER 
        match:
          context: ANY
          listener:
            portNumber: 80
            filterChain:
              filter:
                name: envoy.http_connection_manager
                subFilter:
                  name: envoy.router
        patch:
          operation: INSERT_BEFORE
          value:
            name: envoy.ext_authz
            config: 
              grpc_service:
                envoy_grpc: 
                  cluster_name: outbound|8888||ext-authz.default.svc.cluster.local
    
    ```
- istio使用地域感知
    1. 由于我们的k8s集群是单集群-跨机房部署，请求需要优先就近路由到本地机房服务去处理以降低请求延时。istio1.1版本以上提供了地域感知的功能（Locality Load Balancing），能够在无侵入的情况下实现就近路由的功能。配置方法如下：
    在istio pilot *Deployment.spec.template.spec.containters.env*中加入环境变量：
    ```
    {
        "name": "PILOT_ENABLE_LOCALITY_LOAD_BALANCING",
        "value": "1"
    },
    
    ```
    
    2. 在名为istio的configmap中找到*localityLbSetting*项，确保其处于*enabled: true*状态（默认处于开启状态），如果流量控制需要更加细化，可以参考 [LocalityLoadBalancerSetting](https://istio.io/docs/reference/config/networking/destination-rule/#LocalityLoadBalancerSetting)进行配置
    
    3. 在k8s node上打上region/zone标签，标签全称请参考：[k8s知名标签](https://kubernetes.io/zh/docs/reference/kubernetes-api/labels-annotations-taints/)
    
    ```
    
    kubectl label node worker-sz02 topology.kubernetes.io/region=shenzhen
    kubectl label node worker-sz03 topology.kubernetes.io/region=shenzhen
    kubectl label node worker-bj02 topology.kubernetes.io/region=beijing
    kubectl label node worker-wx02 topology.kubernetes.io/region=wuxi
    
    ```
    
    4. 由于 [#12947](https://github.com/istio/istio/issues/12947#issuecomment-478836687)，我们需要在DestinationRule中为Service配置熔断规则，地域感知才会生效。DestinationRule示例如下：
    
    ```
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: outlier
    spec:
      host: "google.com"
      trafficPolicy:
        outlierDetection:
          consecutiveErrors: 7
          interval: 5m
          baseEjectionTime: 15m
    ```
    
    配置完成后进行测试，在DestinationRule中配置LoadBalance规则，无论是轮询还是一致性hash，请求都是在本地机房处理的（例如从深圳ingress-gateway进来的请求，只会在深圳机房内进行处理），满足我们的设计需要
                                
-  istio+k8s服务暴露方式
    
    ```    
    深圳机房                                                                         
                                                                             
        
        Physical Server1                                                             
        +-----------------------------------------------------+     
        |                             PodA                    |                                        
        |                             +---------------------+ |
        | Docker端口直接映射到主机端口    |  DeamonSet+HostPort | | 
        |<--------------------------->|+k8sService          |-|---------------+                   
        |                             |istio-ingress-gateway| |               |
        |                             +---------------------+ |               |              
        |                                      |              |               |
        |                                      |flannel       |               |
        |                          PodB        V              |               |
        |                   +--------------------------+      |               |
        |                   |  DeploymentSet+k8sService|      |               |flannel
        |                   |    BIZ-Service1          |      |               |
        |                   |                          |      |               |
        |                   +--------------------------+      |               |  
        |                                                     |               |
        |                                                     |               |
        |                                                     |               |
        |                                                     |               |
        +-----------------------------------------------------+               |
                                                                              |
                                                                              |
                                                                              |
        Physical Server2                                                      |
        +------------------------------------------------------+              |
        |                             PodC                     |              |                      
        |                   +------------------------+         |              |
        |                   |DeploymentSet+k8sService|         |              |
        |                   |    BIZ_Service2        |<--------|--------------+ 
        |                   |                        |         |
        |                   +------------------------+         |
        |                                      |               |
        |                                      |flannel        | 
        |                          PodD        V               |
        |                   +--------------------------+       |
        |                   |  DeploymentSet+k8sService|       |
        |                   |    BIZ_Service3          |       |
        |                   |                          |       |
        |                   +-------------------------+|       |                
        |                                                      |               
        |                                                      |
        |                                                      |
        |                                                      |
        +------------------------------------------------------+ 
                  
                  
    ```
    istio-ingresss-gateway使用DaemonSet进行部署，在DaemonSet中添加NodeSelector，对打上了front_end=true标签的节点进行部署，由于我们使用HostPort的方式进行服务暴露，部署时需要确定物理机上的80/443等端口未被占用。为了使用istio地域感知功能，需要为ingress-gateway配置k8s Service，否则地域感知不生效。内部的业务服务使用Deployment+k8sService的方式进行部署，可配置水平伸缩容器。

    **A&Q**
    
    1. istio-ingress-gateway为什么不使用k8s NodePort进行服务暴露？如下图所示，使用NodePort的情况下，从节点1进入的请求可能被转发到节点2进行处理，不适用于跨机房k8s集群
    
    ![image](/images/1-1.png)
    
    2. 为何不使用k8s官方推荐的LoadBalance作为istio-ingress-gateway的暴露方式？
    
    使用LB需要网络设备的支持（LoadBalancer在数据链路层发送ARP），需要服务器处于同一交换机内。对于跨机房k8s集群不适用。[LB原理](https://juejin.im/entry/5aeb158f518825672033e90f)
    
- 使用sds进行https单向证书配置
    在istio-ingressgateway Gateway.spec.servers添加tls
    
    ```
    kind: Gateway
    apiVersion: networking.istio.io/v1alpha3
    metadata:
      name: ingressgateway
      namespace: istio-system
      selfLink: >-
        /apis/networking.istio.io/v1alpha3/namespaces/istio-system/gateways/ingressgateway
      uid: ad31ce80-6caf-11ea-b592-6c92bf4940c4
      resourceVersion: '490919'
      generation: 3
      creationTimestamp: '2020-03-23T02:40:36Z'
      labels:
        operator.istio.io/component: IngressGateways
        operator.istio.io/managed: Reconcile
        operator.istio.io/version: 1.5.0
        release: istio
      annotations:
        kubectl.kubernetes.io/last-applied-configuration: >
          {"apiVersion":"networking.istio.io/v1alpha3","kind":"Gateway","metadata":{"annotations":{},"labels":{"operator.istio.io/component":"IngressGateways","operator.istio.io/managed":"Reconcile","operator.istio.io/version":"1.5.0","release":"istio"},"name":"ingressgateway","namespace":"istio-system"},"spec":{"selector":{"istio":"ingressgateway"},"servers":[{"hosts":["*"],"port":{"name":"http","number":80,"protocol":"HTTP"}}]}}
    spec:
      servers:
        - hosts:
            - '*'
          port:
            name: http
            number: 80
            protocol: HTTP
        - hosts:
            - front.100.com
          port:
            name: https
            number: 443
            protocol: HTTPS
          tls:
            credentialName: yy-cr
            mode: SIMPLE
            privateKey: sds
            serverCertificate: sds
      selector:
        istio: ingressgateway
    
    ```
    在终端输入
        
    ```
    #注意文件名称必须为cert与key，否则sds不识别
    kubectl create secret generic  cr-yy --from-file=cert  --from-file=key -n istio-system 
    ```
        


    
    
