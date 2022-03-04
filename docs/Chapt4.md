# 第4章 管理员手册

## 4.1 管理手册

### 4.1.1 进阶配置

**网络节点配置**

部署OpenStack云时,每个服务的REST API都以一系列网络节点的形式呈现。这些节点是:管理URL、内部URL和外部URL

kolla提供了两种分配三种网络节点的方式:

- **融合**:全部三种网络节点共享同一个IP Address
- **分离**:external URL被分配一个与admin URL和internal URL共享地址不同的IP地址

关于这些选项的配置变量为:

- `kolla_internal_vip_address`
- `network_interface`
- `kolla_external_vip_address`
- `kolla_external_vip_interface`

融合选项配置如下两个变量,并接收另外两个选项的默认值,在这个配置中internal、external和admin节点的全部REST请求全部流经相同的网络

```yaml
kolla_internal_vip_address: "10.10.10.254"
network_interface: "eth0"
```

分离选项配置全部四个变量,在这个配置中internal和external节点的REST请求流量流经不同的网络	

```yaml
kolla_internal_vip_address: "10.10.10.254"
network_interface: "eth0"
kolla_external_vip_address: "10.10.20.254"
kolla_external_vip_interface: "eth1"
```

**FQDN配置**

当为互联网中的服务器配置地址,一般情况下会为其配置一个像`www.example.net`这样的名字来取代IP地址,如果想要在你的Kolla部署中使用名字来为网络端点配置地址,使用如下变量:
- kolla_internal_fqdn 
- kolla_external_fqdn

```yaml
kolla_internal_fqdn: inside.mykolla.example.net
kolla_external_fqdn: mykolla.example.net
```

名称与地址的映射必须使用DNS或/etc/hosts文件实现

**RabbitMQ Hostname Resolution**

RabbitMQ不使用IP地址,因此api_interface的IP地址应该是可解析的主机名,以确保所有RabbitMQ集群的主机可以提前解析彼此的主机名

**TLS Configuration**

配置TSL在[这里](##4.1.2)

## 4.1.2 TLS配置

本节介绍如何配置Kolla Ansible部署启用TLS协议的OpenStack。启用在提供的内部和/或外部VIP地址上,OpenStack客户端可以对OpenStack服务的网络通信进行认证和加密。

当一个OpenStack服务公开一个API端点时,Kolla Ansible会为该服务配置HAProxy监听内部和/或外部的VIP地址。HAProxy容器将vip上的负载平衡请求发送给运行服务容器的节点。

OpenStack api的TLS配置有两层:
1. 在内部和/或外部VIP上启用TLS, OpenStack客户端和在VIP上监听的HAProxy之间的通信是安全的。
2. 在后端网络上启用TLS,因此HAProxy和后端之间的通信API服务是安全的。

>**注意**: TLS认证基于受信任的证书颁发机构签署的证书。商业ca的例子有Comodo、赛门铁克、GoDaddy和GlobalSign。Letsencrypt.org是一个免费提供可信证书的CA。如果您的项目不可能使用受信任的CA,您可以使用私有CA,例如Hashicorp Vault,为您的域创建证书,或查看
生成一个私有证书颁发机构来使用Kolla Ansible生成的私有CA。
>关于支持ACME的ca的详细信息,如letsencrypt.org,请参见ACME http-01 challenge support。

**快速开始**

>**注意**: 由Kolla Ansible生成的证书使用简单的证书颁发机构设置,不适合生产部署。生产部署中应该只使用受信任的证书颁发机构签名的证书。
>
为了使用TLS部署 OpenStack 的 external,internal和backend APIs,在globals.yml配置如下内容:

```yaml
kolla_enable_tls_internal: "yes"
kolla_enable_tls_external: "yes"
kolla_enable_tls_backend: "yes"
kolla_copy_ca_into_containers: "yes"
```

如果在Debian 或 Ubuntu上部署

```yaml
openstack_cacert: "/etc/ssl/certs/ca-certificates.crt"
```

如果在CentOS 或 RHEL上部署:

```yaml
openstack_cacert: "/etc/pki/tls/certs/ca-bundle.crt
```

Kolla Ansible certificates命令生成一个私有的测试证书颁发机构,然后使用CA为启用的VIP(s)签名生成的证书,以在OpenStack部署中测试TLS。假设你正在使用`multinode`:

```bash
kolla-ansible -i ~/multinode certificates
```

**为 internal/external VPI 配置 TLS**

配置控制 TLS 相关配置参数

- kolla_enable_tls_external
- kolla_enable_tls_internal
- kolla_internal_fqdn_cert
- kolla_external_fqdn_cert
