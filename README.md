****

****[https://www.cnblogs.com/keerya/p/7987886.html
http://www.ansible.com.cn/
****

| 192.168.150.100 | ansible-server | zabbix |                  |
| --------------- | -------------- | ------ | ---------------- |
| 192.168.150.101 | ansible-web    |        | ：3000**grafana** |
| 192.168.150.102 | ansible-db     |        |                  |
|                 |                |        |                  |

# <span style>一、安装与配置</span>
## 1.安装

```bash
	#yum安装
	yum install epel-release -y
	yum install ansible –y
	
	#查看ansible版本
	ansible --version

```

## 2.管理节点与被管理节点建⽴SSH信任关系
	2.1.生成私钥
		[root@server ~]# ssh-keygen 
	  2.2.向主机分发私钥
		[root@server ~]# ssh-copy-id root@192.168.37.122
		[root@server ~]# ssh-copy-id root@192.168.37.133

## 3. ansible配置文件
修改配置文件，创建主机清单文件 ：写在[]里的是组名，[ ]下面的是组内的主机名
	[root@server ~]# vim /etc/ansible/hosts
		[web]
		192.168.37.122
		192.168.37.133

# 二、核心组件

主机清单（Inventory）
模块（Modules）
任务（Tasks）和剧本（Playbooks）
角色（Roles）

# <span style> 三、ansible-doc 命令行模块</span>

```bash
ansible <ip组> -m <模块> -a <argument参数> [options]


ansible dev,test,prod -a 'getenforce' 
```

命令的工作目录默认是远程用户的主目录（~），command 模块不会保留shell的环境状态（包括当前工作目录）

 ansible-doc -l  #列出当前系统中ansible的所有模块
ansible-doc  -s  模块名   #查看模块的参数
ansible-doc  模块名   #查看模块的详细信息包括用法

### 1.ping模块

`ansible all -m ping`

### 2.command模块

```shell
chdir=/etc/sysconfig  ---执行命令前先切换到指定目录
creates=/doc/1.txt：判断指定文件是否存在，如果存在，不执行后面的操作
removes：判断指定文件是否存在，如果存在，执行后面的操作

```

- 注意，该命令不支持==`| 管道命令`==。

`ansible all -m command -a ‘removes=/etc/sysconfig/network-scripts/ifcfg-ens33 ip addr ’`

### 3.shell模块

ansible all -m shell -a 'echo "niki_ansible" |cat >>1.txt`

ansible webservers -m shell -a 'ifconfig | grep ens33’

ansible webservers -m shell -a 'ifconfig > /opt/kx.txt’

### 4.file模块

```shell
state　　#状态，有以下选项：

directory：如果目录不存在，就创建目录
file：文件不存在，不会被创建
touch：文件不存在，创建新文件，如果文件或目录已存在，则更新其最后修改时间
absent：删除目录、文件或者取消链接文件
link：创建软链接，hard：创建硬链接
```

`ansible web -m file -a 'path=/data/app state=directory’`   创建目录

`ansible web -m file -a 'path=/data/a state=absent’`      删除文件

`ansible dbservers -m file -a ' group=root mode=644 path=/opt/host_ansible'` #修改文件的属主属组权限等

### 5.copy模块-将文件复制到远程主机

```bash
src　　　　#被复制到远程主机的本地文件。可以是绝对路径，也可以是相对路径。如果路径是一个目录，则会递归复制，用法类似于"rsync"
content　　　#用于替换"src"，可以直接指定文件的值
dest　　　　#必选项，将源文件复制到的远程主机的绝对路径
backup　　　#当文件内容发生改变后，在覆盖之前把源文件备份，备份文件包含时间信息
directory_mode　　　　#递归设定目录的权限，默认为系统默认权限
```

 `ansible web -m copy -a 'src=~/hello dest=/data/hello'` 

`ansible web -m copy -a 'content="I am keer\n" dest=/data/name mode=666’`

### 6.**fetch 模块-从远程主机获取文件到本地**

```bash
dest：用来存放文件的目录
src：在远程拉取的文件，**并且必须是一个file**，不能是目录
```

`ansible web -m fetch -a 'src=/data/hello dest=/data'`  

### 7.**yum/apt 模块**

```yaml
name：要管理的包名
state：#present--->安装 latest--->安装最新的, absent---> 卸载软件。
```
****
### 8. service 模块

　　该模块用于服务程序的管理。  
　　其主要选项如下：

> `arguments` #命令行提供额外的参数  
> `enabled` #设置开机启动。  
> `name=` #服务名称  
> `runlevel` #开机启动的级别，一般不用指定。  
> `sleep` #在重启服务的过程中，是否等待。如在服务关闭以后等待2秒再启动。(定义在剧本中。)  
> `state` #有四种状态，分别为：`started`--->启动服务， `stopped`--->停止服务， `restarted`--->重启服务， `reloaded`--->重载配置

　　下面是一些例子：  
**① 开启服务并设置自启动**

```dart
[root@server ~]# ansible web -m service -a 'name=nginx state=started enabled=true' 
192.168.37.122 | SUCCESS => {
    "changed": true, 
    "enabled": true, 
    "name": "nginx", 
    "state": "started", 
    ……
}
192.168.37.133 | SUCCESS => {
    "changed": true, 
    "enabled": true, 
    "name": "nginx", 
    "state": "started", 
    ……
}
```

## firewalld 模块

一个动态防火墙管理工具
**1. **state**：设置规则的状态。
    - `enabled`：启用规则。
    - `disabled`：禁用规则。
2. **permanent**：设置规则是否永久生效。
    - `yes`：规则永久生效，重启后依然有效。
    - `no`：规则临时生效，重启后失效。
3. **immediate**：是否立即应用规则变更。
    - `yes`：立即应用规则变更。
    - `no`：规则变更将在下一次 `firewalld` 重启时生效。
4. **service**：指定要允许或拒绝的服务名称，例如 `http`、`https`、`ssh` 等。
5. **port**：指定要允许或拒绝的端口，可以是单个端口（如 `8080/tcp`）或端口范围（如 `1024-2048/tcp`）。**


    ```yaml
    
    #启动 firewalld 服务**：
    - name: Start firewalld service
      ansible.builtin.firewalld:
        state: started
        enabled： yes
        
		#添加允许规则：
    - name: Open port 80 in firewalld
      ansible.builtin.firewalld:
        service: http
        permanent: yes
        state: enabled
        immediate: yes
    ```


## Setup模块
使用**setup模块**获取被管理主机的所有facts信息，可以使用filter来查看指定的信息。setup模块获取的整个facts信息被包装在一个JSON格式的数据结构中，ansible_facts是最外层的值。我们可以通过以下Ansible Ad-Hoc命令查看facts信息：
```
ansible all -m setup | grep hostname

 ansible all -m setup -a 'filter="ansible_nodename"'   
```

# 四、playbook-剧本

### playbook格式

**playbook 是 ansible 用于配置，部署，和管理被控节点的剧本。**`类似于脚本`

1. `-`和`:`后必须<font color="#ff0000">空一格</font>
2. 大小写敏感,使用缩进表示层级关系
3.  缩进时不允许使用tab键、只允许使用空格
4. 缩进的空格数目不重要，只要相同层级的元素左侧对齐即可
5. **任务列表**：每个任务前使用`-` 表示一个新的任务。

```yaml
---   
- name: Example playbook  
  hosts: web：test  # 指定此 Play 将在哪些主机上运行
  remote_user: root
  
  tasks:  
    - name: connect test  
      ping:    
  
    - name: new file  
      file: 
        group: root
        mode: 744
        path: /niki1/file1
        state: touch 
        
        
    #以上为普通形式，行数较多，但更容易操作。任务的关键字垂直堆叠，更容易区分。
    #以下可以运行但不推荐
    tasks:
      - name: shorthand form
        service: name=httpd enabled=true state=started
        
```



### 运行剧本

空运行 `ansible-playbook -C pb.yml`

直接运行 `ansible-playbook pb.yml`

语法验证 **`ansible-playbook --syntax-check pb.yml`**

指定从某个task开始运行`ansible-playbook pb.yml --start-at-task='xxx'`  


<font color="#00b050">| 绿色代表执行成功，系统保持原样</font>

<font color="#ffff00">| 黄色代表系统代表系统状态发生改变</font>

<font color="#ff0000">| 红色代表执行失败，显示错误输出</font>


#### ansible-navigator容器中运行剧本
ansible-navigator 利用容器化的执行环境，用于与 Ansible Playbooks、Roles、Collections 等交互。
ansible-navigator run playbook.yml <span style="background:rgba(173, 239, 239, 0.55)">-m stdout</span>
-`-m stdout`：这个选项指定输出模式为 `stdout`，即将执行结果输出到标准输出（终端），而不是使用交互式界面。
### **Handler+notify-触发器**

在特定条件下触发；接收到其它任务的通知`notify`时被触发

Handlers 最佳的应用场景是用来重启服务,或者触发系统重启操作.除此以外很少用到了.

### Tag-标签

为tasks或play添加标记，选择性地执行playbook的特定部分。

### 错误处理

如果命令或脚本的退出码不为零，可以使用如下方式替代

```yaml

tasks:
  -name: run this command and ignore the result
   shell: /usr/bin/somecommand || /bin/true
转错为正 如果命令失败则执行 true

或者使用ignore_errors来忽略错误信息
tasks:
		-name: run this command and ignore the result
		 shell: /usr/bin/somecommand
		 ignore_errors: True 忽略错误
```

### **Variable-变量**

变量名：仅能由字母、数字和下划线组成，且只能以字母开头
变量定义：直接定义  key=value 
根据变量的作⽤范围⼤体的将变量分为:

	全局变量
	剧本变量
	资产变量
	
	内置变量:内置变量⼏乎都是以 ansible_ 为前缀。
	系统自带事实变量facts：ansible setup模块   
  ansible all -m setup -a 'filter="ansible_nodename"'     #查询主机名

```yaml
1.系统自带事实变量facts：ansible setup模块   
  ansible all -m setup -a 'filter="ansible_nodename"'     #查询主机名
  
2./etc/ansible/hosts(主机清单)中定义变量

3.在playbook中定义
       vars:
        - var1: value1
        - var2: value2

4.在独立的变量YAML文件中定义
			vim vars.yml
			pack: vsftpd
			service: vsftpd
			
			引用变量文件
			vars_files:
			  - vars.yml
			  
```

变量调用：

```yaml
 通过{{ variable_name }} 调用变量
 通过-e指定  
 ansible-playbook test.yml -e "hosts=123"
```

### 示例

```yaml
#调用变量文件   
---
- hosts: web
  name: vars_log file
  vars_files:
     - /niki2/vars.yml

  tasks:
    - name: new file
      file:
         path: /niki1/{{ name1 }}_log.txt
         mode: "{{ mode1 }}"
         state: touch

```

### **Template-模板**

根据模板文件**动态**生成对应的配置文件，命名必须以 **.j2** 结尾（`Jinja2`：Jinja2是python的一种模板语言）

```yaml
---
- hosts: all
  remote_user: root

  tasks:
    - name: install nginx  #安装nginx，若为centos7需要启用 EPEL 仓库
      yum:
        name: nginx
        state: present

    - name: copy template
      template:
        src: /etc/ansible/templates/nginx.conf.j2 
        #nginx.conf.j2模板设置worker_processes 的值为   {{ ansible_processor_vcpus*2}};
        dest: /etc/nginx/nginx.conf
      notify: restart service
      
     
    - name: service start
      service:
        name: nginx
        state: started
        enabled: yes #开机自启

  handlers:
    - name: restart service
      service:
        name: nginx
        state: restarted
```

### 条件测试when与循环迭代  with_items

- when语句：在task中使用
- 循环：迭代，需要重复执行的任务；
    
    对迭代项的引用，固定变量名为"item"，而后，要在task中使用with_items给定要迭代的元素列表；
    

```yaml
---
- hosts: all
  remote_user: root

  tasks:
    - name: add groups
      group:
        name: "{{ item }}"
      with_items:
        - group1
        - group2
        - group3

    - name: add users
      user:
        name: "{{ item.name }}"
        group: "{{ item.group }}"
      with_items:
        - {name: 'u1',group: 'group1'}
        - {name: 'u2',group: 'group2'}
        - {name: 'u3',group: 'group3'}

```

# 五、roles-角色

用于**层次性，结构化**地组织playbook
roles通过分别将变量(vars)、文件(file)、任务(tasks)、模块(modules)及处理器(handlers)放置于单独的目录中，并可以便捷地include它们的一种机制。角色一般用于基于主机构建服务的场景中，但也可以是用于构建守护进程等场景中。 
### 创建role
```shell
ansible-galaxy init my_role


```

### 安装roels
```shell
#直接从galaxy安装
ansible-galaxy install <role_name>

#从 requirements.yml 文件安装多个角色
vim requirements.yml

- src: geerlingguy.apache
- src: geerlingguy.mysql
- 
#安装
ansible-galaxy install -r requirements.yml
---
 - name: webservers role
   hosts: webservers
   roles:
	 - apache

```

# 六、collection-集合
Collection 是一个更高级的封装，包含多个 Roles、模块、插件等，允许用户更全面地共享和分发自动化内容。它可以包含多个功能模块，适用于更复杂的项目。

## 如何使用 Ansible Collections
### 1、安装 Collection
```shell
#可以从 Ansible Galaxy 或其他来源安装 Collection：
ansible-galaxy collection install <namespace>.<collection_name>

#也可以从本地目录或通过 requirements.yml 文件安装collection：


```


### 2、Playbook 中使用 Collection
在你的 playbook.yml 文件中，你可以指定要使用的 Collection：
```yaml
---
- name: Example Playbook
  hosts: all
  collections:
    - my_namespace.my_collection
  tasks:
    - name: Example task
      my_namespace.my_collection.my_module:
        name: example
        state: present
```
### 3、管理 Collection

```shell
#列出已安装的 Collection：
ansible-galaxy collection list

#卸载 Collection：
ansible-galaxy collection uninstall my_namespace.my_collection

```

### 使用 requirements.yml 文件管理 Collection

> [!success] Title
> 创建一个 requirements.yml 文件来管理 Collection 
> 
> collections模块中name通常为集合名字。**如果直接提供 URL，格式为完整的链接。**


```shell
---
collections:
  - name: my_namespace.my_collection
    source: https://my.custom.repo/path/to/collection/
    version: 1.0.0


#然后使用以下命令安装所有 Collection：
ansible-galaxy collection install -r requirements.yml 

-r, --requirements
从指定的 requirements 文件中安装 Collections。

-p, --collections-path
指定安装 Collections 的目录。如果没有指定，默认安装到系统的 Ansible Collections 路径。
```


# 七、ansible galaxy

https://galaxy.ansible.com/ui/search/
Ansible Galaxy 是一个在线平台，提供一个广泛的 Ansible 资源库，包括 **角色（roles）**、**集合（collections）** 和其他自动化内容。用户可以通过 Galaxy 搜索、下载和分享这些资源。

```shell
#安装角色
ansible-galaxy role install <role_name>
#列出已安装的角色或集合：
ansible-galaxy role list
```

# 八、debug  msg
在Ansible中，`debug`模块是一个非常有用的工具，它允许你在执行任务时输出变量的值或执行过程中的信息。这对于调试和理解你的Ansible playbooks的执行流程非常有用。

### 使用方法

1. **基本用法**:
   ```yaml
   - name: Print a debug message
     debug:
       msg: "This is a debug message"
   ```

2. **输出变量**:
   ```yaml
   - name: Print the value of a variable
     debug:
       msg: "The value of my_var is {{ my_var }}"
   ```

3. **条件输出**:
   ```yaml
   - name: Print a message if a condition is met
     debug:
       msg: "This will only print if the condition is true"
     when: my_condition is defined and my_condition
   ```

4. **输出变量内容**:
   ```yaml
   - name: Print the content of a variable
     debug:
       var: my_var
   ```
### 调试技巧

- **在每个任务后添加debug模块**: 这可以帮助你跟踪变量的值和任务的执行状态。
- **使用`when`条件**: 只在特定条件下输出信息，避免不必要的输出。
- **使用`verbosity`**: 通过增加verbosity级别，可以输出更多的调试信息。在命令行中使用`-v`或`-vv`参数。


# 九、ansible-vault
`ansible-vault` 是一个用于加密和解密敏感数据的工具。它允许你以加密的形式存储敏感信息，如密码、密钥或任何其他敏感数据，以确保这些信息在Playbooks和变量文件中不会被暴露
