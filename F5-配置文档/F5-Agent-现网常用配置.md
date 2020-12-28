# 现网环境 F5 agent 部署配置文档

## 现网环境 neutron lbaas 关键配置

安装完 Neutron Lbaas 后，需要运行建表命令，构建 neutron lbaas 数据表在 neutron database 中：

```bash
neutron-db-manage --subproject neutron-lbaas upgrade head
```

#### 配置项

##### service_plugins

`service_plugins` 配置项在 `neutron.conf` 配置文件`[DEFAULT]` section 中，`service_plugins`是指 neutron 需要加载的 plugins 类型。如果配置中已经存在了一些其他 plugins 的配置，在配置后加逗号，再加上`neutron_lbaas.services.loadbalancer.plugin.LoadBalancerPluginv2`如下：

```ini
[DEFAULT]
...
service_plugins = <已存在的其他 plugins>,neutron_lbaas.services.loadbalancer.plugin.LoadBalancerPluginv2
...
```

##### service_provider

`service_provider`配置项指定提供服务的类型，名称和driver位置。

此配置项一般在 `neutron_lbaas.conf` 的 `[service_providers]` section 中。如果安装完 neutron_lbaas 文件后没有 `neutron_lbaas.conf` 文件，需要找专业 `Openstack` 平台实施创建此配置文件。配置如下：

```ini
[service_providers]
...
# DMZ provider
service_provider=LOADBALANCERV2:DMZ:neutron_lbaas.drivers.f5.v2_DMZ.DMZ

# CORE provider
service_provider=LOADBALANCERV2:CORE:neutron_lbaas.drivers.f5.v2_CORE.CORE:default
...
```

`service_provider`的配置模式是：`<服务类型 agent_type>:<Provider 服务名称>:<Provider Driver>`

##### f5_driver_perf_mode

`f5_driver_perf_mode` 是 F5 添加在 `neutron_lbaas.conf` `[DEFAULT]` 中的配置项。F5 agent 提供了 `1，2，3` 三个 mode 的性能优化，一般现网环境直接使用 mode `3`。配置如下：

```ini
[DEFAULT]
...
f5_driver_perf_mode = 3
...
```

##### agent_down_time

`agent_down_time` 配置不是我们新增的参数,是在 `neutron.conf` 文件中本来就有的一个参数，我们设置为 300 是 best practice 配置。

```ini
[DEFAULT]
...
agent_down_time = 300
...
```

##### to_speedup_populate_logic

`to_speedup_populate_logic` 配置是我们新增的参数, 一个加速配置，减少一些db查询操作的参数, 在 `neutron.conf` 文件中配置，配置如下：

```ini
[DEFAULT]
...
to_speedup_populate_logic = true
...
```

## 现网环境 F5 lbaas agent 关键配置

F5 的配置文件存在于 `/etc/neutron/services/f5/f5-openstack-agent.ini`中。

