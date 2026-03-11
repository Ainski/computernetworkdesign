# 医院信息化网络改造设计

## 一、用户需求

### 1. 医院规模

- 三级甲等医院
- 开放床位：800 张

### 2. 网络分区

- 医疗内网：承载 HIS、PACS、LIS 等核心医疗业务系统
- 办公外网：行政、财务、互联网访问等办公业务
- 设备专网：医疗设备、物联网终端、监护设备等专用网络

### 3. 终端数量

- 医生工作站：300 台
- 护士工作站：200 台
- 行政办公电脑：150 台
- 移动医疗终端：100 台

### 4. 安全要求

- 内外网**物理隔离**
- 医疗数据**加密传输**
- 上网行为**审计与管理**

### 5. 可靠性要求

- 核心设备**双机热备**
- 主干链路**冗余备份**
- 满足 **7×24 小时**不间断运行

------

## 二、设计要求

1. 撰写网络改造**可行性分析报告**
2. 设计**三网分离**的网络架构
3. 规划**无线网络覆盖**（病房、门诊区域）
4. 设计**网络安全防护体系**
5. 设备选型与**配置清单**
6. 配置核心交换机 **VRRP、链路聚合**





虚拟环境 Cisco Packet Tracer 7.0.0



# 1 ip分配

| 网络分区 | 用途                               | VLAN ID | 设备数量     | 网段           | 网关          |
| -------- | ---------------------------------- | ------- | ------------ | -------------- | ------------- |
| 医疗内网 | 300台医生工作站                    | 10      | 300          | 10.10.10.0/23  | 10.10.11.254  |
|          | 200台护士工作站                    | 20      | 200          | 10.10.20.0/24  | 10.10.20.254  |
|          | 医疗内网服务器，完成HIS，PACS，LIS | 30      | 1 扩展冗余12 | 10.10.30.0/28  | 10.10.30.14   |
| 办公外网 | 150台行政办公电脑                  | 210     | 150          | 10.10.210.0/24 | 10.10.210.254 |
|          | 外网路由器                         | 220     | 1 扩展冗余12 | 10.10.220.0/28 | 10.10.220.14  |
|          | 财务行政服务器                     | 230     | 1 扩展冗余12 | 10.10.230.0/28 | 10.10.230.14  |
|          | 客户病房无线网络覆盖               | 240     | 4094         | 10.20.0.0/20   | 10.20.15.254  |
| 设备专网 | 100台医疗移动终端                  | 110     | 100          | 10.10.110.0/25 | 10.10.110.126 |
|          | 设备信息服务器                     | 120     | 1 扩展冗余12 | 10.10.120.0/28 | 10.10.120.14  |
| 管理网络 | 网络设备管理                       | 40      |              | 10.10.40.0/24  | 10.10.40.254  |
|          | 无线AP                             | 50      | 85           | 10.10.50.0/24  | 10.10.50.254  |
|          | 设备AP                             | 60      | 85           | 10.10.60.0/24  | 10.10.60.254  |

本设计采用私有IPv4地址空间 `10.0.0.0/8`，并选用 `10.10.0.0/16` 作为核心地址块，同时为无线访客网络单独划分 `10.20.0.0/20`，以避免与内网地址重叠。所有VLAN的划分遵循以下原则：

- 每个VLAN对应一个独立的子网，用于隔离不同业务或管理流量。
- 子网大小根据终端数量、冗余需求和未来扩展进行估算。
- 网关地址通常取子网内最后一个可用IP地址（如 `.254` 或 `.14`），便于记忆和统一管理。
- 管理VLAN（40、50、60）独立于业务VLAN，确保设备管理安全。

以下为各VLAN的详细计算过程：

## 1.1  医疗内网

| VLAN | 用途           | 终端数量 | 子网掩码 | 子网大小 | 网络地址   | 广播地址     | 可用地址范围              | 网关地址     | 计算说明                                                     |
| ---- | -------------- | -------- | -------- | -------- | ---------- | ------------ | ------------------------- | ------------ | ------------------------------------------------------------ |
| 10   | 医生工作站     | 300      | /23      | 512      | 10.10.10.0 | 10.10.11.255 | 10.10.10.1 ~ 10.10.11.254 | 10.10.11.254 | 300台终端需至少300个IP，/23提供512个地址，满足当前并留有约212个扩展空间。网关选用最后一个可用地址。 |
| 20   | 护士工作站     | 200      | /24      | 256      | 10.10.20.0 | 10.10.20.255 | 10.10.20.1 ~ 10.10.20.254 | 10.10.20.254 | 200台终端使用/24（256地址），足够并留有56个冗余。            |
| 30   | 医疗内网服务器 | 1 (+12)  | /28      | 16       | 10.10.30.0 | 10.10.30.15  | 10.10.30.1 ~ 10.10.30.14  | 10.10.30.14  | 服务器初期仅1台，但考虑到冗余和未来可能增加，采用/28提供14个可用地址（除去网络地址和广播地址），网关为最后一个可用地址.14。 |

## 1.2 办公外网

| VLAN | 用途             | 终端数量 | 子网掩码 | 子网大小 | 网络地址    | 广播地址      | 可用地址范围                | 网关地址      | 计算说明                                                     |
| ---- | ---------------- | -------- | -------- | -------- | ----------- | ------------- | --------------------------- | ------------- | ------------------------------------------------------------ |
| 210  | 行政办公电脑     | 150      | /24      | 256      | 10.10.210.0 | 10.10.210.255 | 10.10.210.1 ~ 10.10.210.254 | 10.10.210.254 | 150台终端，/24足够，留有103个扩展。                          |
| 220  | 外网路由器       | 1 (+12)  | /28      | 16       | 10.10.220.0 | 10.10.220.15  | 10.10.220.1 ~ 10.10.220.14  | 10.10.220.14  | 路由器接口只需少数地址，/28提供14个可用，可容纳未来多出口或备用路由器。 |
| 230  | 财务行政服务器   | 1 (+12)  | /28      | 16       | 10.10.230.0 | 10.10.230.15  | 10.10.230.1 ~ 10.10.230.14  | 10.10.230.14  | 同服务器区，预留扩展。                                       |
| 240  | 客户病房无线网络 | 4094     | /20      | 4096     | 10.20.0.0   | 10.20.15.255  | 10.20.0.1 ~ 10.20.15.254    | 10.20.15.254  | 无线终端数量难以精确估算，但为满足大量移动设备接入，使用/20（4096地址），足够覆盖全院访客。网关同样取最后一个可用地址。 |

## 1. 3 设备专网

| VLAN | 用途           | 终端数量 | 子网掩码 | 子网大小 | 网络地址    | 广播地址      | 可用地址范围                | 网关地址      | 计算说明                                              |
| ---- | -------------- | -------- | -------- | -------- | ----------- | ------------- | --------------------------- | ------------- | ----------------------------------------------------- |
| 110  | 医疗移动终端   | 100      | /25      | 128      | 10.10.110.0 | 10.10.110.127 | 10.10.110.1 ~ 10.10.110.126 | 10.10.110.126 | 100台移动终端使用/25提供128地址，满足并留有28个扩展。 |
| 120  | 设备信息服务器 | 1 (+12)  | /28      | 16       | 10.10.120.0 | 10.10.120.15  | 10.10.120.1 ~ 10.10.120.14  | 10.10.120.14  | 服务器区，/28足够。                                   |

