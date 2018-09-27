---
title: PyMOTW-3 --- Unix-specific Services
categories: language
tags: PyMOTW-3
---

虽然Python解释器可移植，并且许多标准库模块支持多个平台，但是有一些模块可以访问特定于平台的功能。本节中描述的模块仅在基于Unix的系统（如Linux，macOS，FreeBSD和OpenBSD）上提供访问功能。

## pwd — Unix Password Database

目的：从Unix密码数据库中读取用户数据。  
pwd模块可用于从Unix密码数据库（通常是`/etc/passwd`）读取用户信息。
只读接口返回类似于元组的对象，该对象有命名属性用于密码记录的标准字段。

Index	| Attribute	| Meaning
:---	| :---		| :---
0		| pw_name	| The user’s login name
1		| pw_passwd	| Encrypted password (optional)
2		| pw_uid	| User id (integer)
3		| pw_gid	| Group id (integer)
4		| pw_gecos	| Comment/full name
5		| pw_dir	| Home directory
6		| pw_shell	| Application started on login, usually a command interpreter

### Querying All Users

此示例打印系统上所有“real”（真实）用户的报告，包括其主目录（其中“real”定义为name不以“_”开头）。
要加载整个密码数据库，请使用getpwall（）。 返回值是具有未定义顺序的列表，因此需要在打印报表之前对其进行排序。

```python
#pwd_getpwall.py
import pwd
import operator

# Load all of the user data, sorted by username
all_user_data = pwd.getpwall()
interesting_users = sorted(
    (u for u in all_user_data if not u.pw_name.startswith('_')),
    key=operator.attrgetter('pw_name')
)

# Find the longest lengths for a few fields
username_length = max(len(u.pw_name) for u in interesting_users) + 1
home_length = max(len(u.pw_dir) for u in interesting_users) + 1
uid_length = max(len(str(u.pw_uid)) for u in interesting_users) + 1

# Print report headers
fmt = ' '.join(['{:<{username_length}}',
                '{:>{uid_length}}',
                '{:<{home_length}}',
                '{}'])
print(fmt.format('User',
                 'UID',
                 'Home Dir',
                 'Description',
                 username_length=username_length,
                 uid_length=uid_length,
                 home_length=home_length))
print('-' * username_length,
      '-' * uid_length,
      '-' * home_length,
      '-' * 20)

# Print the data
for u in interesting_users:
    print(fmt.format(u.pw_name,
                     u.pw_uid,
                     u.pw_dir,
                     u.pw_gecos,
                     username_length=username_length,
                     uid_length=uid_length,
                     home_length=home_length))
```
上面的大多数示例代码都可以很好地处理格式化结果。 最后的for循环显示了如何按名称访问记录中的字段。

```
$ python3 pwd_getpwall.py

User               UID Home Dir          Description
---------- ----------- ----------------- --------------------
Guest              201 /Users/Guest      Guest User
daemon               1 /var/root         System Services
daemon               1 /var/root         System Services
dhellmann          501 /Users/dhellmann  Doug Hellmann
nobody      4294967294 /var/empty        Unprivileged User
nobody      4294967294 /var/empty        Unprivileged User
root                 0 /var/root         System Administrator
root                 0 /var/root         System Administrator
```

### Querying User By Name

要读取有关一个用户的信息，不必读取整个密码数据库。 使用getpwnam（），按名称检索有关用户的信息。

```python
#pwd_getpwnam.py
import pwd
import sys

username = sys.argv[1]
user_info = pwd.getpwnam(username)

print('Username:', user_info.pw_name)
print('Password:', user_info.pw_passwd)
print('Comment :', user_info.pw_gecos)
print('UID/GID :', user_info.pw_uid, '/', user_info.pw_gid)
print('Home    :', user_info.pw_dir)
print('Shell   :', user_info.pw_shell)
```
运行此示例的系统上的密码存储在主用户数据库之外的影子文件中，因此密码字段在设置时将报告为全*。

```
$ python3 pwd_getpwnam.py dhellmann

Username: dhellmann
Password: ********
Comment : Doug Hellmann
UID/GID : 501 / 20
Home    : /Users/dhellmann
Shell   : /bin/bash

$ python3 pwd_getpwnam.py nobody

Username: nobody
Password: *
Comment : Unprivileged User
UID/GID : 4294967294 / 4294967294
Home    : /var/empty
Shell   : /usr/bin/false
```

### Querying User By UID

还可以通过其数字用户ID查找用户。这对于查找文件的所有者很有用：

```python
#pwd_getpwuid_fileowner.py
import pwd
import os

filename = 'pwd_getpwuid_fileowner.py'
stat_info = os.stat(filename)
owner = pwd.getpwuid(stat_info.st_uid).pw_name

print('{} is owned by {} ({})'.format(filename, owner, stat_info.st_uid))
```

```
$ python3 pwd_getpwuid_fileowner.py

pwd_getpwuid_fileowner.py is owned by dhellmann (501)
```

数字用户ID也可用于查找有关当前正在运行进程的用户的信息：

```python
#pwd_getpwuid_process.py
import pwd
import os

uid = os.getuid()
user_info = pwd.getpwuid(uid)
print('Currently running with UID={} username={}'.format(uid, user_info.pw_name))
```

```
$ python3 pwd_getpwuid_process.py

Currently running with UID=501 username=dhellmann
```

## grp — Unix Group Database

目的：从Unix group数据库中读取group数据。  
grp模块可用于从组数据库（通常是`/etc/group`）中读取有关Unix组的信息。 只读接口返回类似于元组的对象，这些对象具有命名属性用于组记录的标准字段。

