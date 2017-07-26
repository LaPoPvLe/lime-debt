---
typora-copy-images-to: ../assets
---

# Consul

[TOC]



## 简介

- 是什么？
- 解决什么问题？
- 与其他产品相比特点是？
- 如何开始？

### Consul 是什么

Consul 拥有很多组件，但作为一个整体，是你基础设施中的一个发现与配置服务的工具。其提供的主要功能：

- **服务发现**： Consul 的客户端可以*提供*服务，如 api 或 mysql，并且其他客户端可以使用 Consul *发现*给定服务的提供者。运用 DNS 或 HTTP，应用可以很容易的找到他们依赖的服务。
- **健康检查**： Consul 客户端可以提供任何数量的健康检查，无论是与给定服务相关联（“网页服务器是否返回 200 OK”），还是与本地 node 相关（“内存使用率是否低于 90%”）。这些信息可以被操作员用于监控集群健康状况，也用于服务发现组件路由流量避开非健康的主机。
- **KV Store**：应用可以使用 Consul 分级键/值存储实现任何数量的目的，包括动态配置、功能开关、协作、领导人选举等等。简单的 HTTP API 使得使用上十分方便。
- **多数据中心**： Consul 支持多个数据中心且开箱即用。也就是说，Consul 的用户不用担心建立更多层次的抽象来扩展到多个区域。

Consul 被设计成对 DevOps 社区和应用开发者都是友好的，使其成为完美的现代、弹性架构。

### Consul 的基础设施