## 1.4 管理网络

管理网络独立于业务网络，用于所有网络设备（交换机、路由器、防火墙、无线控制器等）的带外管理。

| VLAN | 用途           | 设备数量 | 子网掩码 | 子网大小 | 网络地址   | 广播地址     | 可用地址范围              | 网关地址     | 计算说明                                                     |
| ---- | -------------- | -------- | -------- | -------- | ---------- | ------------ | ------------------------- | ------------ | ------------------------------------------------------------ |
| 40   | 网络设备管理   | 约80     | /24      | 256      | 10.10.40.0 | 10.10.40.255 | 10.10.40.1 ~ 10.10.40.254 | 10.10.40.254 | 核心、汇聚、接入、防火墙等设备总数约80台，/24可容纳254个设备，满足并留有大量冗余。 |
| 50   | 办公外网AP管理 | 85       | /24      | 256      | 10.10.50.0 | 10.10.50.255 | 10.10.50.1 ~ 10.10.50.254 | 10.10.50.254 | 80台AP + 4台PoE交换机 + 1台WLC = 85个设备，/24足够。         |
| 60   | 设备专网AP管理 | 85       | /24      | 256      | 10.10.60.0 | 10.10.60.255 | 10.10.60.1 ~ 10.10.60.254 | 10.10.60.254 | 同理，85个设备，/24满足。                                    |

## 1.5 地址分配说明

- **网关位置**：统一采用子网内最后一个可用IP作为网关（如 `/24` 的 `.254`，`/28` 的 `.14`，`/25` 的 `.126`），便于记忆和脚本配置。
- **地址范围预留**：每个子网的第一个可用地址（如 `.1`）通常分配给服务器或关键设备，其余地址通过DHCP动态分配。静态设备（如网络设备管理地址）在对应管理VLAN内连续分配，避免与动态池冲突。
- **扩展性**：所有子网均预留了至少20%的冗余地址，部分子网（如医生工作站/23）预留更多，以适应未来医院信息化发展。

# 2 网络拓扑设计

## 2.1 设备清单

| 设备名称                | 型号            | 数量 | 所属网络 | 角色                                                   | IP地址/管理地址                                              |
| :---------------------- | :-------------- | :--- | :------- | :----------------------------------------------------- | :----------------------------------------------------------- |
| Core-Int-1 / Core-Int-2 | Cisco 3650-24PS | 2    | 医疗内网 | 核心交换机（VRRP、DHCP、三层网关）                     | 10.10.40.1, 10.10.40.2（管理VLAN 40）                        |
| Dist-Int-Hospital-1/2   | Cisco 3560-24PS | 2    | 医疗内网 | 汇聚交换机 护士用 4 （**带宽扩展 + 冗余**）+ 14        | 10.10.40.3 10.10.40.4（管理VLAN 40）                         |
| Dist-Int-OPD-1/2        | Cisco 3560-24PS | 2    | 医疗内网 | 汇聚交换机 医生用 4 + 14                               | 10.10.40.5 10.10.40.6（管理VLAN 40）                         |
| Acc-Int-Doctor          | Cisco 2960-24TT | 14   | 医疗内网 | 医生工作站接入14 * 22 = 308 1个端口作上行通讯          | 10.10.40.7-20（管理VLAN 40，每台一个）                       |
| Acc-Int-Nurse           | Cisco 2960-24TT | 10   | 医疗内网 | 护士工作站接入 10 * 22 = 207 1个端口作上行通讯         | 10.10.40.21-30（管理VLAN 40）                                |
| Server-Int-Switch       | Cisco 2960-24TT | 1    | 医疗内网 | 服务器接入交换机                                       | 10.10.40.32（管理VLAN 40）<br>密码cisco                      |
| Int-Server              | Server-PT       | 1    | 医疗内网 | 医疗内网服务器                                         | 10.10.30.2（业务VLAN 30）                                    |
| Docter-Computer         | PC              | 1    | 医疗内网 | 医生电脑                                               | dhcp 获取 （文件中为10.10.10.4）                             |
| Nurse-Computer          | PC              | 1    | 医疗内网 | 护士电脑                                               | dhcp 获取 （文件中为10.10.20.3）                             |
| Core-Ext-1 / Core-Ext-2 | Cisco 3650-24PS | 2    | 办公外网 | 核心交换机（VRRP、DHCP、三层网关）                     | 10.10.40.33, 10.10.40.34（管理VLAN 40）                      |
| Dist-Ext-Admin-1/2      | Cisco 3560-24PS | 2    | 办公外网 | 行政楼汇聚交换机                                       | 10.10.40.35 10.10.40.36（管理VLAN40）                        |
| Acc-Ext-Office          | Cisco 2960-24TT | 7    | 办公外网 | 行政办公接入8*22 = 156                                 | 10.10.40.37-44（管理VLAN 40）                                |
| Dist-Ext-Wireless-1/2   | Cisco 3560-24PS | 2    | 办公外网 | 无线汇聚交换机 4端口上行，1端口wlc ，4端口ap接入交换机 | 10.10.40.45 10.10.40.46（管理VLAN 40）                       |
| Dist-Ext-WLC-1/2        | Cisco WLC-2504  | 2    | 办公外网 | 无线网络控制器                                         | 10.10.50.1（管理VLAN 50）10.10.50.86                         |
| Acc-Ext-AP-Switch       | Cisco 2960-24PS | 4    | 办公外网 | AP 接入交换机（PoE 供电）                              | 10.10.50.2, 10.10.50.3, 10.10.50.4, 10.10.50.5（管理VLAN 50） |
| Acc-Ext-AP              | Cisco 3702i     | 80   | 办公外网 | 无线网络接入点                                         | 10.10.50.6 ~ 10.10.50.85（管理VLAN 50，每AP一个）            |
| Switch-Ext-Server       | Cisco 2960-24TT | 1    | 办公外网 | 服务器接入交换机                                       | 10.10.40.56（管理VLAN 40）                                   |
| Server-Ext-FW           | ASA 5506        | 1    | 办公外网 | 服务器及出口防火墙                                     | 10.10.40.47（管理VLAN 40）                                   |
| Switch-Ext-Router       | Cisco 2960-24TT | 1    | 办公外网 | 路由器接入交换机                                       | 10.10.40.57（管理VLAN 40）                                   |
| Ext-Router              | Cisco 2911      | 1    | 办公外网 | 互联网出口路由器                                       | 10.10.40.48（管理VLAN 40）                                   |
| Ext-Syslog              | Server-PT       | 1    | 办公外网 | 上网行为审计                                           | 10.10.40.58（管理VLAN 40）                                   |
| Switch-Out              | Cisco 2960-24TT |      |          | 外网交换机                                             | 20.20.20.1                                                   |
| PC-Out                  |                 |      |          | 外网ip                                                 | 20.20.20.2                                                   |
| PC-MNG-Ext              |                 |      | 办公外网 |                                                        | 10.10.210.3                                                  |
| Finance-Server          | Server-PT       | 1    | 办公外网 | 财务行政服务器                                         | 10.10.230.2（业务VLAN 230）                                  |
| Core-Dev-1/2            | Cisco 3650-24PS | 2    | 设备专网 | 设备专网核心                                           | 10.10.40.49 10.10.40.50（管理VLAN 40）                       |
| Dist-Dev-1/2            | Cisco 3560-24PS | 2    | 设备专网 | 设备汇聚交换机                                         | 10.10.40.51（管理VLAN 40）10.10.40.52                        |
| Dist-Dev-WLC-1/2        | Cisco WLC-2504  | 2    | 设备专网 | 移动设备无线网络控制器                                 | 10.10.60.1（管理VLAN 60） 10.10.60.86                        |
| Acc-Dev-AP-Switch       | Cisco 2960-24PS | 4    | 设备专网 | 移动设备AP 接入交换机（PoE 供电）                      | 10.10.60.2, 10.10.60.3, 10.10.60.4, 10.10.60.5（管理VLAN 60） |
| Acc-Dev-AP              | Cisco 3702i     | 80   | 设备专网 | 移动设备无线网络接入点                                 | 10.10.60.6 ~ 10.10.60.85（管理VLAN 60，每AP一个）            |
| Server-Dev-FW           | ASA 5506        | 1    | 设备专网 | 服务器区防火墙                                         | 10.10.40.53（管理VLAN 40）                                   |
| Server-Dev-Switch       | Cisco 2960-24TT | 1    | 设备专网 | 服务器接入交换机                                       | 10.10.40.54（管理VLAN 40）                                   |
| Dev-Server              | Server-PT       | 1    | 设备专网 | 设备专网服务器                                         | 10.10.120.2（设备VLAN 120）                                  |
| Syslog-Server           | Server-PT       | 1    |          | 日志审计服务器                                         | 10.10.40.55（VLAN 40）                                       |



