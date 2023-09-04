---
title: CTFd Whale镜像上传和镜像更新功能
description: 镜像上传更新，一键完成
categories:
- CTFd
tags:
- CTFd
- Scoreboard
date: 2023-09-04
photos: /ctfd/2023/09/04/Scoreboard-Remaster/scoreboard.png
---
之前对学长的计分板进行了二次开发，最近要准备办招新赛和校赛，需要团队模式，一看团队模式之前的计分板插件完全用不了了，索性直接推倒重构。

首先就是对standings的处理，原先学长的方法是通过多层嵌套循环查询数据表来获得数据，只能说确实能跑起来。我们直接把数据库查询都单独拿出循环外面，然后循环内只处理数据，能优化不少性能，代码修改如下：

```python
def get_score_by_challenge_id(challenges, challenge_id):
    for challenge in challenges:
        if challenge.id == challenge_id:
            return challenge.value
    return None

# @cache.memoize(timeout=60)
def get_matrix_standings():
    # 获取所有题目数据并获取数目
    challenges = Challenges.query.filter(*[]).order_by(Challenges.id.asc()).all()
    # 获取所有分数数据和解题数据，并过滤无效数据
    solves = db.session.query(Solves.date.label('date'), Solves.challenge_id.label('challenge_id'),
                              Solves.user_id.label('user_id'),
                              Solves.team_id.label('team_id')).all()
    awards = db.session.query(Awards.user_id.label('user_id'), Awards.team_id.label('team_id'),
                              Awards.value.label('value'), Awards.date.label('date')).all()
    freeze = utils.get_config('freeze')
    if freeze:
        freeze = unix_time_to_utc(freeze)
        solves = solves.filter(Solves.date < freeze)
        awards = awards.filter(Awards.date < freeze)

    # 创建一个字典来存储每个challenge_id的前三条数据
    top_solves = defaultdict(list)
    # 将solves数据按照challenge_id和date排序
    sorted_solves = sorted(solves, key=lambda x: (x.challenge_id, x.date))
    # 遍历排序后的solves数据
    for solve in sorted_solves:
        challenge_id = solve.challenge_id
        # 检查是否已经存储了三条数据，如果是，跳过
        if len(top_solves[challenge_id]) >= 3:
            continue
        # 否则将数据添加到对应的challenge_id中
        top_solves[challenge_id].append({
            'date': solve.date,
            'user_id': solve.user_id,
            'team_id': solve.team_id
        })

    mode = get_config("user_mode")
    if mode == TEAMS_MODE:
        teams = db.session.query(Teams.id.label('team_id'), Teams.name.label('name')).all()
        matrix_scores = []
        for team in teams:
            team_solves = [solve for solve in solves if solve[3] == team.team_id]
            total_score = 0
            team_status = []
            for solve in team_solves:
                challenge_id = solve[1]
                rank = 4
                for index, top_solve in enumerate(top_solves[challenge_id]):
                    if top_solve['team_id'] == team.team_id:
                        rank = index + 1
                # 按血加分
                score = get_score_by_challenge_id(challenges, challenge_id)
                if rank == 1:
                    score *= Decimal('1.1')
                elif rank == 2:
                    score *= Decimal('1.05')
                elif rank == 3:
                    score *= Decimal('1.03')
                total_score += score

                # 记录解决状态和排名
                team_status.append({'challenge_id': challenge_id, 'rank': rank})

            award_value = 0
            # 奖项加分
            for award in awards:
                if award.team_id == team.team_id:
                    total_score += award.value
                    award_value += award.value

            matrix_scores.append(
                {'name': team.name, 'id': team.team_id, 'total_score': total_score,
                 'challenge_solved': team_status , 'award_value': award_value})
        matrix_scores.sort(key=lambda x: x['total_score'], reverse=True)
        return matrix_scores
    else:
        users = db.session.query(Users.id.label('user_id'), Users.name.label('name'), Users.sid.label('sid')).all()
        matrix_scores = []
        for user in users:
            user_solves = [solve for solve in solves if solve[2] == user.user_id]
            total_score = 0
            user_status = []
            for solve in user_solves:
                challenge_id = solve[1]
                rank = 4
                for index, top_solve in enumerate(top_solves[challenge_id]):
                    if top_solve['user_id'] == user.user_id:
                        rank = index + 1
                # 按血加分
                score = get_score_by_challenge_id(challenges, challenge_id)
                if rank == 1:
                    score *= Decimal('1.1')
                elif rank == 2:
                    score *= Decimal('1.05')
                elif rank == 3:
                    score *= Decimal('1.03')
                total_score += score

                # 记录解决状态和排名
                user_status.append({'challenge_id': challenge_id, 'rank': rank})
            award_value = 0
            # 新生加分
            if get_config("matrix:score_switch"):
                if user.sid:
                    if str(user.sid[:4]) in str(get_config("matrix:score_grade")):
                        total_score += get_config("matrix:score_num")
                        award_value += get_config("matrix:score_num")
            # 奖项加分
            for award in awards:
                if award.user_id == user.user_id:
                    total_score += award.value
                    award_value += award.value

            matrix_scores.append(
                {'name': user.name, 'id': user.user_id, 'total_score': total_score,
                 'challenge_solved': user_status , 'award_value': award_value})
        matrix_scores.sort(key=lambda x: x['total_score'], reverse=True)
        return matrix_scores
```

