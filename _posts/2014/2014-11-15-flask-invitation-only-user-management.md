---
layout: post
title: Flask邀请制用户管理
category: 编程
description: 通过修改权限时的检查和禁止直接删除管理员账户的方式，保证在管理员邀请制服务中保留至少一个管理员。
tags: ["Flask", "Unittest", "Python"]
---

相比于用户注册管理员审核的机制，由管理员直接增加用户的“邀请制”显然更加简单，在小范围的使用中也比较方便。邀请制最大的问题是，至少得有一个管理员。服务初始化添加管理员后，如何保证不掉进没有管理员的尴尬坑呢？

### 修改用户权限时检查

如果系统中只有一个管理员，而这个管理员又在尝试降低自己的权限，这是绝对不能允许的，完成交班之前哪儿也不能去。以下是参考代码：

```py
class EditUserForm(Form):
    email = StringField(u'邮箱', validators=[Required(), Length(1, 64), Email()])
    nickname = StringField(u'昵称', validators=[Required(), Length(1, 64)])
    roleid = SelectField(u'权限', coerce=int)
    password = PasswordField(u'密码', filters=[lambda x: x or None])
    submit = SubmitField(u'更新用户')

    def __init__(self, original_nickname, *args, **kwargs):
        Form.__init__(self, *args, **kwargs)
        self.original_nickname = original_nickname

    def validate(self):
        if not Form.validate(self):
            return False
        if User.query.filter_by(email=self.email.data).first().role.name == 'Administrator':
            adminid = Role.query.filter_by(name='Administrator').first().id
            if self.roleid.data != adminid:
                if len(User.query.filter_by(role_id=adminid).all()) < 2:
                    self.roleid.errors.append(u'至少保留一个Administrator')
                    return False
        if self.nickname.data == self.original_nickname:
            return True
        user = User.query.filter_by(nickname=self.nickname.data).first()
        if user != None:
            self.nickname.errors.append(u'昵称已存在，请重新选择')
            return False
        return True
```

### 不允许删除管理员账户

“先打倒再干掉”是一个比较符合逻辑的过程，如果不允许直接删除管理员账户，结合上面的修改检查，就可以保证系统一直至少有一个管理员存在了。以下是参考代码：

```py
@manage.route('/del-user/<id>')
@login_required
@admin_required
def del_user(id):
    user = User.query.get(id)
    if user is not None:
        if user.role.name != 'Administrator':
            db.session.delete(user)
            db.session.commit()
            flash(u'成功删除用户')
        else:
            flash(u'无法删除管理员账户，请降权后删除')
            return redirect(url_for('.index'))
    else:
        abort(404)
    return redirect(url_for('.index'))
```

### 单元测试

测试自己程序的逻辑还是很有必要的，以下是两个测试用例：

```py
def test_edit_user(self):
    self.add_user('foo@example.com', 'Administrator')
    self.add_user('bar@example.com', 'User')
    self.login_user('foo@example.com', 'foo')
    rid = Role.query.filter_by(name='Moderator').first().id
    # 拒绝无效的用户id
    data={'0-email': 'bar@example.com',
            '0-nickname': 'foobar',
            '0-roleid': rid,
            '0-password': None}
    rv = self.client.post(url_for('manage.index'), data=data, follow_redirects=True)
    self.assertTrue(b'更新用户错误' in rv.data)
    # 至少存留一个管理员
    uid = str(User.query.filter_by(email='foo@example.com').first().id)
    data={uid+'-email': 'foo@example.com',
            uid+'-nickname': 'foo',
            uid+'-roleid': rid,
            uid+'-password': None}
    rv = self.client.post(url_for('manage.index'), data=data, follow_redirects=True)
    self.assertTrue(b'至少保留一个Administrator' in rv.data)
    # 保持昵称唯一
    uid = str(User.query.filter_by(email='bar@example.com').first().id)
    rid = Role.query.filter_by(name='Administrator').first().id
    data={uid+'-email': 'bar@example.com',
            uid+'-nickname': 'foobar',
            uid+'-roleid': rid,
            uid+'-password': None}
    data.update({uid+'-nickname': 'foo'})
    rv = self.client.post(url_for('manage.index'), data=data, follow_redirects=True)
    self.assertTrue(b'昵称已存在' in rv.data)
    # 拒绝无效的权限id
    data.update({uid+'-nickname': 'foobar', uid+'-roleid': -1})
    rv = self.client.post(url_for('manage.index'), data=data, follow_redirects=True)
    self.assertTrue(b'Not a valid choice' in rv.data)
    # 密码字段提供与否都可以通过
    data.update({uid+'-roleid': rid})
    rv = self.client.post(url_for('manage.index'), data=data, follow_redirects=True)
    self.assertTrue(b'成功更新用户' in rv.data)
    data.update({uid+'-password': 'foobar'})
    rv = self.client.post(url_for('manage.index'), data=data, follow_redirects=True)
    self.assertTrue(b'成功更新用户' in rv.data)
    # bar以及是管理员了，现在foo应该可以给自己降权
    rid = Role.query.filter_by(name='Moderator').first().id
    uid = str(User.query.filter_by(email='foo@example.com').first().id)
    data={uid+'-email': 'foo@example.com',
            uid+'-nickname': 'foo',
            uid+'-roleid': rid,
            uid+'-password': None}
    rv = self.client.post(url_for('manage.index'), data=data, follow_redirects=True)
    self.assertTrue(b'成功更新用户' in rv.data)
    rv = self.client.get(url_for('main.account'))
    self.assertTrue(b'用户角色: Moderator' in rv.data)
    self.client.get(url_for('main.logout'), follow_redirects=True)
    # bar可以使用其新密码登陆并具有管理员权限
    self.login_user('bar@example.com', 'foobar')
    rv = self.client.get(url_for('main.account'))
    self.assertTrue(b'用户角色: Administrator' in rv.data)

def test_del_user(self):
    self.add_user('foo@example.com', 'Administrator')
    self.add_user('bar@example.com', 'Administrator')
    self.login_user('foo@example.com', 'foo')
    # 无法直接删除管理员账户
    uid = User.query.filter_by(email='bar@example.com').first().id
    rv = self.client.get(url_for('manage.del_user', id=-1))
    self.assertEqual(rv.status_code, 404)
    rv = self.client.get(url_for('manage.del_user', id=uid), follow_redirects=True)
    self.assertTrue(b'无法删除管理员账户' in rv.data)
    # 降权后可删除
    rid = Role.query.filter_by(name='Moderator').first().id
    data={str(uid)+'-email': 'bar@example.com',
            str(uid)+'-nickname': 'bar',
            str(uid)+'-roleid': rid,
            str(uid)+'-password': None}
    rv = self.client.post(url_for('manage.index'), data=data, follow_redirects=True)
    self.assertTrue(b'成功更新用户' in rv.data)
    rv = self.client.get(url_for('manage.del_user', id=uid), follow_redirects=True)
    self.assertTrue(b'成功删除用户' in rv.data)
    self.assertFalse(b'bar@example.com' in rv.data)
```
