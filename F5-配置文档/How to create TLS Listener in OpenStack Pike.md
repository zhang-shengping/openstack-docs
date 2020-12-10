# 前置条件
常用配置已经完成，HTTP Listener 能够正常下发，BIG-IP创建成功。
Barbican服务启动和Barbican Client安装成功。

# 更新Neutron 配置文件 
vi /etc/neutron/neutron.conf
```ini 

根据实际环境设置，有的OS支持v2.0，有的是v3，下面是v2.0举例。

[service_auth]
auth_url=http://10.251.13.59:5000/v2.0
admin_user = admin
admin_tenant_name = admin
admin_password=mfaWxhNKjtQ8GfFDf9agr6HsY
auth_version = 2
```

# 更新f5-openstack-agent 配置文件 
vi /etc/neutron/services/f5/f5-openstack-agent.ini
```ini 
cert_manager = f5_openstack_agent.lbaasv2.drivers.bigip.barbican_cert.BarbicanCertManager
 
# Keystone v3 authentication:

auth_version = v3
os_auth_url = http://10.251.13.59:5000/v3
os_username = admin
os_password = mfaWxhNKjtQ8GfFDf9agr6HsY
os_user_domain_name = default
os_project_name = admin
os_project_domain_name = default
 
f5_parent_ssl_profile = clientssl
```

# 重启Neutron Server
systemctl restart neutron-server

# 重启f5-openstack-agent
systemctl restart f5-openstack-agent

# 创建证书
```ini
# Create certificate chain and key

openssl genrsa -des3 -out ca.key 1024
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt  
openssl x509  -in  ca.crt -out ca.pem 
openssl genrsa -des3 -out ca-int_encrypted.key 1024
openssl rsa -in ca-int_encrypted.key -out ca-int.key 
openssl req -new -key ca-int.key -out ca-int.csr -subj "/CN=ca-int@acme.com"
openssl x509 -req -days 3650 -in ca-int.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out ca-int.crt 
openssl genrsa -des3 -out server_encrypted.key 1024
openssl rsa -in server_encrypted.key -out server.key 
openssl req -new -key server.key -out server.csr -subj "/CN=server@acme.com"
openssl x509 -req -days 3650 -in server.csr  -CA ca-int.crt -CAkey ca-int.key -set_serial 01 -out server.crt
```

# Barbican 创建证书
```ini
# create a barbican secret for the certificate
$ barbican secret store --payload-content-type='text/plain' --name='certificate' --payload="$(cat server.crt)" --os-project-id 5fecf9bdd7b84e1eb9f08b14023400c6
+---------------+---------------------------------------------------------------------------+
| Field         | Value                                                                     |
+---------------+---------------------------------------------------------------------------+
| Secret href   | http://10.145.108.36:9311/v1/secrets/91b5da83-a5c9-4646-b46e-a7099f4fed17 |
| Name          | certificate                                                               |
| Created       | None                                                                      |
| Status        | None                                                                      |
| Content types | {u'default': u'text/plain'}                                               |
| Algorithm     | aes                                                                       |
| Bit length    | 256                                                                       |
| Secret type   | opaque                                                                    |
| Mode          | cbc                                                                       |
| Expiration    | None                                                                      |
+---------------+---------------------------------------------------------------------------+
 
# create a barbican secret for the symmetric key
$ barbican secret store --payload-content-type='text/plain' --name='private_key' --payload="$(cat server.key)" --os-project-id 5fecf9bdd7b84e1eb9f08b14023400c6
+---------------+---------------------------------------------------------------------------+
| Field         | Value                                                                     |
+---------------+---------------------------------------------------------------------------+
| Secret href   | http://10.145.108.36:9311/v1/secrets/76629adf-5835-4e5a-88eb-2b6dfed9eb9f |
| Name          | private_key                                                               |
| Created       | None                                                                      |
| Status        | None                                                                      |
| Content types | {u'default': u'text/plain'}                                               |
| Algorithm     | aes                                                                       |
| Bit length    | 256                                                                       |
| Secret type   | opaque                                                                    |
| Mode          | cbc                                                                       |
| Expiration    | None                                                                      |
+---------------+---------------------------------------------------------------------------+

# create a barbican certificate type container
$ barbican secret container create --name='tls_container' --type='certificate' --secret="certificate=http://10.145.108.36:9311/v1/secrets/91b5da83-a5c9-4646-b46e-a7099f4fed17" --secret="private_key=http://10.145.108.36:9311/v1/secrets/76629adf-5835-4e5a-88eb-2b6dfed9eb9f" --os-project-id 5fecf9bdd7b84e1eb9f08b14023400c6
+----------------+------------------------------------------------------------------------------+
| Field          | Value                                                                        |
+----------------+------------------------------------------------------------------------------+
| Container href | http://10.145.108.36:9311/v1/containers/c53d1cfb-4532-41e3-a2a9-af201c42d483 |
| Name           | tls_container                                                                |
| Created        | None                                                                         |
| Status         | ACTIVE                                                                       |
| Type           | certificate                                                                  |
| Certificate    | http://10.145.108.36:9311/v1/secrets/91b5da83-a5c9-4646-b46e-a7099f4fed17    |
| Intermediates  | None                                                                         |
| Private Key    | http://10.145.108.36:9311/v1/secrets/76629adf-5835-4e5a-88eb-2b6dfed9eb9f    |
| PK Passphrase  | None                                                                         |
| Consumers      | None                                                                         |
+----------------+------------------------------------------------------------------------------+
```