```
!3650 重置端口所有配置
conf t
def int g0/1
def int g0/2
show int g 0/1
show int g 0/2
```



# 3 各个服务器的配置

## 3.1 医疗内网

### 3.1.1 核心层链路配置

#### 3.1.1.1 Core-Int-1 

```bash
enable
configure terminal

hostname Core-Int-1
no ip domain-lookup 


vlan 10
 name Doctor_Workstation
exit
vlan 20
 name Nurse_Workstation
exit
vlan 30
 name Medical_Server
exit
vlan 40
 name Management
exit


interface range GigabitEthernet1/0/1 - 2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode active
 no shutdown
exit

interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan all
 no shutdown
exit

interface Vlan40
 ip address 10.10.40.1 255.255.255.0
 no shutdown
exit

! ========== VLAN 10（医生工作站）- VRRP 配置 ==========

interface Vlan10
ip address 10.10.10.1 255.255.254.0
no shutdown
standby 10 ip 10.10.11.254
standby 10 priority 120 
standby 10 preempt
exit

! ========== VLAN 20（护士工作站）- VRRP 配置 ==========
interface Vlan20
 ip address 10.10.20.1 255.255.255.0
 no shutdown
 standby 20 ip 10.10.20.254
 standby 20 priority 120
 standby 20 preempt
exit

! ========== VLAN 30（医疗内网服务器）- VRRP 配置 ==========
interface Vlan30
 ip address 10.10.30.1 255.255.255.240
 no shutdown
 standby 30 ip 10.10.30.14
 standby 30 priority 120
 standby 30 preempt
exit

! 启用三层路由（核心交换机必备）
ip routing

! dhcp vlan10 docter

service dhcp
ip dhcp pool VLAN10-Doctor
network 10.10.10.0 255.255.254.0  
default-router 10.10.11.254
ip dhcp excluded-address 10.10.10.1
ip dhcp excluded-address 10.10.10.2
ip dhcp excluded-address 10.10.11.254

! dhcp vlan 20 nurse

ip dhcp pool VLAN20-Nurse
network 10.10.20.0 255.255.255.0
default-router 10.10.20.254

ip dhcp excluded-address 10.10.20.1
ip dhcp excluded-address 10.10.20.2
ip dhcp excluded-address 10.10.20.254


 
! 保存配置
exit
write memory
```



#### 3.1.1.2 Core-Int-2

```bash
enable
configure terminal

hostname Core-Int-2
no ip domain-lookup

! 管理VLAN
vlan 10
 name Doctor_Workstation
exit
vlan 20
 name Nurse_Workstation
exit
vlan 30
 name Medical_Server
exit
vlan 40
 name Management
exit

! 双核心互联 - 链路聚合
interface range GigabitEthernet1/0/1 - 2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode active
 no shutdown
exit

interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shutdown
exit

! 管理地址
interface Vlan40
 ip address 10.10.40.2 255.255.255.0
 no shutdown
exit

! ========== VLAN 10 - VRRP 配置（备核心） ==========
interface Vlan10
 ip address 10.10.10.2 255.255.254.0
 no shutdown
 standby 10 ip 10.10.11.254
 standby 10 priority 100
 standby 10 preempt
exit

! ========== VLAN 20 - VRRP 配置（备核心） ==========
interface Vlan20
 ip address 10.10.20.2 255.255.255.0
 no shutdown
 standby 20 ip 10.10.20.254
 standby 20 priority 100
 standby 20 preempt
exit

! ========== VLAN 30 - VRRP 配置（备核心） ==========
interface Vlan30
 ip address 10.10.30.2 255.255.255.240
 no shutdown
 standby 30 ip 10.10.30.14
 standby 30 priority 100
 standby 30 preempt
exit

! 启用三层路由
ip routing

! dhcp

ip dhcp pool VLAN10-Doctor-Backup
 network 10.10.10.0 255.255.254.0
 default-router 10.10.11.254
 lease 7 0 0

ip dhcp excluded-address 10.10.10.1
ip dhcp excluded-address 10.10.10.2
ip dhcp excluded-address 10.10.11.254

ip dhcp pool VLAN20-Nurse-Backup
 network 10.10.20.0 255.255.255.0
 default-router 10.10.20.254
 lease 7 0 0

ip dhcp excluded-address 10.10.20.1
ip dhcp excluded-address 10.10.20.2
ip dhcp excluded-address 10.10.20.254

interface Vlan10
 ip helper-address 10.10.10.1
exit
interface Vlan20
 ip helper-address 10.10.20.1
exit


service dhcp
! 医生工作站备用地址池
ip dhcp pool VLAN10-Doctor-Backup
 network 10.10.10.0 255.255.254.0
 default-router 10.10.11.254
exit

ip dhcp excluded-address 10.10.10.1
ip dhcp excluded-address 10.10.10.2
ip dhcp excluded-address 10.10.11.254

! 护士工作站备用地址池
ip dhcp pool VLAN20-Nurse-Backup
 network 10.10.20.0 255.255.255.0
 default-router 10.10.20.254
exit

ip dhcp excluded-address 10.10.20.1
ip dhcp excluded-address 10.10.20.2
ip dhcp excluded-address 10.10.20.254

! DHCP中继
interface Vlan10
 ip helper-address 10.10.10.1
exit
interface Vlan20
 ip helper-address 10.10.20.1
exit

! 保存配置
exit
write memory
```

#### 3.1.1.3 配置验证

验证vrrp  配置

```bash
!Core Int 1
Core-Int-1#show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   120 P Active   local           10.10.10.2      10.10.11.254   
Vl20        20   120 P Active   local           10.10.20.2      10.10.20.254   
Vl30        30   120 P Active   local           10.10.30.2      10.10.30.14    
Vl40        40   120 P Active   local           10.10.40.2      10.10.40.254   
Core-Int-1#
```

