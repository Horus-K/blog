---
title: keepalived邮件通知
date: 2020-03-10 14:19:15
tags:
- keepalived
categories: 高可用
---

yum install mailx

脚本与mail.rc所有keeplive机器都要有

<!--more-->

vim /etc/mail.rc
set from=1625317303@qq.com
set smtp=smtp.qq.com
set smtp-auth-user=1625317303@qq.com
set smtp-auth-password=ydqwzayxwuupijjav #邮箱授权码
set smtp-auth=login
set ssl-verify=ignore

编写通知脚本

```
vim /etc/keepalived/check.sh
#!/bin/bash
contact='1625317303@qq.com'
notify() {
mailsubject="$(hostname) to be $1, vip  转移"
mailbody="$(date +'%F %T'): vrrp transition, $(hostname) changed to be $1"
echo "$mailbody" | mail -s "$mailsubject" $contact
}
case $1 in
master)
notify master
;;
backup)
notify backup
;;
fault)
notify fault
;;
*)
echo "Usage: $(basename $0) {master|backup|fault}"
exit 1
;;
esac
notify_master "/etc/keepalived/check.sh master"
notify_backup "/etc/keepalived/check.sh backup"
notify_fault "/etc/keepalived/check.sh fault"
```

![img](keepalived邮件通知/screenshot_20190607_200247.png)

systemctl reload keepalived