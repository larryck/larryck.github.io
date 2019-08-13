---
title:  neutron l3 agent fail with "Version does not support indexing"
description: 
categories:
- openstack
tags:
- bug
---

# 错误现象
a. vrouter未创建
b. neutron-l3-agent服务fail
c. neutron-l3-agent log显示 "versionutils.is_compatible 'Version' object does not support indexing" 错误

# 已有的方案，未解决问题
社区已有一个相同问题的的[解决方案](https://review.opendev.org/#/c/141653/)，提到：

```
Setuptools 8 was released to PyPI yesterday, it returns a
Version object when parse_version is called. Earlier we used
to get a tuple of strings. Fortunately the Version object
has an iterator as well. So adding a list() will support both
cases
```

patch

```
--- a/nova/openstack/common/versionutils.py
+++ b/nova/openstack/common/versionutils.py
@@ -194,8 +194,8 @@ def is_compatible(requested_version, current_version, same_major=True):
         True.
     :returns: True if compatible, False if not
     """
-    requested_parts = pkg_resources.parse_version(requested_version)
-    current_parts = pkg_resources.parse_version(current_version)
+    requested_parts = list(pkg_resources.parse_version(requested_version))
+    current_parts = list(pkg_resources.parse_version(current_version))
 
     if same_major and (requested_parts[0] != current_parts[0]):
         return False
```
这个修改在我的环境上不生效，继续报"TypeError: 'Version' object is not iterable"错误。


# 解决方案
通过阅读versionutils.py以及Setuptools相关代码我们发现，Version对象有一个public属性正好提供了代码中的requested_parts[0]的功能，因而我们的解决方案是将
    if same_major and (requested_parts[0] != current_parts[0]):
修改成
    if same_major and (requested_parts.public != current_parts.public):

