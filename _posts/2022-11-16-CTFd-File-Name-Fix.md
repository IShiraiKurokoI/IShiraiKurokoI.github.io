---
title: CTFd 附件名称问题
description: 解决中日韩名称上传后只剩后缀的问题
categories:
- CTFd
tags:
- CTFd
- Security
date: 2022-11-16
---
在使用CTFd上传题目附件时候发现题目文件只能使用英文，中文名文件上传后会直接只剩拓展名。
原因在于CTFd使用了flask原生的filename = secure_filename(filename)，导致中文名被抹去。

于是修复如下：

/CTFd/utils/__init__.py 增加如下：

```python
import unicodedata
import os
import re

_windows_device_files = (
    "CON",
    "AUX",
    "COM1",
    "COM2",
    "COM3",
    "COM4",
    "LPT1",
    "LPT2",
    "LPT3",
    "PRN",
    "NUL",
)


def fixed_secure_filename(filename: str) -> str:
    filename = unicodedata.normalize("NFKD", filename)
    filename = filename.encode("utf8", "ignore").decode("utf8")  # 编码格式改变
    for sep in os.path.sep, os.path.altsep:
        if sep:
            filename = filename.replace(sep, " ")
    _filename_utf8_add_strip_re = re.compile(r'[^A-Za-z0-9_\u2E80-\uFE4F.-]')
    filename = str(_filename_utf8_add_strip_re.sub('', '_'.join(filename.split()))).strip('._')  # 添加新规则
    if (
            os.name == "nt"
            and filename
            and filename.split(".")[0].upper() in _windows_device_files
    ):
        filename = f"_{filename}"
    return filename
```

/CTFd/api/v1/challenges.py 修改如下：

```python
		import urllib
		
		...
        if authed():
            user = get_current_user()
            team = get_current_team()

            # TODO: Convert this into a re-useable decorator
            if is_admin():
                pass
            else:
                if config.is_teams_mode() and team is None:
                    abort(403)

            unlocked_hints = {
                u.target
                for u in HintUnlocks.query.filter_by(
                    type="hints", account_id=user.account_id
                )
            }
            files = []
            for f in chal.files:
                token = {
                    "user_id": user.id,
                    "team_id": team.id if team else None,
                    "file_id": f.id,
                }
                files.append(urllib.parse.unquote(
                    url_for("views.files",
                            path=f.location,
                            token=serialize(token))))
        else:
            files = [
                urllib.parse.unquote(url_for("views.files", path=f.location)) for f in chal.files
            ]
        ...
```

/CTFd/utils/uploads/uploaders.py 修改如下：

```python
 from CTFd.utils import fixed_secure_filename
```

```python
def upload(self, file_obj, filename):
    if len(filename) == 0:
        raise Exception("Empty filenames cannot be used")

    filename = fixed_secure_filename(filename)
    md5hash = hexencode(os.urandom(16))
    file_path = posixpath.join(md5hash, filename)

    return self.store(file_obj, file_path)
```

```python
def upload(self, file_obj, filename):
    filename = filter(
        self._clean_filename, fixed_secure_filename(filename).replace(" ", "_")
    )
    filename = "".join(filename)
    if len(filename) <= 0:
        return False

    md5hash = hexencode(os.urandom(16))

    dst = md5hash + "/" + filename
    self.s3.upload_fileobj(file_obj, self.bucket, dst)
    return dst
```

完活！

另外为了让前端好看点，让文件按钮占满全宽：

/CTFd/themes/主题名称/templates/challenge.html

![image-20230630174156014](image-20230630174156014.png)

将col-md-4 col-sm-4的class改为 col-md-12 col-sm-12 

完活下机。