Consul 是分布式、高可用系统。本节将涵盖基础，故意忽略一些不必要的细节，便于快速理解 Consul 是如何工作的。更多详细细节，查看[In-depth architecture overview](https://www.consul.io/docs/internals/architecture.html)。

任何节点，只要为 Consul 提供服务，都要运行一个 *Consul* 代理。发现其他服务或获取/设置键值数据是不需要运行一个代理。代理的职责是节点上服务及节点自身的健康检查。

代理与一台或多台 *Consul* 服务器通话。Consul 服务器是数据存储和复制地方。服务器自己选举一个首领。尽管一台服务器也能工作，但是推荐 3 到 5 台，避免错误场景导致的数据丢失。推荐每个数据中心一个 Consul 集群。

基础组件需要发现其他服务或者节点可以查询任何 Consul 服务器 *或* 任何 Consul 代理。代理会自动转发查询。

每个数据中心运行一个 Consul 服务器集群。当一个跨数据中心服务发现或配置请求被执行时，本地 Consul 服务器转发请求到远程数据中心并返回结果。

## Consul vs. 其他软件

## 入门

### 安装 Consul

首先需要在机器上安装 Consul。Consul 通过二进制包形式发布适用于所有支持的平台和体系结构。也可以从源码编译安装，参见[文档](https://www.consul.io/docs/index.html)。

#### 开始安装 Consul

1.  [下载](https://www.consul.io/downloads.html)适合操作系统的版本；
2.  解压，执行 consul 文件。包中的任何其他文件都可以被安全的移除不影响功能；
3.  最后，确保 consul 二进制包在 PATH 中。

#### 验证安装

```shell
$ consul
Usage: consul [--version] [--help] <command> [<args>]

Available commands are:
    agent          Runs a Consul agent
    catalog        Interact with the catalog
    event          Fire a new event
    exec           Executes a command on Consul nodes
    force-leave    Forces a member of the cluster to enter the "left" state
    info           Provides debugging information for operators.
    join           Tell Consul agent to join cluster
    keygen         Generates a new encryption key
    keyring        Manages gossip layer encryption keys
    kv             Interact with the key-value store
    leave          Gracefully leaves the Consul cluster and shuts down
    lock           Execute a command holding a lock
    maint          Controls node or service maintenance mode
    members        Lists the members of a Consul cluster
    monitor        Stream logs from a Consul agent
    operator       Provides cluster-level tools for Consul operators
    reload         Triggers the agent to reload configuration files
    rtt            Estimates network round trip time between nodes
    snapshot       Saves, restores and inspects snapshots of Consul server state
    validate       Validate config files/directories
    version        Prints the Consul version
    watch          Watch for changes in Consul
```



### 运行 Consul 代理

代理执行服务器或客户端模式运行。每个数据中心至少有一个服务器，虽然推荐一个集群推荐 3 或 5 台服务器。

其他所有代理在客户端模式运行。客户端是一个极其轻量的进程用于注册服务，运行健康检查，及转发查询到服务器。集群中任何节点，每个都必须运行代理。

引导数据中心的更多信息，参见：[文档](https://www.consul.io/docs/guides/bootstrapping.html)。

#### 启动代理

为了简单，用开发模式启动 Consul 代理。该模式十分有用，使得搭建单节点 Consul 环境，快速且简单。但是不可用于生产环境，其不会持久化任何状态。

```shell
$ consul agent -dev
==> Starting Consul agent...
==> Consul agent running!
           Version: 'v0.9.0'
           Node ID: '1a96a5c8-02e7-ca46-2c0a-b3a92c355738'
         Node name: '3c2e7d05ff7b'
        Datacenter: 'dc1'
            Server: true (bootstrap: false)
       Client Addr: 127.0.0.1 (HTTP: 8500, HTTPS: -1, DNS: 8600)
      Cluster Addr: 127.0.0.1 (LAN: 8301, WAN: 8302)
    Gossip encrypt: false, RPC-TLS: false, TLS-Incoming: false

==> Log data will now stream in as it occurs:

    2017/07/26 02:23:20 [DEBUG] Using random ID "1a96a5c8-02e7-ca46-2c0a-b3a92c355738" as node ID
    2017/07/26 02:23:20 [INFO] raft: Initial configuration (index=1): [{Suffrage:Voter ID:127.0.0.1:8300 Address:127.0.0.1:8300}]
    2017/07/26 02:23:20 [INFO] raft: Node at 127.0.0.1:8300 [Follower] entering Follower state (Leader: "")
    2017/07/26 02:23:20 [INFO] serf: EventMemberJoin: 3c2e7d05ff7b.dc1 127.0.0.1
    2017/07/26 02:23:20 [INFO] serf: EventMemberJoin: 3c2e7d05ff7b 127.0.0.1
    2017/07/26 02:23:20 [INFO] consul: Adding LAN server 3c2e7d05ff7b (Addr: tcp/127.0.0.1:8300) (DC: dc1)
    2017/07/26 02:23:20 [INFO] agent: Started DNS server 127.0.0.1:8600 (udp)
    2017/07/26 02:23:20 [INFO] consul: Handled member-join event for server "3c2e7d05ff7b.dc1" in area "wan"
    2017/07/26 02:23:20 [INFO] agent: Started DNS server 127.0.0.1:8600 (tcp)
    2017/07/26 02:23:20 [INFO] agent: Started HTTP server on 127.0.0.1:8500
    2017/07/26 02:23:20 [WARN] raft: Heartbeat timeout from "" reached, starting election
    2017/07/26 02:23:20 [INFO] raft: Node at 127.0.0.1:8300 [Candidate] entering Candidate state in term 2
    2017/07/26 02:23:20 [DEBUG] raft: Votes needed: 1
    2017/07/26 02:23:20 [DEBUG] raft: Vote granted from 127.0.0.1:8300 in term 2. Tally: 1
    2017/07/26 02:23:20 [INFO] raft: Election won. Tally: 1
    2017/07/26 02:23:20 [INFO] raft: Node at 127.0.0.1:8300 [Leader] entering Leader state
    2017/07/26 02:23:20 [INFO] consul: cluster leadership acquired
    2017/07/26 02:23:20 [INFO] consul: New leader elected: 3c2e7d05ff7b
    2017/07/26 02:23:20 [DEBUG] consul: reset tombstone GC to index 3
    2017/07/26 02:23:20 [INFO] consul: member '3c2e7d05ff7b' joined, marking health alive
    2017/07/26 02:23:20 [INFO] agent: Synced node info
==> Failed to check for updates: Get https://checkpoint-api.hashicorp.com/v1/check/consul?arch=amd64&os=linux&signature=&version=0.9.0: x509: failed to load system roots and no roots provided
```

Consul 启动日志。从中，可以发现代理运行在服务器模式并声明为集群的领导者。此外，本地成员被标记为健康的集群成员。

#### 集群成员

从其他终端执行 `consul members`，可以查看 Consul 集群的成员。

```shell
$ consul members
Node          Address         Status  Type    Build  Protocol  DC
3c2e7d05ff7b  127.0.0.1:8301  alive   server  0.9.0  2         dc1
```

输出显示了我们的节点、运行的地址、及其健康状态、在集群中的状态、和一些版本信息。额外元信息可以通过 -detailed 标志查看。

`members` 命令的输出是基于[gossip protocol](https://www.consul.io/docs/internals/gossip.html)和最终一致。也就是说，在任何时间点，本地代理所看到的全局视图可能与服务器上的状态不完全匹配。需要全局的强一致性视图，请使用 [HTTP API](https://www.consul.io/api/index.html) 转发请求到 Consul 服务器。

```shell
$ curl localhost:8500/v1/catalog/nodes
[
    {
        "ID": "f7bebada-8151-f0ae-a61d-b9e4cb39f6bd",
        "Node": "3c2e7d05ff7b",
        "Address": "127.0.0.1",
        "Datacenter": "dc1",
        "TaggedAddresses": {
            "lan": "127.0.0.1",
            "wan": "127.0.0.1"
        },
        "Meta": {},
        "CreateIndex": 5,
        "ModifyIndex": 6
    }
]
```

此外，还可以通过 [DNS Interface](https://www.consul.io/docs/agent/dns.html) 查询节点。确认 Consul 代理的 DNS 服务运行在 8600 端口才能通过 DNS 查找。DNS 记录格式（如：“Armons-MacBook-Air.node.consul”）后面会覆盖更多详情。

```shell
$ dig @127.0.0.1 -p 8600 3c2e7d05ff7b

; <<>> DiG 9.9.5-3ubuntu0.15-Ubuntu <<>> @127.0.0.1 -p 8600 3c2e7d05ff7b
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: 25411
;; flags: qr rd; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;3c2e7d05ff7b.			IN	A

;; Query time: 0 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Wed Jul 26 04:27:13 UTC 2017
;; MSG SIZE  rcvd: 30

```

> dig 安装：`apt-get install dnsutils`

#### 关闭代理

`ctrl+c`（终端信号）正常停止代理。终端后，应该看到它离开集群并关闭。

```shell
2017/07/26 04:31:37 [INFO] Caught signal:  interrupt
    2017/07/26 04:31:37 [INFO] Graceful shutdown disabled. Exiting
    2017/07/26 04:31:37 [INFO] agent: Requesting shutdown
    2017/07/26 04:31:37 [INFO] consul: shutting down server
    2017/07/26 04:31:37 [WARN] serf: Shutdown without a Leave
    2017/07/26 04:31:37 [WARN] serf: Shutdown without a Leave
    2017/07/26 04:31:37 [INFO] manager: shutting down
    2017/07/26 04:31:37 [INFO] agent: consul server down
    2017/07/26 04:31:37 [INFO] agent: shutdown complete
    2017/07/26 04:31:37 [INFO] agent: Stopping DNS server 127.0.0.1:8600 (tcp)
    2017/07/26 04:31:37 [INFO] agent: Stopping DNS server 127.0.0.1:8600 (udp)
    2017/07/26 04:31:37 [INFO] agent: Stopping HTTP server 127.0.0.1:8500
    2017/07/26 04:31:37 [INFO] agent: Waiting for endpoints to shut down
    2017/07/26 04:31:37 [INFO] agent: Endpoints down
    2017/07/26 04:31:37 [INFO] Exit code:  1
```

当优雅的离开，Consul 会通知其他集群成员节点*离开*。如果强制杀死代理进程，集群的其他成员会检测到节点*错误*。当成员离开时，其服务和检测将从目录中移除。Consul 会自动尝试重连接到*失败*的节点，支持其从某些网络原因中恢复，但*离开*的节点讲不在联系。

此外，如果代理是服务器，优雅的退出是必要的来避免导致潜在的可用性终端影响 [consensus protocol](https://www.consul.io/docs/internals/consensus.html)。查看[guides section](https://www.consul.io/docs/guides/index.html)获得更多细节如何安全的添加和移除服务器。

### 服务

#### 定义服务

服务有两种注册方式：

-  通过 [service definition](https://www.consul.io/docs/agent/services.html)；
-  通过调用适当的 [HTTP API](https://www.consul.io/api/index.html)。

**服务定义** 是注册服务最常用的方式。基于上一步涉及的代理配置之上进行构建。

1.  创建 Consul 配置目录。Consul 会加载配置目录下的所有文件，一种 Unix 系统通用约定方式，目录名为：`/etc/consul.d`(`.d` 的后缀表示“这个目录包含一套配置文件”)。

   ```shell
   $ sudo mkdir /etc/consul.d
   ```

2.  写入一个服务定义配置文件。假设，我们有一个名为 “web” 运行在 80 端口的服务。此外，标记一个标签，用于额外方式查询服务：

   ```shell
   $ echo '{"service": {"name": "web", "tags": ["rails"], "port": 80}}' \
   | sudo tee /etc/consul.d/web.json
   ```

3.  重启代理，提供配置目录：

   ```shell
   $ consul agent -dev -config-dir=/etc/consul.d
   ==> Starting Consul agent...
   ...
       [INFO] agent: Synced service 'web'
   ...
   ```

   注意输出中的 "已同步" web 服务。表示代理加载了从配置文件中加载了服务定义，并且成功注册到了服务目录。

   如果需要注册多个服务，可以创建多个服务定义文件在 Consul 配置目录下。



#### 查询服务

一旦代理已启动且服务已同步，就可以通过 DNS 或 HTTP API 查询服务了。

##### DNS API

服务的 DNS 名称为 `NAME.service.consul`。默认，所有的 DNS 名称通常是 `consul` 名空间，但 [this is configurable](https://www.consul.io/docs/agent/options.html#domain)。`service` 的子域名告诉 Consul 正在查询服务，Name 是服务的名称。

对刚注册的 web 服务，完整的域名为：`web.service.consul`：

   ```shell
   $ dig @127.0.0.1 -p 8600 web.service.consul
   ; <<>> DiG 9.9.5-3ubuntu0.15-Ubuntu <<>> @127.0.0.1 -p 8600 web.service.consul
   ; (1 server found)
   ;; global options: +cmd
   ;; Got answer:
   ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 30906
   ;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
   ;; WARNING: recursion requested but not available

   ;; OPT PSEUDOSECTION:
   ; EDNS: version: 0, flags:; udp: 4096
   ;; QUESTION SECTION:
   ;web.service.consul.		IN	A

   ;; ANSWER SECTION:
   web.service.consul.	0	IN	A	127.0.0.1

   ;; Query time: 2 msec
   ;; SERVER: 127.0.0.1#8600(127.0.0.1)
   ;; WHEN: Wed Jul 26 06:33:15 UTC 2017
   ;; MSG SIZE  rcvd: 63
   ```

返回的记录包括服务可用的节点的 IP 地址。记录只能保存地址。

可以通过 DNS API 获取完整的地址和端口对的 SRV 记录：

   ```shell
   $ dig @127.0.0.1 -p 8600 web.service.consul SRV
   ; <<>> DiG 9.9.5-3ubuntu0.15-Ubuntu <<>> @127.0.0.1 -p 8600 web.service.consul SRV
   ; (1 server found)
   ;; global options: +cmd
   ;; Got answer:
   ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 65512
   ;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 2
   ;; WARNING: recursion requested but not available

   ;; OPT PSEUDOSECTION:
   ; EDNS: version: 0, flags:; udp: 4096
   ;; QUESTION SECTION:
   ;web.service.consul.		IN	SRV

   ;; ANSWER SECTION:
   web.service.consul.	0	IN	SRV	1 1 80 3c2e7d05ff7b.node.dc1.consul.

   ;; ADDITIONAL SECTION:
   3c2e7d05ff7b.node.dc1.consul. 0	IN	A	127.0.0.1

   ;; Query time: 0 msec
   ;; SERVER: 127.0.0.1#8600(127.0.0.1)
   ;; WHEN: Wed Jul 26 06:44:53 UTC 2017
   ;; MSG SIZE  rcvd: 105
   ```



##### HTTP API

```shell
$ curl http://localhost:8500/v1/catalog/service/web
[
    {
        "ID": "b4ea4a76-0bcc-f659-a75d-1e817d50548c",
        "Node": "3c2e7d05ff7b",
        "Address": "127.0.0.1",
        "Datacenter": "dc1",
        "TaggedAddresses": {
            "lan": "127.0.0.1",
            "wan": "127.0.0.1"
        },
        "NodeMeta": {},
        "ServiceID": "web",
        "ServiceName": "web",
        "ServiceTags": [
            "rails"
        ],
        "ServiceAddress": "",
        "ServicePort": 80,
        "ServiceEnableTagOverride": false,
        "CreateIndex": 6,
        "ModifyIndex": 6
    }
]
```

此目录 API 提供所有托管指定服务的节点。仅查询 [health checks](https://www.consul.io/intro/getting-started/checks.html) 中健康的实例。这正是 DNS 正在做的。如下：

```shell
$ curl http://localhost:8500/v1/health/service/web?passing
[
    {
        "Node": {
            "ID": "b4ea4a76-0bcc-f659-a75d-1e817d50548c",
            "Node": "3c2e7d05ff7b",
            "Address": "127.0.0.1",
            "Datacenter": "dc1",
            "TaggedAddresses": {
                "lan": "127.0.0.1",
                "wan": "127.0.0.1"
            },
            "Meta": {},
            "CreateIndex": 5,
            "ModifyIndex": 6
        },
        "Service": {
            "ID": "web",
            "Service": "web",
            "Tags": [
                "rails"
            ],
            "Address": "",
            "Port": 80,
            "EnableTagOverride": false,
            "CreateIndex": 6,
            "ModifyIndex": 6
        },
        "Checks": [
            {
                "Node": "3c2e7d05ff7b",
                "CheckID": "serfHealth",
                "Name": "Serf Health Status",
                "Status": "passing",
                "Notes": "",
                "Output": "Agent alive and reachable",
                "ServiceID": "",
                "ServiceName": "",
                "ServiceTags": [],
                "CreateIndex": 5,
                "ModifyIndex": 5
            }
        ]
    }
]
```

#### 更新服务

服务定义可以通过更新服务配置文件并发送一个 `SIGHUP` 到代理。这允许你更新服务而不需要停机或导致服务查询不可用。

此外，HTTP API 动态的添加、移除、及修改服务。

### Consul 集群

当 Consul 代理刚启动时，其一开始不知道其他任何节点：它是一个孤立的集群。为了学习其他集群成员，代理必须*加入*一个已存在的集群。为了加入一个已存在的集群，其仅需要知道一个*单一*已存在成员。当其加入后，代理会与这个成员交流（gossip）并快速发现集群中的其他成员。一个 Consul 代理可以加入任何其他代理，并不限于服务器模式。

#### 启动代理s

> 原示例中使用的 Vagrant，因为不熟悉，就采用 Docker 了。基于 Ubuntu:14.04 构建了镜像 consul:1.0

```
No.		hostname		ip
1		consul			172.17.0.2
2		consul2			172.17.0.3
```

1.  进入第一台服务器，在前面示例中，使用 `-dev` 标志来快速设置开发服务器。但是，在集群中光这些是不够的。现在起省略 `-dev` 表示，而按照下面内容指定集群标志。

   - **node**：集群中的每个节点必须有唯一的名称。默认下，Consul 用机器的主机名，但可以用 `-node` 命令行参数手工覆盖；
   - **bind**：指定一个 [bind address](https://www.consul.io/docs/agent/options.html#_bind) 指定 Consul 坚挺的地址，*必须* 可以被集群中的其他节点访问到。但是绑定地址并不是绝对必须的，因为其通常会提供最好的选择。Consul 默认会尝试监听系统中所有的 IPv4 接口，但如果找到多个私有 IPs 会启动失败。因为生产环境服务器通常有多个接口，指定一个绑定地址确保永远不会绑定到错误接口。
   - **server**：第一个节点作为集群中唯一的服务器。通过 `-server` 开关表明；
   - **bootstrap-expect**：建议 Consul 服务器我们期望加入的额外服务器的数量。目的是延迟日志复制直到期望数量的服务器成功加入。参见：[bootstrapping guide](https://www.consul.io/docs/guides/bootstrapping.html)；
   - **enable_script_checks**：设置为 true 为了启用健康检测能执行外部脚本。生产环境使用，希望配置 [ACLs](https://www.consul.io/docs/guides/acl.html) 共同配合控制注册任意脚本；
   - **config-dir**：指定服务和检测定义的目录。

   完整的实例如下：

   ```shell
   $ consul agent -server -bootstrap-expect=1 \
       -data-dir=/tmp/consul -node=agent-one -bind=172.17.0.2 \
       -enable-script-checks=true -config-dir=/etc/consul.d

   ==> WARNING: BootstrapExpect Mode is specified as 1; this is the same as Bootstrap mode.
   ==> WARNING: Bootstrap mode enabled! Do not enable unless necessary
   ==> Starting Consul agent...
   ==> Consul agent running!
              Version: 'v0.9.0'
              Node ID: '631e19dd-64ee-9d91-c7ca-242663224064'
            Node name: 'agent-one'
           Datacenter: 'dc1'
               Server: true (bootstrap: true)
          Client Addr: 127.0.0.1 (HTTP: 8500, HTTPS: -1, DNS: 8600)
         Cluster Addr: 172.17.0.2 (LAN: 8301, WAN: 8302)
       Gossip encrypt: false, RPC-TLS: false, TLS-Incoming: false

   ==> Log data will now stream in as it occurs:
   ...
   ```

2.  在第二台服务器中，节点名为：`agent-two`，且为客户端模式：

   ```shel
   $ consul agent -data-dir=/tmp/consul -node=agent-two -bind=172.16.0.3 \
       -enable-script-checks=true -config-dir=/etc/consul.d
       
   ==> Starting Consul agent...
   ==> Consul agent running!
              Version: 'v0.9.0'
              Node ID: '061017c1-22e5-873e-e889-f8bb9aa4e6e9'
            Node name: 'agent-two'
           Datacenter: 'dc1'
               Server: false (bootstrap: false)
          Client Addr: 127.0.0.1 (HTTP: 8500, HTTPS: -1, DNS: 8600)
         Cluster Addr: 172.17.0.3 (LAN: 8301, WAN: 8302)
       Gossip encrypt: false, RPC-TLS: false, TLS-Incoming: false

   ==> Log data will now stream in as it occurs:
   ...
      2017/07/26 07:56:00 [ERR] agent: failed to sync remote state: No known Consul servers

   ...
   ```

   目前，有两个 Consul 代理正在执行：一个服务器和一个客户端。两个 Consul 代理并不直到对方并且都是单个节点的集群。可以通过 `consul members` 验证：

   ```shell
   # consul
   $ consul members
   Node       Address          Status  Type    Build  Protocol  DC
   agent-one  172.17.0.2:8301  alive   server  0.9.0  2         dc1

   # consul2
   consul members
   Node       Address          Status  Type    Build  Protocol  DC
   agent-two  172.17.0.3:8301  alive   client  0.9.0  2         dc1
   ```

#### 加入集群

让第一个代理加入第二个代理，通过运行下列命令：

```shell
# consul
$ consul join 172.17.0.3
Successfully joined cluster by contacting 1 nodes.
```

每个代理日志都会输出加入集群的信息。

```shell
# consul
2017/07/26 08:02:47 [INFO] agent: (LAN) joining: [172.17.0.3]
2017/07/26 08:02:47 [INFO] serf: EventMemberJoin: agent-two 172.17.0.3
2017/07/26 08:02:47 [INFO] consul: member 'agent-two' joined, marking health alive
2017/07/26 08:02:47 [INFO] agent: (LAN) joined: 1 Err: <nil>

# consul2
2017/07/26 08:02:47 [INFO] serf: EventMemberJoin: agent-one 172.17.0.2
2017/07/26 08:02:47 [INFO] consul: adding server agent-one (Addr: tcp/172.17.0.2:8300) (DC: dc1)
2017/07/26 08:02:47 [INFO] consul: New leader elected: agent-one
```

现在，通过 `consul members` 可以查看每个代理成员，发现两个代理都已经互相认识了：

```shell
# consul
$ consul members
Node       Address          Status  Type    Build  Protocol  DC
agent-one  172.17.0.2:8301  alive   server  0.9.0  2         dc1
agent-two  172.17.0.3:8301  alive   client  0.9.0  2         dc1

# consul
$ consul members
Node       Address          Status  Type    Build  Protocol  DC
agent-one  172.17.0.2:8301  alive   server  0.9.0  2         dc1
agent-two  172.17.0.3:8301  alive   client  0.9.0  2         dc1

```

> **提醒**： 加入一个集群，一个 Consul 代理仅需要知道*一个已存在成员*。加入集群后，代理们会相互交流扩展到全部成员信息。

#### 启动时自动加入集群

理想状态下，任何时候数据中心有新的节点出现，它应该自动加入 Consul 集群而不需要人为干涉。Consul 推进自动加入通过启用 AWS、Google Cloud 或 Azure 中实例的自动发现用指定的标签键值。为了使用这种集成，添加 `retry_join_ec2`，`retry_join_gce` 或 `retry_join_azure` 嵌套对象到 Consul 配置文件。允许一个新的节点加入集群二不需要任何硬编码配置。除此之外，在启动时加入集群通过 `-join` 或 `start_join` 硬编码已知 Consul 代理地址。

#### 查询节点

与查询服务类似，Consul 提供了 API 查询节点自身。有 DNS API 和 HTTP API 两种。

**DNS API**： NAME.node.consul 或者 Name.node.DATACENTER.consul，如果数据中心省略，Consul 仅搜索本地数据中心。

实例：从 "agent-one" 查询 "agent-two" 的地址：

```shell
# consul
$ dig @127.0.0.1 -p 8600 agent-two.node.consul
...
;; QUESTION SECTION:
;agent-two.node.consul.		IN	A

;; ANSWER SECTION:
agent-two.node.consul.	0	IN	A	172.17.0.3
...
```

除了服务以外，查找节点的能力对于系统管理任务也是非常有用的。举例来说，知道节点的地址并SSH连接与使其成为集群一部分并查询它一样容易。

#### 退出集群

只要优雅的退出代理（通过 Ctrl-C）或者强制杀死某个代理。优雅退出使得节点变为*离开*状态，否则是*失败*状态。更多区别参见：[here](https://www.consul.io/intro/getting-started/agent.html#stopping)。



### 健康检查

节点和服务的健康检查，其是服务发现的关键组成部分，来避免使用非健康的服务。

#### 定义检查

与服务类似，检查可以通过提供[检查定义](https://www.consul.io/docs/agent/checks.html) 或调用适当的 HTTP APi 完成注册。

**检查定义**：相对通用的定义方式。在第二个节点（consul2）的 Consul 配置目录中创建两个定义文件：

```shell
# consul2
$ echo '{"check": {"name": "ping",
  "script": "ping -c1 baidu.com >/dev/null", "interval": "30s"}}' \
  >/etc/consul.d/ping.json
$ echo '{"service": {"name": "web", "tags": ["rails"], "port": 80,
  "check": {"script": "curl localhost >/dev/null 2>&1", "interval": "10s"}}}' \
  >/etc/consul.d/web.json
```

第一个定义添加主机级别检查名称为“ping”。该检测间隔 30 秒执行一次，调用 `ping -c1 baidu.com`。在一个基于脚本的健康检查中，检查使用与 Consul 进程启动一样的用户。如果命令以非零退出码退出，节点会被标记为非健康的。也是任何基于脚本健康检查的约定。

第二个命令修改名称为 web 的服务，添加检查每 10 秒请求一次通过 curl 来验证 web 服务是否可访问。与主机级别的健康检查一样，如果脚本以非零退出码退出，服务会被标记为非健康的。

现在，重启第二个代理，重新加载通过 `consul reload`，或发送 SIGHUP 信号。可以看到下列日志：

```shell
==> Starting Consul agent...
...
    [INFO] agent: Synced service 'web'
    [INFO] agent: Synced check 'service:web'
    [INFO] agent: Synced check 'ping'
    [WARN] Check 'service:web' is now critical
```

代理已同步新的定义。最后一行表示 web 服务的检查结果是危险的。因为并没有运行任何 web 服务，所以 curl 测试失败了！

#### 检查健康状态

使用 HTTP API 来观察他们。首先，查找任何失败检测通过这个命令（注意，可以在任何节点运行）：

```shell
$ curl http://localhost:8500/v1/health/state/critical
[{"Node":"agent-two","CheckID":"service:web","Name":"Service 'web' check","Status":"critical","Notes":"","Output":"","ServiceID":"web","ServiceName":"web","ServiceTags":["rails"],"CreateIndex":259,"ModifyIndex":324}]
```

可以看到只有一个检测，web 服务检测，是 危险 状态。

此外，还可以尝试 DNS 查询 web 服务。Consul 不会范围任何结果如果服务是不健康的。

```shell
$ dig @127.0.0.1 -p 8600 web.service.consul

# 成功返回了 172.17.0.2 ....悲剧！！！
# 原因是构建image时，默认都有web服务，导致consul2的 web 服务出错后，在consul上任有一个web服务正常运行
...
;; QUESTION SECTION:
;web.service.consul.		IN	A

;; AUTHORITY SECTION:
consul.			0	IN	SOA	ns.consul. postmaster.consul. 1501059152 3600 600 86400 0
...
```