验证dhcp设置

```
Core-Int-1#show ip dhcp pool

Pool VLAN10-Doctor :
 Utilization mark (high/low)    : 100 / 0
 Subnet size (first/next)       : 0 / 0 
 Total addresses                : 510
 Leased addresses               : 2
 Excluded addresses             : 6
 Pending event                  : none

 1 subnet is currently in the pool
 Current index        IP address range                    Leased/Excluded/Total
 10.10.10.1           10.10.10.1       - 10.10.11.254      2    / 6     / 510

Pool VLAN20-Nurse :
 Utilization mark (high/low)    : 100 / 0
 Subnet size (first/next)       : 0 / 0 
 Total addresses                : 254
 Leased addresses               : 0
 Excluded addresses             : 6
 Pending event                  : none

 1 subnet is currently in the pool
 Current index        IP address range                    Leased/Excluded/Total
 10.10.20.1           10.10.20.1       - 10.10.20.254      0    / 6     / 254
Core-Int-1#
```
```
! 使用1/0/7接口测试
int g 1/0/7 
sw mode acc
sw acc vlan 10
no sh
```

![image-20260311012011387](./images/image-20260311012011387.png)

可以看到，图片中的pc2 成功获取到了dhcp分配的地址。

### 3.1.2 门诊汇聚层

#### 3.1.2.1 Core-Int-1 补充配置

```
enable
configure terminal

! ========== 1. 对接 Dist-Int-OPD-1 的接口（G1/0/3） ==========
interface GigabitEthernet1/0/3
 switchport trunk encapsulation dot1q 
 switchport mode trunk
 switchport trunk allowed vlan 10,40
 no shutdown
exit

! ========== 2. 对接 Dist-Int-OPD-2 的接口（G1/0/4） ==========
interface GigabitEthernet1/0/4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,40
 no shutdown
exit

! ========== 3. 配置管理VLAN 40的虚拟网关（HSRP） ==========
! 核心层管理VLAN需要虚拟网关，确保汇聚层运维可达
interface Vlan40
 standby 40 ip 10.10.40.254    
 ! 管理VLAN虚拟网关（汇聚层默认网关指向此地址）
 standby 40 priority 120       
 ! Core-Int-1作为主网关
 standby 40 preempt            
 ! 抢占模式
exit

! ========== 4. 启用生成树（防环路，医院网络必备） ==========
spanning-tree mode rapid-pvst        
! 快速生成树，秒级故障切换
spanning-tree vlan 10,40 root primary 
! Core-Int-1作为VLAN10/40的主根桥

! 保存配置
exit
write memory
```

#### 3.1.2.2 Core-Int-2 补充配置

```
enable
configure terminal

! ========== 1. 对接 Dist-Int-OPD-1 的接口（G1/0/3） ==========
interface GigabitEthernet1/0/3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,40
 no shutdown
exit

! ========== 2. 对接 Dist-Int-OPD-2 的接口（G1/0/4） ==========
interface GigabitEthernet1/0/4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,40
 no shutdown
exit

! ========== 3. 配置管理VLAN 40的虚拟网关（HSRP） ==========
interface Vlan40
 standby 40 ip 10.10.40.254    
 ! 和Core-Int-1一致的虚拟网关
 standby 40 priority 100       
 ! Core-Int-2作为备网关
 standby 40 preempt            
 ! 抢占模式
exit

! ========== 4. 启用生成树 ==========
spanning-tree mode rapid-pvst
spanning-tree vlan 10,40 root secondary  
! Core-Int-2作为备用根桥

! 保存配置
exit
write memory
```

#### 3.1.2.3 Dist-Int-OPD-1 配置

```
enable
configure terminal

hostname Dist-Int-OPD-1
no ip domain-lookup

vlan 10
name Doctor_Workstation
exit

vlan 40
name Management
exit

interface FastEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,40
 no shutdown
exit

interface FastEthernet0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,40
 no shutdown
exit

interface Vlan40
 ip address 10.10.40.5 255.255.255.0
 no shutdown
exit

ip default-gateway 10.10.40.254

interface range FastEthernet0/3 - 24
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,40
 no shutdown
exit

spanning-tree mode rapid-pvst
exit
write memory
```

#### 3.1.2.4 Dist-Int-OPD-2 配置

```
enable
configure terminal

hostname Dist-Int-OPD-2
no ip domain-lookup

vlan 10
name Doctor_Workstation
exit

vlan 40
name Management
exit

interface FastEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,40
 no shutdown
exit

interface FastEthernet0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,40
 no shutdown
exit

interface Vlan40
 ip address 10.10.40.6 255.255.255.0
 no shutdown
exit

ip default-gateway 10.10.40.254

interface range FastEthernet0/3 - 24
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,40
 no shutdown
exit

spanning-tree mode rapid-pvst
exit
write memory
```



#### 3.1.2.5 配置验证

```bash
! ==============Dist-Int-OPD-1 ping 核心================

Dist-Int-OPD-1#ping 10.10.40.1

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.40.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 0/9/30 ms

Dist-Int-OPD-1#ping 10.10.40.2

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.40.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 0/5/25 ms




Dist-Int-OPD-1#show int tr
Port        Mode         Encapsulation  Status        Native vlan
Fa0/1       on           802.1q         trunking      1
Fa0/2       on           802.1q         trunking      1
Fa0/3       on           802.1q         trunking      1
Fa0/4       on           802.1q         trunking      1

Port        Vlans allowed on trunk
Fa0/1       10,40
Fa0/2       10,40
Fa0/3       10,40
Fa0/4       10,40

Port        Vlans allowed and active in management domain
Fa0/1       10,40
Fa0/2       10,40
Fa0/3       10,40
Fa0/4       10,40

Port        Vlans in spanning tree forwarding state and not pruned
Fa0/1       10,40
Fa0/2       10
Fa0/3       10,40
Fa0/4       10,40

Dist-Int-OPD-1#show spanning-tree vlan 10
VLAN0010
  Spanning tree enabled protocol rstp
  Root ID    Priority    24586
             Address     0001.C93B.4029
             Cost        19
             Port        1(FastEthernet0/1)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    28682  (priority 28672 sys-id-ext 10)
             Address     0001.C960.EC57
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/3            Desg FWD 19        128.3    P2p
Fa0/2            Root FWD 19        128.2    P2p
Fa0/1            Root FWD 19        128.1    P2p
Fa0/4            Desg FWD 19        128.4    P2p

Dist-Int-OPD-1#show spanning-tree vlan 40
VLAN0040
  Spanning tree enabled protocol rstp
  Root ID    Priority    24616
             Address     0001.C93B.4029
             Cost        19
             Port        1(FastEthernet0/1)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    28712  (priority 28672 sys-id-ext 40)
             Address     0001.C960.EC57
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/3            Desg FWD 19        128.3    P2p
Fa0/2            Altn BLK 19        128.2    P2p
Fa0/1            Root FWD 19        128.1    P2p
Fa0/4            Desg FWD 19        128.4    P2p

Dist-Int-OPD-1#



Dist-Int-OPD-1#show interfaces status
Port      Name               Status       Vlan       Duplex  Speed Type
Fa0/1                        connected    1          auto    auto  10/100BaseTX
Fa0/2                        connected    1          auto    auto  10/100BaseTX
Fa0/3                        connected    1          auto    auto  10/100BaseTX
Fa0/4                        connected    1          auto    auto  10/100BaseTX
Fa0/5                        notconnect   1          auto    auto  10/100BaseTX
Fa0/6                        notconnect   1          auto    auto  10/100BaseTX
Fa0/7                        notconnect   1          auto    auto  10/100BaseTX
Fa0/8                        notconnect   1          auto    auto  10/100BaseTX
Fa0/9                        notconnect   1          auto    auto  10/100BaseTX
Fa0/10                       notconnect   1          auto    auto  10/100BaseTX
Fa0/11                       notconnect   1          auto    auto  10/100BaseTX
Fa0/12                       notconnect   1          auto    auto  10/100BaseTX
Fa0/13                       notconnect   1          auto    auto  10/100BaseTX
Fa0/14                       notconnect   1          auto    auto  10/100BaseTX
Fa0/15                       notconnect   1          auto    auto  10/100BaseTX
Fa0/16                       notconnect   1          auto    auto  10/100BaseTX
Fa0/17                       notconnect   1          auto    auto  10/100BaseTX
Fa0/18                       notconnect   1          auto    auto  10/100BaseTX
Fa0/19                       notconnect   1          auto    auto  10/100BaseTX
Fa0/20                       notconnect   1          auto    auto  10/100BaseTX
Fa0/21                       notconnect   1          auto    auto  10/100BaseTX
Fa0/22                       notconnect   1          auto    auto  10/100BaseTX
Fa0/23                       notconnect   1          auto    auto  10/100BaseTX
Fa0/24                       notconnect   1          auto    auto  10/100BaseTX
Gig0/1                       notconnect   1          auto    auto  10/100BaseTX
Gig0/2                       notconnect   1          auto    auto  10/100BaseTX

Dist-Int-OPD-1#
Dist-Int-OPD-1#


Dist-Int-OPD-1#ping 10.10.11.254

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.11.254, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 0/18/37 ms

Dist-Int-OPD-1#
```

