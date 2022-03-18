# 第4章 管理员手册

## 4.1 管理手册
---
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

**Kolla 中 OpenStack Service 配置**

管理员可以通过编辑`/etc/kolla/globals.yml`并且添加下面的内容,改变自定义config files读取的位置。

```yaml
# The directory to merge custom config files the kolla's config files
node_custom_config: "/etc/kolla/config"
```

kolla允许管理员重写services的配置。Kolla一般在`/etc/kolla/config/<< config file >>`,`/etc/kolla/config/<< service name >>/<< config file >>`或者`/etc/kolla/config/<< service name >>/<< hostname >>/<< config file >>`,但是这些位置有时是不同的,应该在适当的Ansible Roles中检查配置任务,以获得支持位置的完整列表。 

例如在配置nova.conf的示例中,假设你有运行于controller-0001,controller-0002,controller-0003的nova,需要配置nova.conf,支持如下的定位方式

- /etc/kolla/config/nova.conf
- /etc/kolla/config/nova/controller-0001/nova.conf
- /etc/kolla/config/nova/controller-0002/nova.conf
- /etc/kolla/config/nova/controller-0003/nova.conf
- /etc/kolla/config/nova/nova-scheduler.conf

使用这种机制,可以覆盖每个project,每个project中的service,指定主机上project中service的配置文件。

覆盖一个选项如配置一样简单,例如,在nova scheduler要覆盖`scheduler_max_attempts`,管理员可以使用如下内容创建`/etc/
kolla/config/nova/nova-scheduler.conf`:

```
[DEFAULT]
scheduler_max_attempts = 100
```

如果管理员想要在myhost节点上配置计算节点CPU和RAM的配额,需要使用如下内容创建`/etc/kolla/config/nova/myhost/nova.conf`:

```
[DEFAULT]
cpu_allocation_ratio = 16.0
ram_allocation_ratio = 5.0
```

使用Oslo Config的所有服务都支持这种合并配置的方法,其中包括绝大多数OpenStack服务,在某些情况下也支持使用YAML配置的服务。由于INI格式是一种非正式的标准,并不是所有的INI文件都可以以这种方式合并。在这些情况下,Kolla支持覆盖整个配置文件。

通过在配置文件中使用Jinja条件可以引入额外的灵活性。例如,您可以创建相对于管理程序模型而言是同构的Nova Cell。在每个
Cell中,你可能希望以不同的方式配置hypervisor,例如下面的覆盖显示了将bandwidth_poll_interval变量设置为Cell函数的一种方法:

```
[DEFAULT]
{% if 'cell0001' in group_names %}
bandwidth_poll_interval = 100
{% elif 'cell0002' in group_names %}
bandwidth_poll_interval = -1
{% else %}
bandwidth_poll_interval = 300
{% endif %}
```

Jinja条件的另一种替代方法是为bandwidth_poll_interval定义一个变量,并根据inventory或host vars中的需求对其进行设置:

```
[DEFAULT]
bandwidth_poll_interval = {{ bandwidth_poll_interval }}
```

kolla允许管理员覆盖所有服务的全局配置文件,它会查找`/etc/kolla/config/global.conf`文件

例如,要修改数据库连接池大小,管理员需要在`/etc/kolla/config/global.conf`中配置:

```
[database]
max_pool_size = 100
```

**Openstack自定义policy**

openstack允许自定义policy。从Q版本开始,默认policy随每个服务的源代码定义,这意味着管理员只需要覆盖他们需要修改的内容。项目通常会提供关于默认策略配置的文档,例如Keystone。

策略可以通过JSON或YAML文件配置。**在Wallaby版本中,JSON格式已被弃用,取而代之的是YAML**。YAML的一个主要好处是它允许使用注释。

例如,要配置Neutron policy ,管理员可以在`/etc/kolla/config/neutron/policy.yaml`中添加自定义rules

管理员可以通过以下命令在服务部署完成后进行这些更改:

```bash
kolla-ansible deploy
```

为了向用户提供正确的界面,Horizon包含了其他services的策略。可能需要在Horizon中复制对这些服务进行的定制。例如,要定制
Horizon,使用YAML格式的Neutron rulse,管理员应在`/etc/kolla/config/Horizon/neutron_policy.yaml`中添加自定义规则。

**IP地址有限环境**

如果一个开发环境没有可用的VIP,可以通过禁止HAProxy,以使用host IP:

```yaml
enable_haproxy: "no"
```
请注意,这种方法不推荐使用,通常也不会由Kolla社区进行测试,但包含在其中,因为有时测试环境中无法使用免费IP。

在这种模式下,仍然需要配置kolla_internal_vip_address,它应该取api_interface接口的IP地址。

**外部 Elasticsearch/Kibana 环境**

可以使用外部Elasticsearch/Kibana环境。为此,首先禁用中央日志记录的部署

```
enable_central_logging: "no"
```

现在,您可以使用参数`elasticsearch_address`来配置外部的地址Elasticsearch环境。


**非默认service端口配置**

有时候要使用一个与默认服务不同的端口号,可以在globals.yaml中配置`<service>_port`:

```
database_port: 3307
```
由于<service>_port值保存在不同的服务配置中,因此建议在部署之前进行以上更改。
  
**使用外部日志服务器**
  
默认情况下,使用Fluentd作为syslog服务器收集Swift和HAProxy日志。当Fluentd被禁用或需要使用外部syslog服务器时,可以在globals.yml中设置syslog参数。例如:
  
