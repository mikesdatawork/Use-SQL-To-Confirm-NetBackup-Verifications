> **Note** — The folder `linguist-samples/` contains tiny real files so GitHub can correctly display all languages used in this repo.  
> The actual content and examples remain in this README.

![Mikes Data Work Git](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_01.png "Mikes Data Work")
AGENT JOBS

# Use SQL To Confirm NetBackup Verifications
**POST DATE: JULY 23, 2018**





## Contents    
- [About Process](##About-Process)  
- [SQL Logic](#SQL-Logic)  
- [Build Info](#Build-Info)  
- [Author](#Author)  
- [License](#License)       

## About-Process

 
<p>You can get a list of all the files that Netbackup has successfully copied by using the BPLIST utility. Ordinarilly this is run from Powershell or Command prompt, but it can be run using SQL, and the results can be pulled into a table, and you can build a process around it if needed. Thats exactly what I did here. </p>   
 
   
## SQL-Logic
```SQL
1	exec master..xp_cmdshell 'C:\NetBackupListplist -R 99 -C MyServerName.MyDomain.com -s 06/23/2018 00:00:00 -e 07/24/2018 00:00:00 -I "F:\BACKUPS"'

```


In order for this backup process to work you'll need to copy the Netbackup BPLIST utility to a root folder. In this example I am using the root of C:. I created a folder called NetBackupList, and there is where I copied the BPLIST utility and all the corresponding DLL's.
You can see the BPLIST.exe utility (in red) along with all the DLL's (in blue). Again; you can use any folder you want, but it would be much easier if it's a root folder, and the folder name has no spaced. This probably shouldn't be in Drive C:, but in this example it works well.

In this example; the BPLIST utility is found here under:
C:\Program Files\VERITAS\NetBackupin

 
Here's all the files you'll need to copy in order for the BPLIST.exe utility to work.
bplist.exe
libnbbase.dll
libnbclient.dll
vrtsLogFormatMsg_3.dll
vrtslogread_3.dll
vxACE_6.dll
vxcPBX.dll
vxicudata_6.dll
vxicui18n_6.dll
vxicuuc_6.dll
vxlis_3.dll
vxul_3.dll

![See Standared BPLIST Utility Dlls]( https://mikesdatawork.files.wordpress.com/2018/07/image002.png "See Standard BPLIST Utility Dlls")
 
When writing this process the firs thing you need to do is parameterize the BPLIST statement by creating variables that you can plug into it. You can see how I'm doing that below. The next thing is create a series of lists. A list of all backups that have occurred through SQL internal tables which is captured from BackupMediaFamily (splitting the path name from the file name), and using the path to run the BPLIST utility against every known backup path, and then taking the results of that and comparing the path list that BPLIST shows you against the database backups you are holding locally. Next you concatinate a Delete statement to run against only the files that were confirmed (using the BPLIST output), and you have a cleanup script which removes only the files that have already been backed up by Netbackup.
     
## SQL-Logic
```SQL
use master;
set nocount on
 
declare     @fqdn_server_name   varchar(255)
declare     @tomorrow       varchar(255)
declare     @30_days        varchar(255)
declare     @file_removal       table ([del_statement] varchar(555))
set     @fqdn_server_name   = (select cast(serverproperty('machinename') as varchar) + '.' + lower(default_domain()) + '.com')
set     @tomorrow       = (select convert(varchar, getdate() + 1, 101)  + ' 00:00:00')
set     @30_days        = (select convert(varchar, getdate() - 30, 101) + ' 00:00:00')
 
-- create temp table to capture list of all confirmed netbackup file copies.
if object_id('tempdb..##netbackup_confirmed') is not null
    drop table  ##netbackup_confirmed
create table    ##netbackup_confirmed ([file_name] varchar(255))
 
-- find all backup path and backup files for the last 30 days.
declare     @duration   datetime
set     @duration   = (select getdate() - 30)
declare     @backup_history table ([location] varchar(255), [backup_file] varchar(255))
insert into @backup_history
select
    'location'          = reverse(right(reverse(upper(bmf.physical_device_name)), len(bmf.physical_device_name) - charindex('\',reverse(bmf.physical_device_name),1) + 1))
,   'backup_file'       = right(bmf.physical_device_name, charindex('\', reverse('\' + bmf.physical_device_name)) - 1)
from        msdb.dbo.backupset bs join msdb.dbo.backupmediafamily bmf on bs.media_set_id = bmf.media_set_id
where       bs.backup_finish_date > @duration
        and bmf.[device_type] not in ('7')
group by    bs.database_name,   bs.backup_finish_date, bmf.physical_device_name, bs.type, bmf.device_type
order by    bs.database_name,   bs.backup_finish_date desc
 
-- build Netbackup BPLIST logic for all known backup paths, and corresponding backup files.
declare @bplist_all     table ([bp_commands] varchar(max))
insert into @bplist_all
select distinct
'insert into ##netbackup_confirmed exec master..xp_cmdshell ''C:\NetBackupListplist -R 99 -C ' + @fqdn_server_name + ' -s ' + @30_days + ' -e ' + @tomorrow + ' -I "' + reverse(stuff(reverse([location]), 1, 1, ''))  + '"'''
from @backup_history
 
-- execute Netbackup BPLIST utility across all unique backup paths gathering all confirmed files copied to enterprise storage and place them into table ##netbackup_confirmed.
declare @populate_nbc       varchar(max)
set @populate_nbc       = ''
select  @populate_nbc       = @populate_nbc + 
[bp_commands]
from    @bplist_all
exec    (@populate_nbc)
 
-- get manual confirmation.  run query against @bplist_all and get the BPLIST command for any drive on the server.
-- run the following command with the target drive letter you're looking for.
-- select replace([bp_commands], 'insert into ##netbackup_confirmed ', '') from @bplist_all where [bp_commands] like '%y:\%'
 
-- created file removal process for only files that have successfully backed up to the enterprise.
insert into @file_removal
select 'exec master..xp_cmdshell ''DEL "' + [location] + [backup_file] + '"'';' from @backup_history where [location] + [backup_file] in (select [file_name] from ##netbackup_confirmed)
order by [backup_file] desc
 
-- remove backups that thave already been moved to enterprise storage.
declare @delete_files   varchar(max)
set @delete_files = ''
select  @delete_files = @delete_files + 
[del_statement] + char(10) 
from    @file_removal
exec    (@delete_files)

```

[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

[![Gist](https://img.shields.io/badge/Gist-MikesDataWork-<COLOR>.svg)](https://gist.github.com/mikesdatawork)
[![Twitter](https://img.shields.io/badge/Twitter-MikesDataWork-<COLOR>.svg)](https://twitter.com/mikesdatawork)
[![Wordpress](https://img.shields.io/badge/Wordpress-MikesDataWork-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)


## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Mikes Data Work](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_02.png "Mikes Data Work")
