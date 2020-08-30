# okd4.5 AWS UPI
#### 官方文档：[https://docs.okd.io/latest](https://docs.okd.io/latest)

### 主要特性 ###
* Operator、Operator Hub、Operator Lifecycle Manager、Helm
* okd版本在线升级
* pod多网卡支持
* 内置protheus、thanos
* CSI支持AWS EFS


### 版本说明 ###
| 组件 | 版本 |
|--|--|
|Fedora CoreOS | 32.20200629.3.0 |
|okd| 4.5.0-0.okd-2020-08-12-020541 |




### 核心过程

1、启动bootstrap节点和master节点（如果使用UPI方式，需要手工启动）。

2、master节点从bootstrap节点获取资源并启动。

3、master节点通过bootstrap节点创建出etcd集群。

4、bootstrap节点启动一个临时的Kubernetes control plane并连接使用上述的etcd集群。

5、临时的Kubernetes Control plane将最终的production control plane调度到master节点。

6、临时的Kubernetes Control plane关闭并将控制权交给production control plane。

7、bootstrap节点将OpenShift Container Platform组件注入到production control plane。

8、安装程序关闭bootstrap节点（UPI部署将需要您手工操作）。

9、接下来，Control plane部署worker nodes。

10、Control plane以Operator形式部署额外的服务。

### 安装步骤 ###
安装步骤参考[https://docs.okd.io/latest/installing/installing_aws/installing-aws-user-infra.html](https://docs.okd.io/latest/installing/installing_aws/installing-aws-user-infra.html)

| 资源清单 | 说明 |
|--|--|
| Docker Registry节点 |  用于集群安装初始化核心镜像的拉取，将镜像保存在local registry中|
| bootstrap节点1台 | master节点从bootstrap节点获取资源并启动
| Master节点3台 | 集群Master节点 |
| Work节点两台 |  集群Work节点|

由于AWS UPI安装okd官方提供了cloudformation模版，我们只需按照文档中的cloudformation template稍作修改即可,本文所有操作在我已经准备好local registry后。

#### 1. 使用openshift-install安装程序创建install-config.yaml，这个文件提供了集群安装的一些初始化信息
 
>     ./openshift-install create install-config --dir=<installation_directory> 

####  2. 创建Kubernetes集群manifest 和Ignition 配置文件
 **注意**：Ignition文件中包含的证书有效期只有24个小时，意味着集群的安装工作必须在24个小时之内完成
 
>     ./openshift-install create manifests --dir=<installation_directory>
>    删除以下文件 （防止自动初始化control plane，同时worker节点手动创建也不需要初始化）
>    
>     rm -f <installation_directory>/openshift/99_openshift-cluster-api_master-machines-*.yaml
>     rm -f <installation_directory>/openshift/99_openshift-cluster-api_worker-machineset-*.yaml


####  3. 创建Ignition file

>     ./openshift-install create ignition-configs --dir=<installation_directory> 
        
这些文件创建完成后会生成一个cluster  infrastructure name，可以用这个来标识AWS集群资源，查看方法：
     
>     jq -r .infraID <installation_directory>/metadata.json 

##### 最后，可以使用cloud formation来创建相关资源了，包括（ELB、bootstrap、control plane、worker、相关EC2  IAM Role、Route53 DNS记录、Lambda函数、各个节点角色的安全组等）

### 4. cloud formation创建资源大致流程

1. 前置准备工作：创建cluster-sec、cluster-dns（具体资源列表参见template文件），本文中使用已有VPC，所以省略了一些前置步骤，如需使用新建VPC，可参见官方文档在创建cluster-sec之前做的一些准备工作。
 
2. 前置工作完成后，可以 创建bootstrap节点了，将ignition-file上传到s3（因为Coreos将ignition-file作为user-data打到ec2实例中的话，这个文件长度过长），然后使用cloud formation创建bootstrap节点
 
    **注意**：官方文档给的template的user-data包含的ignition-file的version与安装程序生成的version可能不一致，注意修改，具体可参见/template/bootstrap.ign。

3. 观察bootstrap节点状态，节点创建完成后会通过lambda函数自动加入到elb，观察elb的22623的服务状态，这个端口是control plan节点拉取ignition-file的服务端口，当端口状态正常后，即可创建control plan节点了。

4. control plan节点也通过cloud formation来创建，步骤不做多描述，注意一点：即和bootstarp一样的问题就是template中ignition-file的version，我是直接把生成的master.ign替换到template中。

5. control plan节点创建完成后，可以在bootstrap节点查看集群初始化状态

         journalctl -b -f -u bootkube.service
当出现“bootkube.service complete”字段时，代表集群初始化完成。


6. control plan节点创建完成后，可以同时创建worker节点，ignition-file也直接写进template中。为什么现在创建worker节点：因为集群初始化的时候，operator部署openshift一些基础组件的时候，有些pod的调度策略没有对master节点tolartion（可以手动配置Depolyment，我这里直接创建了两个worker节点）

          ##### 第6步完成之后，所有资源基本上已经创建完成

7. worker节点加入master节点时会生成相应的CSR请求，同意之

>     oc get csr
>     oc adm certificate approve <csr_name>
可以一次性操作(worker扩容的时候可以做成自动化)：

>     oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
最后，查看cluster operator执行状态：
>     watch -n5 oc get clusteroperators
所有clusteroperators的AVAILABLE的字段为True时，代表集群安装完成。