```yaml
syslog_server: "172.29.9.145"
syslog_udp_port: "514"
```

管理员还可以为Swift和HAProxy配置日志源name,默认情况下Swift和HAProxy使用local0和local1
  
```yml
syslog_swift_facility: "local0"
syslog_haproxy_facility: "local1"
```
  
如果启用了Glance TLS后端(glance_enable_tls_backend),则glance_tls_proxy服务的syslog功能默认使用local2。这可以通过syslog_glance_tls_proxy_facility设置。

如果启用了Neutron TLS后端(neutron_enable_tls_backend), neutron_tls_proxy服务的syslog功能默认使用local4。这可以通过syslog_neutron_tls_proxy_facility设置.

**容器中挂在附加volume**

有时,能够将额外的Docker Volume装入一个或多个容器是很有用的。这可能是为了将第三方组件集成到OpenStack中,或者提供对特定于站点的数据的访问,比如x.509证书包

附加volme可以在三个层次设置:

- globally
- per-service (e.g. nova)
- per-container (e.g. nova-api)

指定全局扩展volume,在globals.yml中设置`default_extra_volumes`
  
```
default_extra_volumes:
- "/etc/foo:/etc/foo"
```
  

  

































---
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

Kolla Ansible certificates命令生成一个私有的测试证书颁发机构,然后使用CA为启用的VIP(s)签名生成的证书,以在OpenStack部署中测试TLS。假设您正在使用`multinode`:

```bash
kolla-ansible -i ~/multinode certificates
```

**为 internal/external VPI 配置 TLS**

配置控制 TLS 相关配置参数

- kolla_enable_tls_external
- kolla_enable_tls_internal
- kolla_internal_fqdn_cert
- kolla_external_fqdn_cert

>**注意**: 如果 TLS 只在 internal 或 external 网络上启动,那么`kolla_internal_vip_address` 和 `kolla_external_vip_address` 必须是不相同的
>
>如果在你的拓扑中只有一个网络配置,TLS只能在internal网络上配置启动
>

默认TLS网络是不启动的,要启动external TLS:

```yaml
kolla_enable_tls_external: "yes"
```

要启动internal TLS:

```yaml
kolla_enable_tls_internal: "yes"
```

使用TLS安全认证需要两个证书文件,这将由证书颁发机构提供:

- 带有私钥信息的服务器证书
- CA 证书

合并的服务器证书和私钥需要提供给Kolla Ansible,通过`kolla_external_fqdn_cert`或者`kolla_internal_fqdn_cert`配置。这些路径默认为`{{kolla_certificates_dir}}/haproxy.pem`和`{{kolla_certificates_dir}}/haproxy-internal.pem`。其中`kolla_certificates_dir`默认为`/etc/kolla/certificates`。

如果客户端还不信任所提供的服务器证书,则需要将CA证书文件分发给客户端。详细内容请参见配置OpenStack客户端TLS和添加CA证书到服务容器。

**为Openstack Client配置TLS**

为`admin-openrc.sh`晚间定位定位CA证书,需要配置`kolla_admin_openrc_cacert`变量,默认并未进行设置。这个path必须在所有运行`admin-openrc.sh`的节点可用。

当TLS在VIP上启动,并且kolla_admin_openrc_cacert 被设置为 /etc/pki/tls/certs/ca-bundle.crt,Openstack client会被admin-openrc.sh设置类似的配置:

```bash
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=demoPassword
export OS_AUTH_URL=https://mykolla.example.net:5000
export OS_INTERFACE=internal
export OS_ENDPOINT_TYPE=internalURL
export OS_MISTRAL_ENDPOINT_TYPE=internalURL
export OS_IDENTITY_API_VERSION=3
export OS_REGION_NAME=RegionOne
export OS_AUTH_PLUGIN=password
# os_cacert is optional for trusted certificates
export OS_CACERT=/etc/pki/tls/certs/ca-bundle.crt
```

**为服务容器添加CA证书**

为拷贝CA证书文件到服务容器:

```yaml
kolla_copy_ca_into_containers: "yes"
```

当`kolla_copy_ca_into_containers`被配置为yes,在`/etc/kolla/certificates/ca` 中的CA证书会被拷贝到service containers。这对于任何自签名或由私有CA签名的证书都是必需的,而且服务映像信任存储库中还没有这些证书。当容器启动时,Kolla将在容器系统信任存储区中安装这些证书。

当将所有证书文件名复制到容器中时,它们都将带有`kola-customca-`前缀。例如,如果一个证书文件`internal.crt`,它在容器中会被命名为`kolla-customca-internal.crt`。

Debian和Ubuntu容器,证书文件会被拷贝到` /usr/local/share/ca-certificates/ `目录

CentOS和RHEL容器,证书文件会被拷贝到`/etc/pki/ca-trust/source/anchors/`目录

在这两种情况下,有效的证书将被添加到系统信任存储区
- Debian和Ubuntu系统/etc/ssl/certs/ca-certificates。
- CentOS和RHEL系统/etc/pki/tls/certs/ca-bundle.crt。

**配置CA bundle**

OpenStack服务默认并不总是信任来自系统信任库的CA证书。要解决这个问题,openstack_cacert变量应该配置容器中CA证书的路径。

在Debian或Ubuntu上使用系统信任存储:

```yaml
openstack_cacert: /etc/ssl/certs/ca-certificates.crt
```
适用于CentOS或RHEL:

```yaml
openstack_cacert: /etc/pki/tls/certs/ca-bundle.crt
```