```ini
[DEFAULT]
# True 开启 False 关闭 debug 功能，建议在 F5 agent 正常启动后设置 debug = False。
debug = True

# lite 表示 agent 切换成 3.0 代码，normal 表示 agent 使用 2.0 代码。
f5_agent_mode = lite

# 此配置和 F5 agent 定时同步（以秒为单位） neutron DB 数据到 Bigip 有关。建议更具环境需求配置。
periodic_interval = 1800

# 此配置和 F5 agent 定时同步（以秒为单位） neutron DB 数据到 Bigip 有关，在没有任何 report_state, 和对 neutron lbaas 操作的情况下，service_resync_interval 决定了每隔多久去做数据同步。建议可以将此值设置大一些，这样在没有对 Lbaas 操作的情况下，不会频繁 sync 数据。注意此配置只存在于 agent 2.0 中，只在使用 agent 2.0 的时候修改此配置才有意义。
# 86400 秒为一天
service_resync_interval = 86400 

# True 定时同步 ESD 文件，注意此配置只存在于 agent 2.0 中，只在使用 agent 2.0 的时候修改此配置才有意义。建议如果环境不使用任何 ESD 功能可以将此值设置为 False。
esd_auto_refresh = True

# 此配置和 rabbitmq 有关系。建议现网环境中将其指定为 service_provider 名称，如 CORE 或者 DMZ
environment_prefix = 'CORE'

# 此配置和 F5 agent 多活，rabbitmq 有关。建议如果开启多活，每个 F5 agent 的 agent_id 需要一样。
agent_id = POD1_CORE

# 如果 F5 agent 对接的是 HA bigip pair 这里需要配置成 pair，如果是 F5 agent 对接的是 standalone 的 bigip，这里配置成 standalone。
f5_ha_type = pair

# f5network1: 在现网中 neutron physical interface （名称 f5network 1）下创建的 loadbalancer 或者 member 资源对应的网络配置，配置在 bigip 的 RAg3 trunk 上。
# default: 在现网中 neutron 任何 interface 下创建的 loadbalancer 或者 member 资源对应的网络配置，都默认配置在 bigip 的 RAg3 trunk 上
f5_external_physical_mappings = f5network1:RAg3:True,default:RAg3:True

# 此配置项一般配置成 False，建议保持默认配置
f5_populate_static_arp = False

# 此配置项一般配置成 True，建议保持默认配置
l2_population = True

# 现网环境中如果使用 2 层网络比如 vlan 和 vxlan，这里需要设置成 False
f5_global_routed_mode = False

# 开启 Openstack project 对应 bigip route domain，建议此值设置为 True。
use_namespaces = True

# 定义每个 project 下最多可以对应多少个 route domain，建议此值设置为 1 每个 project 对映一个 route domain。
max_namespaces_per_tenant = 1

# 隔离 route tables，如果此值为 True 只有在同一个 tenant/project 下的 VIPS 和 pool members 才可以建立连接。
f5_route_domain_strictness = False

# 开启 listener SNAT 功能。建议如果使用 listener SNAT 需要设置此值为 True。
f5_snat_mode = True

# 当同一个 tenant 使用同一个 subnet 在不同 provider 下创建 snat pool 时，需要配置当前 agent 所对应的 provider name，用于确保不同 provider 下 snat pool 中的 member ip 是不一样的。
provider_name = 'CORE'

# 开启 listener SANT pool 功能，并且定义每个 SNAT pool 中 member 个数。建议如果使用 SNAT pool，此值根据环境至少设置为 1。如果设置为 0，listener 为 auto SNAT 功能。
f5_snat_addresses_per_subnet = 1

# 现网暂时不会用到此功能，建议现网保持默认值 False
f5_common_networks = False

# 建议现网保持默认值 True
f5_common_external_networks = True

# 现网暂时不会用到此功能，建议现网保持默认值 False
external_gateway_mode = False

# 建议现网保持默认值
f5_bigip_lbaas_device_driver = f5_openstack_agent.lbaasv2.drivers.bigip.icontrol_driver.iControlDriver

# 配置 BIGIP 管理口地址。建议更具现网环境配置
icontrol_hostname = 10.218.203.139,10.218.203.140

# 配置 BIGIP 管理员用户名。
icontrol_username = admin

# 配置 BIGIP 管理员密码。
icontrol_password = a112312

# F5 agent 3.0 在创建 listener 的时候可以通过此文件定制化 http/https profile 中参数。
f5_extended_profile = /etc/neutron/services/f5/f5-extended-profile.json
```

## F5 agent 2.0 数据同步机制配置

F5 agent 的同步和 F5 agent 配置文件中的`periodic_interval` 和 `service_resync_interval` 有关系。

```ini
# 此配置和 F5 agent 定时同步 neutron DB 数据到 Bigip，清理 error/orphan bigip 数据有关。设置此值，F5 agent 会定期检查 service_resync_interval 配置所设置的时间是否到达，或者是否有用户对 neutron lbaas 进行操作。建议更具环境需求配置。
periodic_interval = 1800

# 此配置和 F5 agent 定时同步 neutron DB 数据到 Bigip 有关，在没有任何 report_state, 和对 neutron lbaas 操作的情况下，service_resync_interval 决定了每隔多久去做数据同步。建议可以将此值设置大一些，这样在没有对 Lbaas 操作的情况下，不会频繁 sync 数据。
# 86400 秒为一天
service_resync_interval = 86400
```

频繁的配置下发和同步数据过程之前可能存在下发速度上的相互影响，因为配置下发和同步数据在同一个下发列队中，这两种操作会按照顺序完成，如果，同步配置量较大，可能会让正常配置下略有等待。具体 `periodic_interval` 和 `service_resync_interval`设置多少要待后期验证和实验更近发布。现在建议 `service_resync_interval` 设置为一天，`periodic_interval `建议设置为 1800 秒。

 环境配置过程中，需要根据环境，用户操作下发频率，同步频率，Bigip 设备更换频率，Bigip 配置更新频率等综合考虑全量同步对配置下发效率的影响。后期我们会进行实验和观察客户状态，给出进一步的分析结果。

## REFERENCES

[F5 Integration for OpenStack Neutron LBaaS - Troubleshooting](https://clouddocs.f5.com/cloud/openstack/v1/troubleshooting/troubleshoot-lbaas.html)