Index	| Attribute	 | Meaning
:---	| :---		 | :---
0		| gr_name	 | Name
1		| gr_passwd	 | Password, if any (encrypted)
2		| gr_gid	 | Numerical id (integer)
3		| gr_mem	 | Names of group members

名称和密码值都是字符串，GID是整数，成员报告为字符串列表。

### Querying All Groups

此示例打印系统上所有“真实”组的报告，包括其成员（其中“real”定义为名称不以“_”开头）。 要加载整个密码数据库，请使用getgrall（）。

```python
#grp_getgrall.py
import grp
import textwrap

# Load all of the user data, sorted by username
all_groups = grp.getgrall()
interesting_groups = {
    g.gr_name: g
    for g in all_groups
    if not g.gr_name.startswith('_')
}
print(len(interesting_groups.keys()))

# Find the longest length for a few fields
name_length = max(len(k) for k in interesting_groups) + 1
gid_length = max(len(str(u.gr_gid)) for u in interesting_groups.values()) + 1

# Set the members field width to avoid table columns wrapping
members_width = 19

# Print report headers
fmt = ' '.join(['{:<{name_length}}',
                '{:{gid_length}}',
                '{:<{members_width}}',
                ])
print(fmt.format('Name',
                 'GID',
                 'Members',
                 name_length=name_length,
                 gid_length=gid_length,
                 members_width=members_width))
print('-' * name_length,
      '-' * gid_length,
      '-' * members_width)

# Print the data
prefix = ' ' * (name_length + gid_length + 2)
for name, g in sorted(interesting_groups.items()):
    # Format members to start in the column on the same line but
    # wrap as needed with an indent sufficient to put the
    # subsequent lines in the members column. The two indent
    # prefixes need to be the same to compute the wrap properly,
    # but the first should not be printed so strip it.
    members = textwrap.fill(
        ', '.join(g.gr_mem),
        initial_indent=prefix,
        subsequent_indent=prefix,
        width=members_width + len(prefix),
    ).strip()
    print(fmt.format(g.gr_name,
                     g.gr_gid,
                     members,
                     name_length=name_length,
                     gid_length=gid_length,
                     members_width=members_width))
```
返回值是具有未定义顺序的列表，因此需要在打印报表之前对其进行排序。

```
$ python3 grp_getgrall.py

34
Name                            GID         Members
------------------------------- ----------- -------------------
accessibility                            90
admin                                    80 root
authedusers                              50
bin                                       7
certusers                                29 root, _jabber,
                                            _postfix, _cyrus,
                                            _calendar, _dovecot
com.apple.access_disabled               396
com.apple.access_ftp                    395
com.apple.access_screensharing          398
com.apple.access_sessionkey             397
com.apple.access_ssh                    399
com.apple.sharepoint.group.1            701 dhellmann
consoleusers                             53
daemon                                    1 root
dialer                                   68
everyone                                 12
group                                    16
interactusers                            51
kmem                                      2 root
localaccounts                            61
mail                                      6 _teamsserver
netaccounts                              62
netusers                                 52
network                                  69
nobody                           4294967294
nogroup                                  -1
operator                                  5 root
owner                                    10
procmod                                   9 root
procview                                  8 root
staff                                    20 root
sys                                       3 root
tty                                       4 root
utmp                                     45
wheel                                     0 root
```

### Group Memberships for a User

另一个常见任务可能是打印给定用户的所有组的列表：

```python
#grp_groups_for_user.py
import grp

username = 'dhellmann'
group_names = set(
    g.gr_name
    for g in grp.getgrall()
    if username in g.gr_mem
)
print(username, 'belongs to:', ', '.join(sorted(group_names)))
```
唯一组名称集(set)在打印之前会进行排序。

```
$ python3 grp_groups_for_user.py

dhellmann belongs to: _appserveradm, _appserverusr, _lpadmin, admin, com.apple.sharepoint.group.1
```

### Finding a Group By Name

与pwd一样，也可以通过名称或数字ID查询有关特定组的信息。

```python
#grp_getgrnam.py
import grp

name = 'admin'
info = grp.getgrnam(name)
print('Name    :', info.gr_name)
print('GID     :', info.gr_gid)
print('Password:', info.gr_passwd)
print('Members :', ', '.join(info.gr_mem))
```
admin组有两个成员：

```
$ python3 grp_getgrnam.py

Name    : admin
GID     : 80
Password: *
Members : root, dhellmann
```

### Finding a Group by ID

要识别运行当前进程的组，请将getgrgid（）与os.getgid（）结合使用。

```python
#grp_getgrgid_process.py
import grp
import os

gid = os.getgid()
group_info = grp.getgrgid(gid)
print('Currently running with GID={} name={}'.format(gid, group_info.gr_name))
```

```
$ python3 grp_getgrgid_process.py

Currently running with GID=20 name=staff
```
要根据文件的权限获取组名，请查找os.stat（）返回的组。

```python
#grp_getgrgid_fileowner.py
import grp
import os

filename = 'grp_getgrgid_fileowner.py'
stat_info = os.stat(filename)
owner = grp.getgrgid(stat_info.st_gid).gr_name

print('{} is owned by {} ({})'.format(filename, owner, stat_info.st_gid))
```
文件状态记录包括文件或目录的所有权和权限数据。

```
$ python3 grp_getgrgid_fileowner.py

grp_getgrgid_fileowner.py is owned by staff (20)
```
