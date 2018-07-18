# Licenses Guide

## 概览

Slurm可以在作业调度时为作业分配软件许可，如果没有软件许可作业将保持等待直到许可证可用。Slurm中的软件许可本质上是共享资源，即未绑定到特定主机但与整个群集关联的已配置资源。

Slurm中的许可证可以通过两种方式配置：

- 本地许可证: 本地许可证是配置在slurm.conf中的集群许可证。
- 远程许可证: 远程许可证由数据库提供，并使用sacctmgr命令进行配置。 远程许可证本质上是动态的，因为在运行sacctmgr时，slurmdbd会更新许可证分配给的所有集群。

## 本地许可证

在slurm.conf中使用Licenses项定义本地许可证。

slurm.conf:

```txt
Licenses=fluent:30,ansys:100
```

可以使用scontrol命令查看已配置的许可证。

```bash
$ scontrol show lic
LicenseName=ansys
    Total=100 Used=0 Free=100 Remote=no
LicenseName=fluent
    Total=30 Used=0 Free=30 Remote=no
```

通过使用-L或--licenses提交选项来请求许可证。

```bash
$ sbatch -L ansys:2 script.sh
Submitted batch job 5212

$ scontrol show lic
LicenseName=ansys
    Total=100 Used=2 Free=98 Remote=no
LicenseName=fluent
    Total=30 Used=0 Free=30 Remote=no
```

## 远程许可证

### 使用案例

一个单位有两个许可证服务器，一个服务器提供FlexNet提供的100个Nastran许可证，另一个服务器由Reprise License Management提供的50个Matlab许可证。该单位有两个名为“fluid”和“pdf”的集群，专门用于使用这两种产品运行模拟作业。管理人员希望在群集之间平均分配Nastran许可证的数量，但是将70％的Matlab许可证分配给群集“pdf”，剩余的30％分配给群集“fluid”。

### 为上述案例配置SLURM

这里我们假设使用sacctmgr命令在slurmdbd中正确配置了两个集群。

```txt
$ sacctmgr show clusters format=cluster,controlhost
   Cluster     ControlHost
---------- ---------------
     fluid     143.11.1.3
       pdf     144.12.3.2
```

使用sacctmgr命令添加许可证，指定许可证的总数以及应分配给每个群集的百分比。这可以在一个步骤中完成，也可以通过多步骤过程完成。

单步完成：

```bash
$ sacctmgr add resource name=nastran cluster=fluid,pdf \
  count=100 percentallowed=50 server=flex_host
  servertype=flexlm type=license
 Adding Resource(s)
  nastran@flex_host
   Cluster - pdf        50%
   Cluster - fluid      50%
 Settings
  Name           = nastran
  Server         = flex_host
  Description    = nastran
  ServerType     = flexlm
  Count          = 100
  Type           = License
```

多步完成：

```bash
$ sacctmgr add resource name=matlab count=50 server=rlm_host \
  servertype=rlm type=license
 Adding Resource(s)
  matlab@rlm_host
 Settings
  Name           = matlab
  Server         = rlm_host
  Description    = matlab
  ServerType     = rlm
  Count          = 50
  Type           = License

$ sacctmgr add resource name=matlab server=rlm_host
  cluster=pdf percentallowed=70
Adding Resource(s)
  matlab@rlm_host
   Cluster - pdf        70%
 Settings
  Server         = rlm_host
  Type           = License

$ sacctmgr add resource name=matlab server=rlm_host
  cluster=fluid percentallowed=30
 Adding Resource(s)
  matlab@rlm_host
   Cluster - fluid     30%
 Settings
  Server         = rlm_host
  Type           = License

```

sacctmgr命令现在将显示许可证的总数:

```bash
$ sacctmgr show resource
      Name     Server     Type  Count % Allocated ServerType 
---------- ---------- -------- ------ ----------- ---------- 
   nastran  flex_host  License    100         100     flexlm 
    matlab   rlm_host  License     50         100        rlm 

$ sacctmgr show resource withclusters
      Name     Server     Type  Count % Allocated ServerType    Cluster  % Allowed 
---------- ---------- -------- ------ ----------- ---------- ---------- ---------- 
   nastran  flex_host  License    100         100     flexlm        pdf         50 
   nastran  flex_host  License    100         100     flexlm      fluid         50 
    matlab   rlm_host  License     50         100        rlm        pdf         70 
    matlab   rlm_host  License     50         100        rlm      fluid         30 
```

现在，使用scontrol命令在两个群集上都可以看到已配置的许可证。

```bash
$ scontrol show lic
LicenseName=matlab@rlm_host
    Total=35 Used=0 Free=35 Remote=yes
LicenseName=nastran@flex_host
    Total=50 Used=0 Free=50 Remote=yes

# On cluster "fluid":
$ scontrol show lic
LicenseName=matlab@rlm_host
    Total=15 Used=0 Free=15 Remote=yes
LicenseName=nastran@flex_host
    Total=50 Used=0 Free=50 Remote=yes
```

将作业提交到远程许可证时，必须指定许可证名称和服务器。

```bash
$ sbatch -L nastran@flex_host script.sh
Submitted batch job 5172
```

可以修改许可证百分比和计数，如下所示：

```bash
$ sacctmgr modify resource name=matlab server=rlm_host set \
  count=200
 Modified server resource ...
  matlab@rlm_host
  Cluster - pdf         - matlab@rlm_host
  Cluster - fluid       - matlab@rlm_host

$ sacctmgr modify resource name=matlab server=rlm_host \
  cluster=pdf set percentallowed=60
 Modified server resource ...
  Cluster - pdf         - matlab@rlm_host

$ sacctmgr show resource withclusters
      Name     Server     Type  Count % Allocated ServerType    Cluster  % Allowed 
---------- ---------- -------- ------ ----------- ---------- ---------- ---------- 
   nastran  flex_host  License    100         100     flexlm        pdf         50 
   nastran  flex_host  License    100         100     flexlm      fluid         50 
    matlab   rlm_host  License    200          90        rlm        pdf         60 
    matlab   rlm_host  License    200          90        rlm      fluid         30 
```

可以在群集上删除许可证，也可以将它们全部删除，如下所示：

```bash
$ sacctmgr delete resource where name=matlab server=rlm_host \
  cluster=fluid
 Deleting resource(s)...
  Cluster - fluid      - matlab@rlm_host

$ sacctmgr delete resource where name=nastran server=flex_host
 Deleting resource(s)...
  nastran@flex_host
  Cluster - pdf        - nastran@flex_host
  Cluster - fluid      - nastran@flex_host

$ sacctmgr show resource withclusters
      Name     Server     Type  Count % Allocated ServerType    Cluster  % Allowed 
---------- ---------- -------- ------ ----------- ---------- ---------- ---------- 
    matlab   rlm_host  License    200          60        rlm      pdf         60 
```

## 动态许可证

使用sacctmgr命令更新许可证计数器和百分比时，将在数据库中更新值，并将更新的值发送到slurmctld守护程序。可以编写一个脚本，用于检测由于添加新许可证或删除旧许可证而导致的全局许可证计数器更改并更新Slurm。如果许可证数量减少，不会影响正在运行的作业，只有新分发的作业会反映新的许可证计数。