验证dhcp

```
!Dist-Int-OPD-1
conf t
int fa 0/5
sw mode acc
sw acc vlan 10
no sh
exit
```

![image-20260311184858306](./images/image-20260311184858306.png)



### 3.1.3 门诊接入层

#### 3.1.3.1 Acc-Int-Doctor-1

```
enable
configure terminal

! 基础配置
hostname Acc-Int-Doctor-1
no ip domain-lookup

! 创建业务VLAN和管理VLAN
vlan 10
 name Doctor_Workstation
exit
vlan 40
 name Management
exit

! ========== 上联 Dist-Int-OPD-1（Fa0/3） ==========
interface FastEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,40  
 no shutdown
exit

! ========== 上联 Dist-Int-OPD-2（Fa0/3） ==========
interface FastEthernet0/2
 switchport mode trunk
 switchport trunk allowed vlan 10,40
 no shutdown
exit

! ========== 管理VLAN配置（运维用） ==========
interface Vlan40
 ip address 10.10.40.7 255.255.255.0  
 no shutdown
exit

! 管理网关指向核心虚拟网关
ip default-gateway 10.10.40.254

! ========== 下联医生工作站PC（所有终端接口） ==========
interface range FastEthernet0/3 - 24
 switchport mode access 
 switchport access vlan 10
 no shutdown
exit

! ========== 生成树（防环路，冗余必备） ==========
spanning-tree mode rapid-pvst

! 保存配置
exit
write memory
```



#### 3.1.3.2 配置验证

```
Acc-Int-Doctor-1#show int trunk
Port        Mode         Encapsulation  Status        Native vlan
Fa0/1       on           802.1q         trunking      1
Fa0/2       on           802.1q         trunking      1

Port        Vlans allowed on trunk
Fa0/1       10,40
Fa0/2       10,40

Port        Vlans allowed and active in management domain
Fa0/1       10,40
Fa0/2       10,40

Port        Vlans in spanning tree forwarding state and not pruned
Fa0/1       40
Fa0/2       none

Acc-Int-Doctor-1#
```

Fa 0/2 是防止环路的行为，none是正常的结果

dhcp ip获取正常

![image-20260311190139978](./images/image-20260311190139978.png)

![image-20260311190239785](./images/image-20260311190239785.png)

网关均能ping通

### 3.1.4 护士工作站汇聚层
#### 3.1.4.1 Core-Int-1 补充配置（对接护士汇聚层）
```
enable
configure terminal

! ========== 1. 对接 Dist-Int-Hospital-1 的接口（G1/0/5） ==========
interface GigabitEthernet1/0/5
 switchport trunk encapsulation dot1q 
 switchport mode trunk
 switchport trunk allowed vlan 20,40
 no shutdown
exit

! ========== 2. 对接 Dist-Int-Hospital-2 的接口（G1/0/6） ==========
interface GigabitEthernet1/0/6
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 20,40
 no shutdown
exit

! ========== 3. 生成树强化配置（护士VLAN 20） ==========
spanning-tree vlan 20 root primary  
! Core-Int-1作为VLAN20主根桥

! 保存配置
exit
write memory
```

#### 3.1.4.2 Core-Int-2 补充配置（对接护士汇聚层）
```
enable
configure terminal

! ========== 1. 对接 Dist-Int-Hospital-1 的接口（G1/0/5） ==========
interface GigabitEthernet1/0/5
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 20,40
 no shutdown
exit

! ========== 2. 对接 Dist-Int-Hospital-2 的接口（G1/0/6） ==========
interface GigabitEthernet1/0/6
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 20,40
 no shutdown
exit

! ========== 3. 生成树强化配置（护士VLAN 20） ==========
spanning-tree vlan 20 root secondary  
! Core-Int-2作为VLAN20备用根桥

! 保存配置
exit
write memory
```

#### 3.1.4.3 Dist-Int-Hospital-1 配置（护士汇聚层1）
```
enable
configure terminal

hostname Dist-Int-Hospital-1
no ip domain-lookup

vlan 20
name Nurse_Workstation
exit

vlan 40
name Management
exit

interface FastEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 20,40
 no shutdown
exit

interface FastEthernet0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 20,40
 no shutdown
exit

interface Vlan40
 ip address 10.10.40.3 255.255.255.0
 no shutdown
exit

ip default-gateway 10.10.40.254

interface range FastEthernet0/3 - 24
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 20,40
 no shutdown
exit

spanning-tree mode rapid-pvst
exit
write memory
```

#### 3.1.4.4 Dist-Int-Hospital-2 配置（护士汇聚层2）
```
enable
configure terminal

hostname Dist-Int-Hospital-2
no ip domain-lookup

vlan 20
name Nurse_Workstation
exit

vlan 40
name Management
exit

interface FastEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 20,40
 no shutdown
exit

interface FastEthernet0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 20,40
 no shutdown
exit

interface Vlan40
 ip address 10.10.40.4 255.255.255.0
 no shutdown
exit

ip default-gateway 10.10.40.254

interface range FastEthernet0/3 - 24
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 20,40
 no shutdown
exit

spanning-tree mode rapid-pvst
exit
write memory
```

#### 3.1.4.5 配置验证

