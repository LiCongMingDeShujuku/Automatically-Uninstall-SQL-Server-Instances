![CLEVER DATA GIT REPO](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/0-clever-data-github.png "李聪明的数据库")

# 自动卸载SQL Server实例
#### Automatically Uninstall SQL Server Instances
**发布-日期: 2016年11月15日 (评论)**


## Contents

- [中文](#中文)
- [English](#English)
- [SQL Logic](#Logic)
- [Build Info](#Build-Info)
- [Author](#Author)
- [License](#License) 


## 中文
最近我遇到了一种情况，我需要在多实例数据库服务器上测试SQL Server实例创建的限制和配置模式。在这种情况下，我使用的是SQL Server 2014 标准版运行安装程序。安装后更改设置，并运行更多安装方案。

你可以想象安装超过15个实例可能有点单调乏味。即使你使用无人参与的安装配置文件，你仍然需要确保安装具有不同的实例名称，然后运行各种检查（DBA Wisdom）以确保一切正常。

为了不这么麻烦，我决定添加一些自动化让我的过程更加统一。

以下SQL逻辑将自动创建无人参与的“卸载”配置文件，然后完成卸载。

注意：使用无人参与配置文件卸载SQL实例的最佳方法之一是，无需每次卸载都重新启动服务器。因此，你可以一次又一次卸载，而无需重新启动计算机。最终你可以实现在适当的时间内卸载它。

注意：此逻辑将一次卸载1个实例，并始终以最近安装的实例为目标。在此过程中假定你都具有统一的命名惯例。在这个例子中，我们使用的是SQLSHARE01,02,03 ......它的目标是最新的SQLSHARE03实例。

操作步骤如下：
1.创建变量表并创建现有实例的列表。
2.向你显示实例列表。
3.使用最新的实例名称创建无人参与的UNINSTALL配置文件。
4.使用QUIET模式运行卸载过程。
5.为你提供最新的剩余实例的列表。
假设你有16个实例。如果要继续自动删除实例，只需多次运行以下逻辑，它将会一直多次执行删除实例。
希望这对你有所帮助。

---

## English
Recently I had a situation where I needed to test out the limits, and configuration patterns of SQL Server Instance Creations on a Multi-Instance Database Server. In this case I was using SQL Server 2014 Standard, and running the installation. Changing settings (post install), and running through more install scenarios.
As you can imagine; installing over 15 instances can be somewhat tedious; even if you are using unattended installation config files… You still need to ensure the install has a different instance name, and then run through a variety of checks (DBA Wisdom) just to make sure everything was ok.
Instead of doing this I decided to make my process more uniform, and nothing gets better than adding some automation.
The following SQL logic will Automatically create the unattended ‘uninstall’ configs file, then run through the uninstallation completely.
Note: One of the best things about uninstalling SQL Instances using the unattended config file is that each uninstall does not need to be followed up with rebooting the server. Therefore you can uninstall again, and again without having to restart the machine. Ultimately you’ll need to uninstall it, but with this process it can be managed at an appropriate time.
Note: This logic will uninstall 1 instance at a time, and it always targets the most recently installed instance. This process assumes you have a uniform naming convention. In this example we are using SQLSHARE01, 02, 03… It targets the most recent instance which is SQLSHARE03. 
The following actions are taken:
1. Creates a variable table and creates a list of the existing instances.
2. Presents you with the list of instances.
3. Creates the unattended UNINSTALL configuration file using the most recent instance name.
4. Runs the process of uninstalling using QUIET mode.
5. Provides you with a new updated list of the remaining instances.
Suppose you have 16 instances. If you want to keep automatically removing the instance; simply run the below logic again, and again… It will keep removing the instances for as many times you run it.
Hope this is useful.


---
## Logic
```SQL
use master;
set nocount on
 
declare @server_name                varchar(255) = (select @@servername)
declare @run_bcp                varchar(8000) 
declare @silent_uninstall_string    varchar(max)
 
-- create table to hold instance names  --创建表来保存实例名称
declare @sql_instances table
(
    [id]        int identity(0,1)
,   [rootkey]       nvarchar(255)
,   [sql_instances] nvarchar(255)
,   [value]     nvarchar(255)
)
  
-- get instances names using xp_regread   --使用xp_regread获取实例名称
insert into @sql_instances ([rootkey], [sql_instances], [value]) execute xp_regread
    @rootkey        = 'hkey_local_machine'
,   @key            = 'software\microsoft\microsoft sql server'
,   @value_name     = 'installedinstances'
 
-- get list of existing instances    --获取现有实例的列表
select 'existing_sql_instances' = [sql_instances]   from @sql_instances
 
-- set prefix name for multi-instance environment aka: Shared SQL Environment "SQLSHARE"
-- produce the next instance name.
declare @prefix     varchar(255) = 'SQLSHARE'
declare @latest_instance    varchar(255) = 
(
select
    case
    when max([sql_instances]) = 'MSSQLSERVER' then @@servername
            else max([sql_instances])
    end as [sql_instances]
from
    @sql_instances si
)
-- reference: https://msdn.microsoft.com/en-us/library/ms144259(v=sql.120).aspx
 --参考：https://msdn.microsoft.com/en-us/library/ms144259(v=sql.120).aspx
-- create uninstall config file   --创建卸载配置文件
set @silent_uninstall_string    =
'	[OPTIONS]
	ACTION          	="Uninstall"
	ENU         		="True"
	QUIET           	="True"
	QUIETSIMPLE     	="False"
	FEATURES        	=SQLENGINE,RS
	HELP            	="False"
	INDICATEPROGRESS    ="False"
	X86         		="False"
	INSTANCENAME        ="' + @latest_instance + '"
'
if object_id('tempdb..##silent_uninstall_string') is not null drop table ##silent_uninstall_string
create table ##silent_uninstall_string ([bcpcommand]    varchar(8000)) insert into  ##silent_uninstall_string select @silent_uninstall_string
 
select  @run_bcp = 
'bcp "select [bcpcommand] from ##silent_uninstall_string" ' + 'queryout "e:\sql_silent_uninstall_config_file\unattended_uninstall_sql_2014_standard.ini" -c -t -T -S' + @server_name exec master..xp_cmdshell @run_bcp
 
-- run uninstall of the latest sql instance installed exec master..xp_cmdshell 
--运行卸载安装的最新sql实例exec master..xp_cmdshell 
'e:\sqlt1svgen1_dependencies\sql_server_2014\standard_x64\setup.exe /configurationfile=e:\sql_silent_uninstall_config_file\unattended_uninstall_sql_2014_standard.ini'
 
-- create table to hold instance names  --创建表来保存实例名称
declare @sql_remaining_instances table
(
    [id]        	int identity(0,1)
,   [rootkey]       nvarchar(255)
,   [sql_instances] nvarchar(255)
,   [value]     	nvarchar(255)
)
  
-- get instances names using xp_regread  --使用xp_regread获取实例名称
insert into @sql_remaining_instances ([rootkey], [sql_instances], [value]) execute xp_regread
    @rootkey        = 'hkey_local_machine'
,   @key            = 'software\microsoft\microsoft sql server'
,   @value_name     = 'installedinstances'
 
-- get list of remaining instances  --获取剩余实例的列表
select 'remaining_sql_instances'    = [sql_instances]   from @sql_remaining_instances

```

[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

- **李聪明的数据库 Lee's Clever Data**
- **Mike的数据库宝典 Mikes Database Collection**
- **李聪明的数据库** "Lee Songming"

[![Gist](https://img.shields.io/badge/Gist-李聪明的数据库-<COLOR>.svg)](https://gist.github.com/congmingshuju)
[![Twitter](https://img.shields.io/badge/Twitter-mike的数据库宝典-<COLOR>.svg)](https://twitter.com/mikesdatawork?lang=en)
[![Wordpress](https://img.shields.io/badge/Wordpress-mike的数据库宝典-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

---
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Lee Songming](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/1-clever-data-github.png "李聪明的数据库")

