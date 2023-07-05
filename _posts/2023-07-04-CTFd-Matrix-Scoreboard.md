---
title: CTFd 比赛计分板插件二次开发
description: 如何把一个插件完善成真正的插件呢?
categories:
- CTFd
tags:
- CTFd
- Scoreboard
date: 2023-07-04
photos: /ctfd/2023/07/04/CTFd-Matrix-Scoreboard/scoreboard.png
---
这个计分板插件最开始是战队里一个学长开发的，能用是能用，但很多东西都是写死的，不够优雅，正好在整合CTFd，进行一手二次开发！

学长原始的仓库地址：https://github.com/Ephemeral1y/ctfd-matrix-scoreboard

二次开发的新的仓库地址：https://github.com/IShiraiKurokoI/CTFd-Matrix-Scoreboard

首先就是处理一下数组越界的问题，原来的插件是指定总题目数量的，如果题目增加了就需要手动修改，这样太不优雅了，修改一下：

```python
...
num = Challenges.query.filter(*[]).order_by(Challenges.id.asc()).all()[-1].id
...
blood = [[] for i in range(num)]
...
```

然后就是解决没有学号的用户导致加分异常的问题：

```python
if get_config("matrix:score_switch"):
    s_user = Users.query.filter_by(id=teamid).first_or_404()
    if s_user.sid:
        if str(s_user.sid[:4]) in str(get_config("matrix:score_grade")):
            score += get_config("matrix:score_num")
```

然后就是动态切换计分板的问题，原来的插件是直接覆盖了主题的计分板模板，这样的话动态切换比较麻烦，修改为：

```python
dir_path = os.path.dirname(os.path.realpath(__file__))
template_path = os.path.join(dir_path, 'scoreboard-matrix.html')
override_template('scoreboard-matrix.html', open(template_path).read())
# 不覆盖原生scoreboard模板，方便切换
app.view_functions['scoreboard.listing'] = scoreboard_view
app.view_functions['scoreboard.score'] = scores
```

函数scoreboard_view也对应修改为：

```python
def scoreboard_view():
    if scores_visible() and not authed():
        return redirect(url_for('auth.login', next=request.path))
    if get_config("matrix:switch"):
        if not scores_visible():
            return render_template('scoreboard-matrix.html',
                                   errors=['当前分数已隐藏'])
        standings = get_standings()
        return render_template('scoreboard-matrix.html', standings=standings,
                               score_frozen=is_scoreboard_frozen(),
                               mode='users' if is_users_mode() else 'teams',
                               challenges=get_challenges(), theme=ctf_theme())
    else:
        freeze = get_config("freeze")
        infos = get_infos()
        if freeze:
            infos.append("计分板已经冻结。")
        if not scores_visible():
            infos.append("当前分数已隐藏。")
        return render_template("scoreboard.html", standings=CTFd.utils.scores.get_standings(), infos=infos)
```

改了这么多有人就要问了，你这配置写哪了，写在开头了：

```python
from CTFd.plugins import (
    register_admin_plugin_menu_bar,
    override_template,
)
from CTFd.utils import get_config, set_config

def setup_default_configs():
    current_year = datetime.datetime.now().year
    year_string = str(current_year)
    for key, val in {
        'setup': 'true',
        'switch': False,
        'score_switch': False,
        'score_grade': year_string,
        'score_num': 1200,
    }.items():
        set_config('matrix:' + key, val)


def load(app):
    plugin_name = __name__.split('.')[-1]
    set_config('matrix:plugin_name', plugin_name)
    app.db.create_all()
    if not get_config("matrix:setup"):
        setup_default_configs()
    if not get_config("matrix:score_switch"):
        set_config("matrix:score_switch", False)
    if not get_config("matrix:score_grade"):
        current_year = datetime.datetime.now().year
        year_string = str(current_year)
        set_config("matrix:score_grade", year_string)
    if not get_config("matrix:score_num"):
        set_config("matrix:score_num", 1200)
    register_admin_plugin_menu_bar(title='比赛计分板',
                                   route='/plugins/matrix/admin/settings')

    page_blueprint = Blueprint("matrix",
                               __name__,
                               template_folder="templates",
                               static_folder="static",
                               url_prefix="/plugins/matrix")
    worker_config_commit = None

    @page_blueprint.route('/admin/settings')
    @admins_only
    def admin_configs():
        nonlocal worker_config_commit
        # 处理GET请求
        if not get_config("matrix:switch") != worker_config_commit:
            worker_config_commit = get_config("matrix:switch")
        return render_template('matrix_config.html')

    app.register_blueprint(page_blueprint)
    
...
```

然后页面的前端模板就不赘述了，详情参看repo：https://github.com/IShiraiKurokoI/CTFd-Matrix-Scoreboard

效果如下：

![image-20230705162332065](image-20230705162332065.png)

![image-20230705162434642](image-20230705162434642.png)

![scoreboard](scoreboard.png)

可通过后端更改开关实时变更计分板样式（主题计分板/比赛计分板）

PS：我们Scr1w战队二次开发的CTFd整合版地址：https://github.com/dlut-sss/CTFD-Public