把所有排名数据只通过一个JSON对象返回模板引擎进行渲染，干净多了，另外也实现了个人赛时给新生加分和ctfd的award加分计算。

另外一部分就是对于challenges这个对象的处理，因为原先写了很多判断，分到各个类别需要很多if，这里我们用字典的形式优化下处理方法：

```python
categories = []

def get_challenges():
    global categories
    categories = []
    if not is_admin():
        if not ctftime():
            if view_after_ctf():
                pass
            else:
                return []
    if challenges_visible() and (ctf_started() or is_admin()):
        challenges = db.session.query(
            Challenges.id,
            Challenges.name,
            Challenges.category,
            Challenges.value
        ).filter(or_(Challenges.state != 'hidden', Challenges.state is None)).order_by(
            Challenges.category.asc()).all()
        category_counts = defaultdict(int)
        challenges_list = []
        for x in challenges:
            challenges_list.append({
                'id': x.id,
                'name': x.name,
                'category': x.category.upper(),
                'value': x.value,
            })
            category_counts[x.category.upper()] += 1
        for category, count in category_counts.items():
            categories.append({
                'category': category.upper(),
                'count': count
            })
        categories = sorted(categories, key=lambda x: x['category'])
        return challenges_list
    return []
```

这不就干净多了？然后再稍微改下处理get请求的地方：

```python
def scoreboard_view():
    language = request.cookies.get("Scr1wCTFdLanguage", "zh")
    if scores_visible() and not authed():
        return redirect(url_for('auth.login', next=request.path))
    if get_config("matrix:switch"):
        if not scores_visible():
            if language == "zh":
                return render_template('scoreboard-matrix.html',
                                       errors=['当前分数已隐藏'])
            else:
                return render_template('scoreboard-matrix.html',
                                       errors=['Score is currently hidden'])
        standings = get_matrix_standings()
        return render_template('scoreboard-matrix.html',
                               standings=standings,
                               score_frozen=is_scoreboard_frozen(),
                               mode='users' if is_users_mode() else 'teams',
                               challenges=get_challenges(),
                               categories=categories,
                               theme=ctf_theme())
    else:
        freeze = get_config("freeze")
        infos = get_infos()
        if language == "zh":
            if freeze:
                infos.append("计分板已经冻结。")
            if not scores_visible():
                infos.append("当前分数已隐藏。")
        else:
            if freeze:
                infos.append("Scoreboard is frozen")
            if not scores_visible():
                infos.append("Score is currently hidden")
        return render_template("scoreboard.html", standings=CTFd.utils.scores.get_standings(), infos=infos)
```

这不就好了？

当然，重构了后端函数之后，前端的渲染方式也需要重写，我们将scoreboard的模板移动到templates目录下方便动态加载（原来学长写的是用的override覆盖的模板。。）：

首先就是表头类别的处理，既然改成了动态渲染表头，那么就改成这样写：