```
Core-Int-1#show interfaces trunk
Port        Mode         Encapsulation  Status        Native vlan
Po1         on           802.1q         trunking      1
Gig1/0/3    on           802.1q         trunking      1
Gig1/0/4    on           802.1q         trunking      1
Gig1/0/5    on           802.1q         trunking      1
Gig1/0/6    on           802.1q         trunking      1

Port        Vlans allowed on trunk
Po1         1-1005
Gig1/0/3    10,40
Gig1/0/4    10,40
Gig1/0/5    20,40
Gig1/0/6    20,40

Port        Vlans allowed and active in management domain
Po1         1,10,20,30,40
Gig1/0/3    10,40
Gig1/0/4    10,40
Gig1/0/5    20,40
Gig1/0/6    20,40

Port        Vlans in spanning tree forwarding state and not pruned
Po1         1,10,20,30,40
Gig1/0/3    10,40
Gig1/0/4    10,40
Gig1/0/5    20,40
Gig1/0/6    20,40

Core-Int-1#show spanning-tree vlan 20
VLAN0020
  Spanning tree enabled protocol rstp
  Root ID    Priority    24596
             Address     0001.C93B.4029
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    24596  (priority 24576 sys-id-ext 20)
             Address     0001.C93B.4029
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Gi1/0/4          Desg FWD 19        128.4    P2p
Gi1/0/3          Desg FWD 19        128.3    P2p
Gi1/0/5          Desg FWD 19        128.5    P2p
Gi1/0/6          Desg FWD 19        128.6    P2p
Po1              Desg FWD 3         128.29   Shr
```

```
Dist-Int-Hospital-1#show interfaces trunk
Port        Mode         Encapsulation  Status        Native vlan
Fa0/1       on           802.1q         trunking      1
Fa0/2       on           802.1q         trunking      1
Fa0/3       on           802.1q         trunking      1

Port        Vlans allowed on trunk
Fa0/1       20,40
Fa0/2       20,40
Fa0/3       20,40

Port        Vlans allowed and active in management domain
Fa0/1       20,40
Fa0/2       20,40
Fa0/3       20,40

Port        Vlans in spanning tree forwarding state and not pruned
Fa0/1       20,40
Fa0/2       none
Fa0/3       20,40

Dist-Int-Hospital-1#ping 10.10.40.254

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.40.254, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 0/0/0 ms

Dist-Int-Hospital-1#ping 10.10.20.254

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.20.254, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 0/0/0 ms

Dist-Int-Hospital-1#ping 10.10.40.1

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.40.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 0/0/1 ms
```

### 3.1.5 护士工作站接入层
#### 3.1.5.1 Acc-Int-Nurse-1
```
enable
configure terminal

! 基础配置
hostname Acc-Int-Nurse-1
no ip domain-lookup

! 创建业务VLAN和管理VLAN
vlan 20
 name Nurse_Workstation
exit
vlan 40
 name Management
exit

! ========== 上联 Dist-Int-Hospital-1（Fa0/3） ==========
interface FastEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 20,40  
 no shutdown
exit

! ========== 上联 Dist-Int-Hospital-2（Fa0/3） ==========
interface FastEthernet0/2
 switchport mode trunk
 switchport trunk allowed vlan 20,40
 no shutdown
exit

! ========== 管理VLAN配置（运维用） ==========
interface Vlan40
 ip address 10.10.40.21 255.255.255.0  
 no shutdown
exit

! 管理网关指向核心虚拟网关
ip default-gateway 10.10.40.254

! ========== 下联护士工作站PC（所有终端接口） ==========
interface range FastEthernet0/3 - 24
 switchport mode access 
 switchport access vlan 20
 no shutdown
exit

! ========== 生成树（防环路，冗余必备） ==========
spanning-tree mode rapid-pvst

! 保存配置
exit
write memory
```

#### 3.1.5.2 快速验证命令（配置后执行）

```bash
Acc-Int-Nurse-1#show interfaces trunk
Port        Mode         Encapsulation  Status        Native vlan
Fa0/1       on           802.1q         trunking      1
Fa0/2       on           802.1q         trunking      1

Port        Vlans allowed on trunk
Fa0/1       20,40
Fa0/2       20,40

Port        Vlans allowed and active in management domain
Fa0/1       20,40
Fa0/2       20,40

Port        Vlans in spanning tree forwarding state and not pruned
Fa0/1       20,40
Fa0/2       none

Acc-Int-Nurse-1#show ip interface brief | include Vlan40
Vlan40                 10.10.40.21     YES manual up                    up


Acc-Int-Nurse-1#show interfaces FastEthernet0/3 switchport | include Access Mode VLAN
Access Mode VLAN: 20 (Nurse_Workstation)

```

![image-20260311201638501](./images/image-20260311201638501.png)

护士电脑能正确入网，dhcp获取ip地址，并能正确地和医生通讯





### 3.1.6 服务器

#### 3.1.6.1 Core-Int-1 补充配置

```
enable
configure terminal

! 对接防火墙GigabitEthernet1/1的接口（G1/0/7）
interface GigabitEthernet1/0/7
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 30,40 
 no shutdown
exit

! VLAN30主根桥（服务器网段生成树主节点）
spanning-tree vlan 30 root primary

! 保存配置
exit
write memory
```

#### 3.1.6.2 Core-Int-2 补充配置

```
enable
configure terminal

! 对接防火墙GigabitEthernet1/2的接口（G1/0/7）
interface GigabitEthernet1/0/7
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 30,40
 no shutdown
exit

! VLAN30备用根桥（核心故障时自动切换）
spanning-tree vlan 30 root secondary

! 保存配置

exit
write memory
```

#### 3.1.6.3 Server-Int-Switch

```
! 进入特权模式
enable
configure terminal

hostname Server-Int-Switch
no ip domain-lookup

! ===================== 2. VLAN配置（匹配医疗内网规划） =====================
! 医疗服务器业务VLAN
vlan 30
 name Medical_Server
exit

! 设备管理VLAN（与核心/防火墙统一）
vlan 40
 name Management
exit

! ===================== 3. 上联核心交换机接口配置（冗余连通） =====================
! Fa0/1 对接 Core-Int-1 的 G1/0/7
interface FastEthernet0/1
 switchport mode trunk 

 switchport trunk allowed vlan 30,40
 no shutdown
exit

! Fa0/2 对接 Core-Int-2 的 G1/0/7（冗余备份）
interface FastEthernet0/2
 switchport mode trunk
 switchport trunk allowed vlan 30,40
 no shutdown
exit

! ===================== 4. 服务器接入端口配置（核心接入） =====================
interface range FastEthernet0/3 - 24
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 no shutdown
exit

! ===================== 5. 管理VLAN配置（保障设备运维） =====================
interface Vlan40
 ip address 10.10.40.32 255.255.255.0
 no shutdown
exit

! 默认网关（指向核心层VRRP虚拟网关，保障冗余）
ip default-gateway 10.10.40.254

! ===================== 6. 生成树配置（防环路，保障冗余） =====================
spanning-tree mode rapid-pvst
spanning-tree vlan 30,40 priority 24576

! ===================== 7. 保存配置（防止丢失） =====================
exit
write memory
```

#### 3.1.6.3 测试配置

```
Server-Int-Switch#ping 10.10.40.254

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.40.254, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 0/0/1 ms

Server-Int-Switch#
```

服务器ping网关

![image-20260311231858058](./images/image-20260311231858058.png)

护士电脑ping服务器

![image-20260311232314372](./images/image-20260311232314372.png)

### 3.1.7 核心服务器配置acl

