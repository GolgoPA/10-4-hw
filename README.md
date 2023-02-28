# Домашнее задание к занятию 10.4 «Резервное копирование»

---

### Задание 1

В чём разница между:

- полным резервным копированием,
- дифференциальным резервным копированием,
- инкрементным резервным копированием.

*Приведите ответ в свободной форме.*

- полное резервное копирование: 
> Каждый раз полное копирование всего.
- дифференциальное резервное копирование:
>  Сначала делается полное копирование всего. При следующих запусках копируются только данные, изменённые с момента полного бэкапа.
- инкрементное резервное копирование:
>   Сначала делается полное копирование всего. При следующих запусках копируются только данные, изменённые с момента последнего проведённого резервного копирования.

---

### Задание 2

Установите программное обеспечении Bacula, настройте bacula-dir, bacula-sd,  bacula-fd. Протестируйте работу сервисов.

*Пришлите конфигурационные файлы для bacula-dir, bacula-sd,  bacula-fd.*
 bacula-dir.conf: 
```
# Default Bacula Director Configuration file
#
#  The only thing that MUST be changed is to add one or more
#   file or directory names in the Include directive of the
#   FileSet resource.
#
#  For Bacula release 9.6.7 (10 December 2020) -- debian bullseye/sid
#
#  You might also want to change the default email address
#   from root to your address.  See the "mail" and "operator"
#   directives in the Messages resource.
#
# Copyright (C) 2000-2020 Kern Sibbald
# License: BSD 2-Clause; see file LICENSE-FOSS
#

Director {                            # define myself
  Name = hw-10-4-n1-dir
  DIRport = 9101                # where we listen for UA connections
  QueryFile = "/etc/bacula/scripts/query.sql"
  WorkingDirectory = "/var/lib/bacula"
  PidDirectory = "/run/bacula"
  Maximum Concurrent Jobs = 20
  Password = "qwerty12345"         # Console password
  Messages = Daemon
  DirAddress = 127.0.0.1
}


...skipping 1 line
  Name = "DefaultJob"
  Type = Backup
  Level = Incremental
  Client = hw-10-4-n1-fd
  FileSet = "Full Set"
  Schedule = "WeeklyCycle"
  Storage = File1
  Messages = Standard
  Pool = File
  SpoolAttributes = yes
  Priority = 10
  Write Bootstrap = "/var/lib/bacula/%c.bsr"
}


#
# Define the main nightly save backup job
#   By default, this job will back up to disk in /nonexistant/path/to/file/archive/dir
Job {
  Name = "BackupClient1"
  JobDefs = "DefaultJob"
}

#Job {
#  Name = "BackupClient2"
#  Client = hw-10-4-n12-fd
#  JobDefs = "DefaultJob"
#}


...skipping 1 line
#  Name = "BackupClient1-to-Tape"
#  JobDefs = "DefaultJob"
#  Storage = LTO-4
#  Spool Data = yes    # Avoid shoe-shine
#  Pool = Default
#}

#}

# Backup the catalog database (after the nightly save)
Job {
  Name = "BackupCatalog"
  JobDefs = "DefaultJob"
  Level = Full
  FileSet="Catalog"
  Schedule = "WeeklyCycleAfterBackup"
  # This creates an ASCII copy of the catalog
  # Arguments to make_catalog_backup.pl are:
  #  make_catalog_backup.pl <catalog-name>
  RunBeforeJob = "/etc/bacula/scripts/make_catalog_backup.pl MyCatalog"
  # This deletes the copy of the catalog
  RunAfterJob  = "/etc/bacula/scripts/delete_catalog_backup"
  Write Bootstrap = "/var/lib/bacula/%n.bsr"
  Priority = 11                   # run after main backup
}

#
# Standard Restore template, to be changed by Console program
#  Only one such job is needed for all Jobs/Clients/Storage ...

...skipping 1 line
Job {
  Name = "RestoreFiles"
  Type = Restore
  Client=hw-10-4-n1-fd
  Storage = File1
# The FileSet and Pool directives are not used by Restore Jobs
# but must not be removed
  FileSet="Full Set"
  Pool = File
  Messages = Standard
  Where = /nonexistant/path/to/file/archive/dir/bacula-restores
}


# List of files to be backed up
FileSet {
  Name = "Full Set"
  Include {
    Options {
      signature = MD5
    }
#
#  Put your list of files here, preceded by 'File =', one per line
#    or include an external list with:
#
#    File = <file-name
#
#  Note: / backs up everything on the root partition.
#    if you have other partitions such as /usr or /home

...skipping 1 line
#
#  By default this is defined to point to the Bacula binary
#    directory to give a reasonable FileSet to backup to
#    disk storage during initial testing.
#
    File = /usr/sbin
  }

#
# If you backup the root directory, the following two excluded
#   files can be useful
#
  Exclude {
    File = /var/lib/bacula
    File = /nonexistant/path/to/file/archive/dir
    File = /proc
    File = /tmp
    File = /sys
    File = /.journal
    File = /.fsck
  }
}

#
# When to do the backups, full backup on first sunday of the month,
#  differential (i.e. incremental since full) every other sunday,
#  and incremental backups other days
Schedule {
  Name = "WeeklyCycle"

...skipping 1 line
  Run = Differential 2nd-5th sun at 23:05
  Run = Incremental mon-sat at 23:05
}

# This schedule does the catalog. It starts after the WeeklyCycle
Schedule {
  Name = "WeeklyCycleAfterBackup"
  Run = Full sun-sat at 23:10
}

# This is the backup of the catalog
FileSet {
  Name = "Catalog"
  Include {
    Options {
      signature = MD5
    }
    File = "/var/lib/bacula/bacula.sql"
  }
}

# Client (File Services) to backup
Client {
  Name = hw-10-4-n1-fd
  Address = localhost
  FDPort = 9102
  Catalog = MyCatalog
  Password = "qwerty12345"          # password for FileDaemon
  File Retention = 60 days            # 60 days

...skipping 1 line
  AutoPrune = yes                     # Prune expired Jobs/Files
}

#
# Second Client (File Services) to backup
#  You should change Name, Address, and Password before using
#
#Client {
#  Name = hw-10-4-n12-fd
#  Address = localhost2
#  FDPort = 9102
#  Catalog = MyCatalog
#  Password = "FJfPGqhDmqesgOtnWnbt1eSY2H9L01yOT2"        # password for FileDaemon 2
#  File Retention = 60 days           # 60 days
#  Job Retention = 6 months           # six months
#  AutoPrune = yes                    # Prune expired Jobs/Files
#}


# Definition of file Virtual Autochanger device
Autochanger {
  Name = File1
# Do not use "localhost" here
  Address = localhost                # N.B. Use a fully qualified name here
  SDPort = 9103
  Password = "qwerty12345"
  Device = FileChgr1
  Media Type = File1
  Maximum Concurrent Jobs = 10        # run up to 10 jobs a the same time

...skipping 1 line
}

# Definition of a second file Virtual Autochanger device
#   Possibly pointing to a different disk drive
Autochanger {
  Name = File2
# Do not use "localhost" here
  Address = localhost                # N.B. Use a fully qualified name here
  SDPort = 9103
  Password = "qwerty12345"
  Device = FileChgr2
  Media Type = File2
  Autochanger = File2                 # point to ourself
  Maximum Concurrent Jobs = 10        # run up to 10 jobs a the same time
}

# Definition of LTO-4 tape Autochanger device
#Autochanger {
#  Name = LTO-4
#  Do not use "localhost" here
#  Address = localhost               # N.B. Use a fully qualified name here
#  SDPort = 9103
#  Password = "GPJ0GTv3xFqSqtSHCu27tESrkEuMjhOcb"         # password for Storage daemon
#  Device = LTO-4                     # must be same as Device in Storage daemon
#  Media Type = LTO-4                 # must be same as MediaType in Storage daemon
#  Autochanger = LTO-4                # enable for autochanger device
#  Maximum Concurrent Jobs = 10
#}


...skipping 1 line
Catalog {
  Name = MyCatalog
  dbname = "bacula"; DB Address = "localhost"; dbuser = "bacula"; dbpassword = "qwerty12345"
}

# Reasonable message delivery -- send most everything to email address
#  and to the console
Messages {
  Name = Standard
#
# NOTE! If you send to two email or more email addresses, you will need
#  to replace the %r in the from field (-f part) with a single valid
#  email address in both the mailcommand and the operatorcommand.
#  What this does is, it sets the email address that emails would display
#  in the FROM field, which is by default the same email as they're being
#  sent to.  However, if you send email to more than one address, then
#  you'll have to set the FROM address manually, to a single address.
#  for example, a 'no-reply@mydomain.com', is better since that tends to
#  tell (most) people that its coming from an automated source.

#
  mailcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula: %t %e of %c %l\" %r"
  operatorcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula: Intervention needed for %j\" %r"
  mail = root = all, !skipped
  operator = root = mount
  console = all, !skipped, !saved
#
# WARNING! the following will create a file that you must cycle from
#          time to time as it will grow indefinitely. However, it will

...skipping 1 line
#
  append = "/var/log/bacula/bacula.log" = all, !skipped
  catalog = all
}


#
# Message delivery for daemon messages (no job).
Messages {
  Name = Daemon
  mailcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula daemon message\" %r"
  mail = root = all, !skipped
  console = all, !skipped, !saved
  append = "/var/log/bacula/bacula.log" = all, !skipped
}

# Default pool definition
Pool {
  Name = Default
  Pool Type = Backup
  Recycle = yes                       # Bacula can automatically recycle Volumes
  AutoPrune = yes                     # Prune expired volumes
  Volume Retention = 365 days         # one year
  Maximum Volume Bytes = 50G          # Limit Volume size to something reasonable
  Maximum Volumes = 100               # Limit number of Volumes in Pool
}

# File Pool definition
Pool {

...skipping 1 line
  Pool Type = Backup
  Recycle = yes                       # Bacula can automatically recycle Volumes
  AutoPrune = yes                     # Prune expired volumes
  Volume Retention = 365 days         # one year
  Maximum Volume Bytes = 50G          # Limit Volume size to something reasonable
  Maximum Volumes = 100               # Limit number of Volumes in Pool
  Label Format = "Vol-"               # Auto label
}


# Scratch pool definition
Pool {
  Name = Scratch
  Pool Type = Backup
}

#
# Restricted console used by tray-monitor to get the status of the director
#
Console {
  Name = hw-10-4-n1-mon
  Password = "qwerty12345"
  CommandACL = status, .status
}

```

---

### Задание 3

Установите программное обеспечении Rsync. Настройте синхронизацию на двух нодах. Протестируйте работу сервиса.

*Пришлите рабочую конфигурацию сервера и клиента Rsync.*

---

### Задание со звёздочкой*
Это задание дополнительное. Его можно не выполнять. На зачёт это не повлияет. Вы можете его выполнить, если хотите глубже разобраться в материале.

---

### Задание 4*

Настройте резервное копирование двумя или более методами, используя одну из рассмотренных команд для папки /etc/default. Проверьте резервное копирование.

*Пришлите рабочую конфигурацию выбранного сервиса по поставленной задаче.*