# Neutron LBaaSv2 Agent/Driver 创建LB 资源 

```ini
# create a listener (BIGIP virtual server) with a certificate
$ neutron lbaas-listener-create --loadbalancer lb0-0 --protocol-port 443 --protocol terminated-https --name listener0-0 --default-tls-container=http://10.145.108.36:9311/v1/containers/c53d1cfb-4532-41e3-a2a9-af201c42d483
Created a new listener:
+---------------------------+------------------------------------------------------------------------------+
| Field                     | Value                                                                        |
+---------------------------+------------------------------------------------------------------------------+
| admin_state_up            | True                                                                         |
| connection_limit          | -1                                                                           |
| default_pool_id           |                                                                              |
| default_tls_container_ref | http://10.145.108.36:9311/v1/containers/c53d1cfb-4532-41e3-a2a9-af201c42d483 |
| description               |                                                                              |
| id                        | 1b54f885-e6a6-457a-b0ee-f176ac0060f3                                         |
| loadbalancers             | {"id": "ef31abdc-469a-4a25-8f1d-37f905f4d15e"}                               |
| name                      | listener0-0                                                                  |
| protocol                  | HTTPS                                                                        |
| protocol_port             | 443                                                                          |
| sni_container_refs        |                                                                              |
| tenant_id                 | 0860ffb043184028bb22918e48e1cc61                                             |
+---------------------------+------------------------------------------------------------------------------+

BIG-IP 配置下发情况：

# Created a new pool:
(overcloud) [heat-admin@overcloud-controller-0 ~]$ neutron lbaas-pool-create --name 251vlan26th-pool --lb-algorithm ROUND_ROBIN --listener listener0-0 --protocol HTTP

# Created a new healthmonitor:
(overcloud) [heat-admin@overcloud-controller-0 ~]$ neutron lbaas-healthmonitor-create --name vlan26thhm --pool 251vlan26th-pool --type PING --delay 15 --timeout 15 --max-retries 5

#Created a new member:
(overcloud) [heat-admin@overcloud-controller-0 ~]$ neutron lbaas-member-create --name THVLAN26Server-3 --subnet 10.251.26.0/24 --address 10.251.26.108 --protocol-port 80 --weight 5 251vlan26th-pool

```

# 验证Terminated HTTPS功能

访问Listener IP地址：
https://10.251.25.104

查看证书：

访问到后端服务器：

