# ansible-role-npc_setup

## How to use
```
$ ansible-galaxy install xiaopal.npc_setup

$ cat playbook.yml && ansible-playbook playbook.yml
- hosts: localhost
  gather_facts: 'no'
  vars:
    # 公共配置
    npc_config:
      app_key: <蜂巢APP_KEY>
      app_secret: <蜂巢APP_SECRET>
      # 用于 ansible 管理的 ssh key 名称，自动在蜂巢创建并下载私钥到~/.npc/ssh_key.<name>
      ssh_key: 
        name: npc-setup
      # 默认云主机镜像名称
      default_instance_image: Debian 8.6
      # 默认云主机规格
      default_instance_type:
        cpu: 2
        memory: 4G
    # 云硬盘
    npc_volumes:
      # 定义三块云硬盘，名称分别为 hd-test-gw,hd-test-a,hd-test-b, 容量100G
      - name: hd-test-{gw,{a..b}}
        capacity: 100G
        present: true
    # 云主机
    npc_instances: 
      # 定义云主机 debian-test-gw
      - name: 'debian-test-gw'
        # 挂载云硬盘
        volumes:
          - hd-test-gw
        # 绑定外网IP: 取值可以是具体IP地址或any/new
        # any 绑定任意可用的外网IP，如果没有则新建IP；new 总是创建新IP
        wan_ip: any
        # 外网带宽
        wan_capacity: 100m
        # 作为ansible_host的地址类型：lan_ip私有IP（默认）, wan_ip公网IP
        ssh_host_by: wan_ip
        # ansible主机分组
        groups:
          - gw
          - jump

      # 同时定义两台云主机 debian-test-a 和 debian-test-b
      - name: 'debian-test-w-{a..b}'
        # 挂载云硬盘，其中hd-test-a挂载到debian-test-a， hd-test-b挂载到 debian-test-b
        volumes:
          - '*:hd-test-{a..b}'
        # 使用 debian-test-gw 作为 ssh 跳板机
        ssh_jump_host: debian-test-gw
        # ansible主机分组
        groups:
          - worker
        # ansible主机变量
        vars:
          host_var1: value

  roles: 
    - xiaopal.npc_setup
  tasks:
    - debug: msg={{groups["all"]}}
    - debug: msg={{npc.instances}}
    - debug: msg={{npc.volumes}}

# 配置出站网关（兼跳板机）
- hosts: gw
  tasks:
    - sysctl:
        name: net.ipv4.ip_forward
        value: 1
        sysctl_set: yes
        state: present
        reload: yes
    - iptables:
        table: nat
        chain: POSTROUTING
        source: 10.173.0.0/16
        destination: '10.173.0.0/16'
        jump: ACCEPT
    # 配置SNAT
    - iptables:
        table: nat
        chain: POSTROUTING
        source: 10.173.0.0/16
        jump: SNAT
        to_source: '{{npc_instance.wan_ip}}'
    # 配置所有worker节点的默认网关
    - with_items: '{{groups["worker"]}}'
      shell: route del default; route add default gw {{npc_instance.lan_ip}}
      delegate_to: '{{item}}'
    
# 测试worker节点， 然后还原所有worker节点的默认网关
- hosts: worker
  tasks:
    - ping:
    - apt: name=sudo state=present update_cache=yes
    - setup:
    - shell: route del default; route add default gw 10.173.32.1

# 清理所有主机和硬盘
- hosts: localhost
  gather_facts: 'no'
  vars:
    npc_setup:
      add_hosts: false
    npc_volumes:
      - name: hd-test-{gw,{a..b}}
        present: false
    npc_instances: 
      - name: 'debian-test-gw'
        present: false
      - name: 'debian-test-w-{a..b}'
        present: false
  roles: 
    - xiaopal.npc_setup
```


## npc setup
```
# npc setup --omit-absent "$(jq -n '{
  npc_volumes: [
    {name:"test-hd-1",capacity:"10G"},
    {name:"test-hd-2",capacity:"10G"}
  ],
  npc_instances: [
    {
      name:"test-vm",
      instance_image:"Debian 8.6",
      ssh_keys:["Xiaohui-GRAYPC"], 
      volumes:["test-hd-1","test-hd-2"]
    }
  ]
}')"
```