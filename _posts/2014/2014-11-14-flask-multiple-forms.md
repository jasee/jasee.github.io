---
layout: post
title: Flask中的单页多表单
category: 编程
description: 在单页面多表单中，通过为表单字段添加前缀实现回收表单后进行表单分辨。
tags: ["Flask", "Wtforms", "Python"]
---

“如果运维人员开发的Web还有一点美观性可言的话，那一定是Bootstrap的功劳”，为了在页面中使用[手风琴][1]效果，就需要面对如何在单个页面上使用多个Wtforms表单的问题。对于暂时没有前端手段的人来说，在没看到类似的Flask实现实例时，也许只能闭门造一个四不像出来了。

计划实现一个用户管理页面，所有用户的查看、修改和删除都可以在该页面进行。思路：

1. 因为用户量基本是个位数，这个页面的性能不用太考虑，可以列出所有用户。(连分页都没有使用)
2. 将用户信息和修改用户表单传递到后端，分别进行用户详细信息的展示和编辑。
3. 修改用户的表单增加用户id作为前缀，在接收到POST请求时，获取该前缀并匹配到对应的表单和需要修改的用户。
4. 根据以上信息进行用户信息更新。

其中第3步是主要的问题，下面是具体的代码。

```py
@manage.route('/index', methods=['GET', 'POST'])
@login_required
@admin_required
def index():
    users = User.query.order_by(User.email).all()
    choices = [(r.id, r.name) for r in Role.query.order_by(Role.name.desc()).all()]
    pairs =[]

    for user in users:
        form = EditUserForm(user.nickname, prefix=str(user.id))
        form.roleid.choices = choices
        form.email.data = user.email
        read_only(form.email)
        pairs.append({'user':user, 'form':form})

    if request.method == 'POST':
        id = int(request.form.lists()[0][0].split('-')[0])
        valid = False
        for pair in pairs:
            if pair['user'].id == id:
                form = pair['form']
                valid = True
                break
        if not valid:
            flash(u'更新用户错误')
            return redirect(url_for('.index'))
        if form.validate_on_submit():
            user = User.query.get(id)
            user.nickname = form.nickname.data
            user.role = Role.query.get(form.roleid.data)
            if form.password.data is not None:
                user.password = form.password.data
            db.session.add(user)
            db.session.commit()
            flash(u'成功更新用户')
        else:
            flash(u'更新用户错误')

    for pair in pairs:
        pair['form'].nickname.data = pair['user'].nickname
        pair['form'].roleid.data = pair['user'].role.id

    return render_template('manage/index.html', pairs=pairs)
```

感觉这种实现还是比较奇怪(ugly)，贴出来一方面是做个记录，给类似我的小白的一点参考，另一方面万一有大神看到了指点我两句呢！
这段代码的单元测试参考[下一篇文章][2]。

[1]: http://v3.bootcss.com/javascript/#collapse-examples
[2]: /2014/11/15/flask-invitation-only-user-management