```jinja2
<thead>
    <tr>
        <th rowspan="3" style="border-top: 0"><b>{{"Place" if en else "排名"}}</b>
        </th>
        {% if get_config('user_mode') == 'teams' %}
        <th rowspan="3" style="text-align: center;border-top: 0"><b>{{"Team" if en else "队伍名称"}}</b>
        </th>
        {% else %}
        <th rowspan="3" style="text-align: center;border-top: 0"><b>{{"Username" if en else "用户昵称"}}</b>
        </th>
        {% endif %}
        <th rowspan="3" class="right-line" style="border-top: 0"><b>{{"Score" if en else "得分"}}</b>
        </th>
        {% for cat in categories %}
            <th data-halign="center" class="table-category-header" data-align="center" colspan="{% if loop.last %}{{ cat.count + 2 }}{% else %}{{ cat.count }}{% endif %}" style="text-align: center;background-color: {{color_hash(cat.category)}};">
                <b>{{ cat.category }}</b>
            </th>
        {% endfor %}
        <th data-halign="center" class="table-category-header" data-align="center" colspan="4" style="text-align: center;background-color: {{color_hash('奖项')}};">
            <b>{{"Award" if en else "奖项"}}</b>
        </th>
    </tr>
    <tr>
        {% for chal in challenges %}
        <div>
            <th class="chalname" title="{{ chal.category }}">
                <div><span>{{ chal.name }}</span></div>
            </th>
        </div>
        {% endfor %}
        <div>
            <th class="chalname right-line" colspan="2">
                <div><span></span></div>
            </th>
        </div>
        <div>
            <th class="right-line" style="border-bottom: 0;vertical-align: bottom;padding-bottom: 0;horiz-align: center" colspan="2">
                <div><span>{{"Award" if en else "奖励"}}</span></div>
            </th>
        </div>
    </tr>
    <tr>
        {% for chal in challenges %}
        <div>
            <th width="10%" class="score" title="{{ chal.category }}">
                <div><span>{{ chal.value }}</span></div>
            </th>
        </div>
        {% endfor %}
        <div>
            <th width="10%" class="score right-line" colspan="2">
                <div><span></span></div>
            </th>
        </div>
        <div>
            <th width="10%" class="score right-line" style="border-top: 0;vertical-align: top;padding-top: 0;horiz-align: center" colspan="2">
                <div><span>{{"Score" if en else "分数"}}</span></div>
            </th>
        </div>
    </tr>
</thead>
```

这里的colorhash是一个全局定义的python函数：

```python
def color_hash(text):
    hash_value = 0
    for char in text:
        hash_value = ord(char) + ((hash_value << 5) - hash_value)
        hash_value = hash_value & hash_value

    # 计算HSL值
    h = ((hash_value % 360) + 360) % 360
    s = (((hash_value % 25) + 25) % 25) + 75
    l = (((hash_value % 20) + 20) % 20) + 40

    return f'hsl({h}, {s}%, {l}%)'

app.add_template_global(color_hash, 'color_hash')
```

然后表格内容直接使用standings进行渲染：

```jinja2
{% raw %}
<tbody>
    {% for team in standings %}
    <tr>
        <td class="left-line">{{ loop.index }}</td>
        <td ><a href="{{ request.script_root }}/{{ mode }}/{{ team.id }}" target="_blank">{{ team.name }}</a></td>
        <td class="right-line">{{ team.total_score }}</td>
        {% for chal in challenges %}
        <td class="chalmark">
            {% for solved_challenge in team.challenge_solved %}
                {% if solved_challenge.challenge_id == chal.id %}
                    {% set rank = solved_challenge.rank %}
                    {% if rank == 1 %}
                        <img src="/plugins/matrix/static/medal1.png" width="100%" height="100%">
                    {% elif rank == 2 %}
                        <img src="/plugins/matrix/static/medal2.png" width="100%" height="100%">
                    {% elif rank == 3 %}
                        <img src="/plugins/matrix/static/medal3.png" width="100%" height="100%">
                    {% else %}
                        <img src="/plugins/matrix/static/flag.png" width="100%" height="100%">
                    {% endif %}
                {% endif %}
            {% endfor %}
        </td>
        {% endfor %}
        <td class="chalmark" colspan="2">
        </td>
        <td class="chalmark left-line right-line" colspan="2">
            {{ team.award_value }}
        </td>
    </tr>
    {% endfor %}
</tbody>
{% endraw %}
```

这样就好了，修复了之前最后一列过小的问题，加入了奖励分数和额外加分项的计算和渲染，优化了视觉效果和计算性能，完活下机！

项目地址：https://github.com/IShiraiKurokoI/CTFd-Matrix-Scoreboard
