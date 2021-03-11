# HP ILO信息收集



## 1.目录结构

![未命名截图1](C:\Users\f1237282\Desktop\未命名截图1.png)

`Batch_DLL` 文件目录实现批量修改账号，ntp设置等功能

`analyze_ilo` 文件目录实现批量获取主机信息

`database` 文件目录实现导入数据库

`ipam` 文件目录实现批量ping IP 及检测是否为ilo端口

`judge_account`文件目录用于检测ilo账号密码（gl这边账号密码不统一）



## 2.main文件 

```python
if __name__ == '__main__':

    num_threads = 200 #设定线程值
    threads = []
    sql = "select HARDWARE_ID,IP_ILO,CHECK_NEW,IP_ILO_STATUS,SITE from ILO_INFO" 
    #从数据库中取出server的相关信息
    # sql ='''
    #           select site_code_id,ip_ilo,count(ip_ilo) as num
    #           from itim_auto.ITIM_ASSETS@itim_auto_pitimdb where CATEGORY_ID='服務器'
    #           and ASSETS_STATUS_ID in ('正式','在用','備用','測試') and assets_model like 'HP%'
    #           group by site_code_id,ip_ilo having count(ip_ilo)=1
    #      '''
    g = db.connection_database(sql) #连接oracle
    start = time.clock()
    def base(g):
        while True:

            try:
                lock.acquire() # 为每个线程加上锁 以防数据‘张冠李戴’
                id, ip, new, status, site = g.next()
                lock.release()
                print(ip)
                if status == '1':
                    test(id, ip.strip(), new, site)
            except StopIteration:
                lock.release() #务必记得释放
                break


    for i in range(num_threads):
       thread = threading.Thread(target=base, args=(g,))
       thread.start()
       threads.append(thread)
    for thread in threads: thread.join() #等待最后一个线程跑完
    end = time.clock()
    print(end-start)
    
    
    
def test(id, ip, new, site):
     #设定登录账号
 
    if site == 'GL':
       try:
           ilo = hpilo.Ilo(ip, login='MonitorTools', password='ping.p.shen@foxconn.com')
           # admin
           ilo.get_product_name()
       except:
           try:
               ilo = hpilo.Ilo(ip, login='admin', password='iL0!@#123')
               ilo.get_product_name()
           except:
               print(ip, "connect failed")
               return

    elif site == 'ZZ':
        try:
            ilo = hpilo.Ilo(ip, login='admin', password='Idpbg123.')
            ilo.get_product_name()
        except:
            try:
                ilo = hpilo.Ilo(ip, login='MonitorTools', password='ping.p.shen@foxconn.com')
                # admin
                ilo.get_product_name()
            except:
                print(ip, "connect failed")
                return
    elif site == 'TY':
        try:
            ilo = hpilo.Ilo(ip, login='admin', password='dpbgty123.')
            ilo.get_product_name()
        except:
            try:
                ilo = hpilo.Ilo(ip, login='MonitorTools', password='ping.p.shen@foxconn.com')
                # admin
                ilo.get_product_name()
            except:
                print(ip, "connect failed")
                return



    server_name = ilo.get_server_name()  #获取主机名
    product_name = ilo.get_product_name()  # 获取产品型号
    fw_version_info = ilo.get_fw_version() #获取版本
    health_info = ilo.get_embedded_health() #获取大部分硬体信息
    serial_number = ilo.get_host_data()[1]['Serial Number']
    s = analyse_json.get_firmware_info(fw_version_info)# 获取FW版本信息
    ilo_model = s['management_processor'] 
    fw_version = s['firmware_version'] 
    fw_data = s['firmware_date']
    general_info = analyse_json.get_glance_info(health_info,fw_version)
    bios_hardware = general_info['bios_hardware'] #获得bios总信息
    fans = general_info['fans'] #获得fans的总信息
    temperature = general_info['temperature'] # 获得温度总信息
    power_supplies = general_info['power_supplies'] # 获得power_supplies总信息
    battery = general_info['battery'] # 获得电池总信息
    processor = general_info['processor'] #获得cpu的总信息
    memory = general_info['memory'] # 获得内存总信息
    network = general_info['network'] # 获得网络总信息
    storage = general_info['storage'] # 获得存储总信息

    
```



## 3.analyse_json.py



`/analyze_ilo/analyse_json.py `该文件为具体拆分分析以上收到的总信息

```python
# 一共17个分析函数，因为ilo3和ilo4、ilo5数据结构差别较大，所以拆出来单独写
def get_firmware_info(all_info):

def get_storage_info_ilo3(all_info):

def get_storage_info(all_info):

def get_memory_info(all_info):

def get_memory_info_ilo3(all_info):

def get_network_info(xmldata):

def get_processors_info_ilo3(all_info):

def get_processors_info(all_info):

def get_fan_info(ilo_model,all_info):

def get_power_info(all_info):

def get_power_summary_info(all_info):

def get_power_summary_ilo3_info(all_info):

def get_power_info_ilo3(all_info):

def get_temperature_info(all_info):

def get_temperature_info_ilo3(all_info):

def get_glance_info(all_info,fw_version):

def get_log_info(ilo):
```



## 4.scan_ilo.py

`/ipam/scan_ilo.py` 该文件用户查看ip和端口的健康状态 

1. 首先通过`def Ping_all():` 函数判断IP是否通
2. 如果通则用`def judge_ilo():` 判断是否是ilo口

**通过nmap扫描IP的22端口 如果是ilo3则会返回 "AllegroSoft RomSShell sshd"**

**如果返回'HP Integrated Lights-Out mpSSH'则是ilo4、ilo5**



## 5.insert_database.py

`/database/insert_database.py`

经过analyse_json.py文件解析了json数据后，调用该文件写入oracle



## 6.实现功能

`/Batch_DLL/change_account.py`

```python
def login_password(ip): 

def add_account():

使用以上两个函数实现增删改操作 在main函数中调用改函数可实现批量操作
```

  

`/Batch_DLL/change_NTP.py`

```python
ilo.mod_network_settings(sntp_server1='10.172.113.1',sntp_server2='10.173.173.163',timezone='Asia/Taipei')
使用该函数可以批量修改ntp设置
```



**ilo3以上的server除了账号密码不清楚外的都有收集到信息**

![未命名截图](C:\Users\f1237282\Desktop\未命名截图.png)

## 7.未实现功能

目前HBA卡 和 阵列卡电池信息无法获得