#### 3.1.7.1 Core-Int-1

```
enable
configure terminal

! ========== 1. 定义极简ACL（放行所有IP流量） ==========
ip access-list extended MEDICAL_SERVER_ACL
 permit ip any any 
exit

! ========== 2. 应用到服务器VLAN三层接口（Vlan30） ==========
interface Vlan30
 ip access-group MEDICAL_SERVER_ACL in
exit

! ========== 3. 保存配置 ==========
exit
write memory
```

#### 3.1.7.2 Core-Int-2

```
enable
configure terminal

! ========== 1. 同步极简ACL配置 ==========
ip access-list extended MEDICAL_SERVER_ACL
 permit ip any any
exit

! ========== 2. 应用到Vlan30接口 ==========
interface Vlan30
 ip access-group MEDICAL_SERVER_ACL in
exit

! ========== 3. 保存配置 ==========
exit
write memory
```

## 3.2 政务外网

### 3.2.1 核心层配置

#### 3.2.1.1 Core-Ext-1 

```
enable
configure terminal

hostname Core-Ext-1
no ip domain-lookup 

! 创建办公外网相关VLAN
vlan 210
 name Admin_Office
exit
vlan 220
 name Ext_Router
exit
vlan 230
 name Finance_Server
exit
vlan 240
 name Guest_Wireless
exit
vlan 40
 name Management
exit
vlan 50
 name AP_Management_Ext
exit

! 双核心互联 - 链路聚合配置
interface range GigabitEthernet1/0/1 - 2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode active
 no shutdown
exit

interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan all
 no shutdown
exit

! 管理VLAN配置
interface Vlan40
 ip address 10.10.40.33 255.255.255.0
 no shutdown
 standby 40 ip 10.10.40.254
 standby 40 priority 120
 standby 40 preempt
exit

! AP管理VLAN配置（办公外网AP）
interface Vlan50
 ip address 10.10.50.1 255.255.255.0
 no shutdown
 standby 50 ip 10.10.50.254
 standby 50 priority 120
 standby 50 preempt
exit

! ========== VLAN 210（行政办公电脑）- VRRP 配置 ==========
interface Vlan210
ip address 10.10.210.1 255.255.255.0
no shutdown
standby 210 ip 10.10.210.254
standby 210 priority 120 
standby 210 preempt
exit

! ========== VLAN 220（外网路由器）- VRRP 配置 ==========
interface Vlan220
 ip address 10.10.220.1 255.255.255.240
 no shutdown
 standby 220 ip 10.10.220.14
 standby 220 priority 120
 standby 220 preempt
exit

! ========== VLAN 230（财务行政服务器）- VRRP 配置 ==========
interface Vlan230
 ip address 10.10.230.1 255.255.255.240
 no shutdown
 standby 230 ip 10.10.230.14
 standby 230 priority 120
 standby 230 preempt
exit

! ========== VLAN 240（客户病房无线网络）- VRRP 配置 ==========
interface Vlan240
 ip address 10.20.0.1 255.255.240.0
 no shutdown
 standby 240 ip 10.20.15.254
 standby 240 priority 120
 standby 240 preempt
exit

! 启用三层路由
ip routing

! DHCP配置 - VLAN210行政办公
service dhcp
ip dhcp pool VLAN210-Admin
network 10.10.210.0 255.255.255.0  
default-router 10.10.210.254
ip dhcp excluded-address 10.10.210.1
ip dhcp excluded-address 10.10.210.2
ip dhcp excluded-address 10.10.210.3
ip dhcp excluded-address 10.10.210.254

! DHCP配置 - VLAN240访客无线
ip dhcp pool VLAN240-Guest
network 10.20.0.0 255.255.240.0
default-router 10.20.15.254

ip dhcp excluded-address 10.20.0.1
ip dhcp excluded-address 10.20.0.2
ip dhcp excluded-address 10.20.15.254

! 生成树配置
spanning-tree mode rapid-pvst        
spanning-tree vlan 210,220,230,240,50 root primary 

! 保存配置
exit
write memory
```

#### 3.2.1.2 Core-Ext-2

```
enable
configure terminal

hostname Core-Ext-2
no ip domain-lookup

! 创建办公外网相关VLAN
vlan 210
 name Admin_Office
exit
vlan 220
 name Ext_Router
exit
vlan 230
 name Finance_Server
exit
vlan 240
 name Guest_Wireless
exit
vlan 40
 name Management
exit
vlan 50
 name AP_Management_Ext
exit

! 双核心互联 - 链路聚合配置
interface range GigabitEthernet1/0/1 - 2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode active
 no shutdown
exit

interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shutdown
exit

! 管理VLAN配置
interface Vlan40
 ip address 10.10.40.34 255.255.255.0
 no shutdown
 standby 40 ip 10.10.40.254
 standby 40 priority 100
 standby 40 preempt
exit

! AP管理VLAN配置（办公外网AP）
interface Vlan50
 ip address 10.10.50.2 255.255.255.0
 no shutdown
 standby 50 ip 10.10.50.254
 standby 50 priority 100
 standby 50 preempt
exit

! ========== VLAN 210（行政办公电脑）- VRRP 配置（备核心） ==========
interface Vlan210
 ip address 10.10.210.2 255.255.255.0
 no shutdown
 standby 210 ip 10.10.210.254
 standby 210 priority 100
 standby 210 preempt
exit

! ========== VLAN 220（外网路由器）- VRRP 配置（备核心） ==========
interface Vlan220
 ip address 10.10.220.2 255.255.255.240
 no shutdown
 standby 220 ip 10.10.220.14
 standby 220 priority 100
 standby 220 preempt
exit

! ========== VLAN 230（财务行政服务器）- VRRP 配置（备核心） ==========
interface Vlan230
 ip address 10.10.230.2 255.255.255.240
 no shutdown
 standby 230 ip 10.10.230.14
 standby 230 priority 100
 standby 230 preempt
exit

! ========== VLAN 240（客户病房无线网络）- VRRP 配置（备核心） ==========
interface Vlan240
 ip address 10.20.0.2 255.255.240.0
 no shutdown
 standby 240 ip 10.20.15.254
 standby 240 priority 100
 standby 240 preempt
exit

! 启用三层路由
ip routing

! DHCP备用地址池配置
ip dhcp pool VLAN210-Admin-Backup
 network 10.10.210.0 255.255.255.0
 default-router 10.10.210.254

ip dhcp excluded-address 10.10.210.1
ip dhcp excluded-address 10.10.210.2
ip dhcp excluded-address 10.10.210.3
ip dhcp excluded-address 10.10.210.254

ip dhcp pool VLAN240-Guest-Backup
 network 10.20.0.0 255.255.240.0
 default-router 10.20.15.254

ip dhcp excluded-address 10.20.0.1
ip dhcp excluded-address 10.20.0.2
ip dhcp excluded-address 10.20.15.254

! DHCP中继配置
interface Vlan210
 ip helper-address 10.10.210.1
exit
interface Vlan240
 ip helper-address 10.20.0.1
exit

! 生成树配置
spanning-tree mode rapid-pvst
spanning-tree vlan 210,220,230,240,50 root secondary  

! 保存配置
exit
write memory
```



#### 3.2.1.3 配置验证

使用g 1/0/24测试

```
en
conf t
int g 1/0/4
sw mode acc
sw acc vlan 210
no sh
exit

```

