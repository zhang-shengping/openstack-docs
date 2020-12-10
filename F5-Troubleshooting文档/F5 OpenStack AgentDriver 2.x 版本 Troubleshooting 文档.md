**F5 OpenStack Agent 没有运行起来**

**问题描述：**
登录Controller 节点，用云平台Admin用户运行 Neutron agent-list, f5-openstack-agent, 或者 f5-oslbaasv2-agent 没有出现在agent列表或者F5 agent没有运行**检查步骤：**

- 检查neutron.conf 配置

`sudo vi /etc/neutron/neutron.confService_plugin` 配置：

```ini
# The service plugins Neutron will use (list value)
service_plugins=router,qos,trunk,neutron_lbaas.services.loadbalancer.plugin.LoadBalancerPluginv2
```

- 检查 neutron_lbaas.conf 配置

`sudo vi /etc/neutron/neutron_lbaas.confService_providers` 配置，举例如下：

```ini
[service_providers]
# Defines providers for advanced services using the format:
# <service_type>:<name>:<driver>[:default] (multi valued)
service_provider=LOADBALANCERV2:CORE:neutron_lbaas.drivers.f5.v2_CORE.CORE:default
```

- 检查Agent日志错误信息

`grep “ERROR” /var/log/neutron/f5-openstack-agent.log`

- 检查f5-openstack-agent服务

`sudo systemctl status f5-openstack-agent.service \\ CENTOS`
`sudo service f5-oslbaasv2-agent status \\ UBUNTU`

- 确认能连接到BIG-IP，并且密码正确

`ssh admin@<big-ip_mgmt_ip>`

确认 Agent 配置文件中 BIG-IP 信息正确性

- iControl hostname （DNS可解析hostname或者IP地址）
- username (用户有权限配置租户内LTM对象 )
- Password

- 使用Global routed mode时在Agent配置文件中注释掉vtep配置项

```ini
#
#f5_vtep_folder = ‘Common’
#f5_vtep_selfip_name = ‘vtep’
#
```

- 使用L2/L3 分段模式

验证Agent配置文件中advertised_tunnel_types匹配Neutron 网络中的provider:network_type。如果配置不匹配，检查网络配置并修正。


**常见操作问题**

- 创建资源要时候同一个用户，要不会发生 folder/partition not found 问题。
- 只用使用 non-shared network 的时候创建的 Loadbalancer 和 Member 资源才会有 route domain id。
- 确认 CentOS 启动的是正确的 systemd 文件，特别是环境里面有多个 Provider 时候，systemd 文件不一样。

 
**常见配置相关问题**

- 检查和 BIGIP 设备有强相关的配置如：
- `f5_external_physical_mappings = default:1.1:True`，检查 1.1 或者 trunk 在 BIGIP 上存不存，如果不存在 F5 agent 创建不了 vlan，导致创建不了 selfip。
- 多 Provider 的情况下，相同 Provider 所管理的多 agent，所有 agent 的 `agent_id = CORE` 和 `environment_prefix = ‘Project’` 配置要相同。其中 agent_id 为 Provider 名称。如果不一致，会导致错误的那个 agent 消费不到 mq 中的消息，相当于这个 agent 不可以下发配置命令到 BIGIP 上。
-  新版本配置文件有参数的增加或者删除，用已有配置文件替换前需要对比已有配置文件和新配置文件，确认新增或者删除配置参数的更新。

 
**Debug 相关提示**

- 资源创建失败，检查到 F5 agent log 中有 selfip 或者 Vxlan 创建错误，可以再检查 neutron-server log，有无 Port 创建失败或者的 ERROR log。`agent_down_time` 配置合适时间在 neutron.conf 文件中。
- 有 ERROR 出现时，先确认 F5 agent 部署架构，Openstack 和 F5 agent 平台代码版本，问题影响范围，问题准确的复现流程（准确到每个步骤命令和参数）。
- 根据Req id来跟踪整个Log的流程。
- 检查相关的服务状态，比如neutron-server，rabbitMQ，agent等状态。