### 3.2.2 行政办公汇聚层配置

#### 3.2.2.1 Core-Ext-1 补充配置

```
enable
configure terminal

! ========== 1. 对接 Dist-Ext-Admin-1 的接口（G1/0/3） ==========
interface GigabitEthernet1/0/3
 switchport trunk encapsulation dot1q 
 switchport mode trunk
 switchport trunk allowed vlan 210,40
 no shutdown
exit

! ========== 2. 对接 Dist-Ext-Admin-2 的接口（G1/0/4） ==========
interface GigabitEthernet1/0/4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 210,40
 no shutdown
exit

! ========== 3. 配置管理VLAN 40的虚拟网关（VRRP） ==========
! 办公外网核心层管理VLAN虚拟网关，汇聚层默认网关指向此地址
interface Vlan40
 standby 40 ip 10.10.40.254    
 ! 管理VLAN虚拟网关统一为10.10.40.254
 standby 40 priority 120       
 ! Core-Ext-1作为主网关
 standby 40 preempt            
 ! 抢占模式，确保主网关故障恢复后切回
exit

! ========== 4. 启用生成树（防环路，办公外网必备） ==========
spanning-tree mode rapid-pvst        
! 快速生成树，秒级故障切换
spanning-tree vlan 210,40 root primary 
! Core-Ext-1作为VLAN210（行政办公）/40的主根桥

! 保存配置
exit
write memory
```

#### 3.2.2.2 Core-Ext-2 补充配置

```
enable
configure terminal

! ========== 1. 对接 Dist-Ext-Admin-1 的接口（G1/0/3） ==========
interface GigabitEthernet1/0/3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 210,40
 no shutdown
exit

! ========== 2. 对接 Dist-Ext-Admin-2 的接口（G1/0/4） ==========
interface GigabitEthernet1/0/4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 210,40
 no shutdown
exit

! ========== 3. 配置管理VLAN 40的虚拟网关（VRRP） ==========
interface Vlan40
 standby 40 ip 10.10.40.254    
 ! 和Core-Ext-1一致的虚拟网关
 standby 40 priority 100       
 ! Core-Ext-2作为备网关
 standby 40 preempt            
 ! 抢占模式
exit

! ========== 4. 启用生成树 ==========
spanning-tree mode rapid-pvst
spanning-tree vlan 210,40 root secondary  
! Core-Ext-2作为VLAN210/40的备用根桥

! 保存配置
exit
write memory
```

#### 3.2.2.3 Dist-Ext-Admin-1

```
enable
configure terminal

hostname Dist-Ext-Admin-1
no ip domain-lookup

! 创建业务VLAN和管理VLAN
vlan 210
 name Admin_Workstation
exit
vlan 40
 name Management
exit

! ========== 上联 Core-Ext-1 的接口（Fa0/1） ==========
interface FastEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 210,40
 no shutdown
exit

! ========== 上联 Core-Ext-2 的接口（Fa0/2） ==========
interface FastEthernet0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 210,40
 no shutdown
exit

! ========== 管理VLAN配置（运维用） ==========
interface Vlan40
 ip address 10.10.40.35 255.255.255.0
 no shutdown
exit

! 管理网关指向核心层虚拟网关
ip default-gateway 10.10.40.254

! ========== 下联行政接入交换机接口（Fa0/3 - 24） ==========
interface range FastEthernet0/3 - 24
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 210,40
 no shutdown
exit

! ========== 生成树配置 ==========
spanning-tree mode rapid-pvst

! 保存配置
exit
write memory
```

#### 3.2.2.4 Dist-Ext-Admin-2

```
enable
configure terminal

hostname Dist-Ext-Admin-2
no ip domain-lookup

! 创建业务VLAN和管理VLAN
vlan 210
 name Admin_Workstation
exit
vlan 40
 name Management
exit

! ========== 上联 Core-Ext-1 的接口（Fa0/1） ==========
interface FastEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 210,40
 no shutdown
exit

! ========== 上联 Core-Ext-2 的接口（Fa0/2） ==========
interface FastEthernet0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 210,40
 no shutdown
exit

! ========== 管理VLAN配置（运维用） ==========
interface Vlan40
 ip address 10.10.40.36 255.255.255.0
 no shutdown
exit

! 管理网关指向核心层虚拟网关
ip default-gateway 10.10.40.254

! ========== 下联行政接入交换机接口（Fa0/3 - 24） ==========
interface range FastEthernet0/3 - 24
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 210,40
 no shutdown
exit

! ========== 生成树配置 ==========
spanning-tree mode rapid-pvst

! 保存配置
exit
write memory
```

#### 3.2.2.5 配置验证

断开Dist-Ext-Admin与Core-Ext-1之间的连接后，仍然能ping通

```
Dist-Ext-Admin-1#ping 10.10.40.36

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.40.36, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 0/0/0 ms

Dist-Ext-Admin-1#ping 10.10.40.33

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.40.33, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 0/0/1 ms

Dist-Ext-Admin-1#ping 10.10.40.34

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.40.34, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 0/0/1 ms

Dist-Ext-Admin-1#ping 10.10.40.254

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.40.254, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 0/0/0 ms

Dist-Ext-Admin-1#
```

### 3.2.3 行政办公接入层

#### 3.2.3.1 Acc-Ext-Office-1

```
enable
configure terminal

! 基础配置
hostname Acc-Ext-Office-1
no ip domain-lookup

! 创建业务VLAN和管理VLAN
vlan 210
 name Admin_Workstation
exit
vlan 40
 name Management
exit

! ========== 上联 Dist-Ext-Admin-1（Fa0/3） ==========
interface FastEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 210,40  
 no shutdown
exit

! ========== 上联 Dist-Ext-Admin-2（Fa0/3） ==========
interface FastEthernet0/2
 switchport mode trunk
 switchport trunk allowed vlan 210,40
 no shutdown
exit

! ========== 管理VLAN配置（运维用） ==========
interface Vlan40
 ip address 10.10.40.37 255.255.255.0  
 ! 接入层管理地址，按清单分配为10.10.40.37
 no shutdown
exit

! 管理网关指向核心层虚拟网关
ip default-gateway 10.10.40.254

! ========== 下联终端接口配置 ==========
! Fa0/3 专用于管理电脑PC5（静态绑定，区分普通办公终端）
interface FastEthernet0/3
 switchport mode access 
 switchport access vlan 210
 no shutdown
 ! 可选：配置端口安全，仅允许PC5的MAC地址接入（增强安全性）
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation restrict
 switchport port-security mac-address sticky  
 ! 自动绑定接入设备MAC（也可手动指定PC5的MAC）
exit

! Fa0/4 - 24 为普通行政办公终端接口（动态DHCP分配）
interface range FastEthernet0/4 - 24
 switchport mode access 
 switchport access vlan 210
 no shutdown
 ! 基础端口安全（可选）
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation restrict
exit

! ========== 生成树配置（防环路，冗余必备） ==========
spanning-tree mode rapid-pvst
! 优化接入层生成树（可选，加速终端接入）
spanning-tree portfast default

! 保存配置
exit
write memory
```

#### 3.2.3.2 配置验证

管理员ip 10.10.210.3

![image-20260312002158668](./images/image-20260312002158668.png)

dhcp

![image-20260312002206027](./images/image-20260312002206027.png)