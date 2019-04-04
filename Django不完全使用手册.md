# Django不完全使用手册

## 安装Django：

```
pip install Django
```

验证是否安装成功：

```
import django
print(django.get_version())
```

或者

```
python -m django --version
```

## 初始化

```
django-admin startproject mysite
```

启动服务器于端口（http://127.0.0.1:8000/）：
```
python manage.py runserver
```

## 创建一个应用

```
py manage.py startapp polls
```

至此目录结构应当为：

```
polls/
    __init__.py
    admin.py
    apps.py
    migrations/
        __init__.py
    models.py
    tests.py
    views.py
```

## 创建一个视图
在views.py中输入：
```
from django.http import HttpResponse

def index(request):
    return HttpResponse("Hello, world. You're at the polls index.")
    
```

在 polls 目录里新建一个 urls.py 文件。
打开该文件输入：

```
from django.urls import path

from . import views

urlpatterns = [
    path('', views.index, name='index'),
]
```
urls.py存储的是views.py对应的链接存储的地方。如上面的`view.index`实际上唤起了`view.py`文件中的index变量。

在mysite/urls.py中插入：
```
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path('polls/', include('polls.urls')),
    path('admin/', admin.site.urls),
]
```

`path()`函数有两个必须的参数：route和view。view会寻找一个HttpRequest作为参数。

## 数据库操作

数据库操作主要在setting.py中。TIME_ZONE在其中修改。
创建默认表
```
python manage.py migrate
```

## 重要！模型的创建（即关系）

在`models.py`中放置的是我们的模型以及模型的参数：

如：
```
from django.db import models


class Question(models.Model):
    question_text = models.CharField(max_length=200)
    pub_date = models.DateTimeField('date published')


class Choice(models.Model):
    question = models.ForeignKey(Question, on_delete=models.CASCADE)
    choice_text = models.CharField(max_length=200)
    votes = models.IntegerField(default=0)
```

在choice中，以question作为外键，并有一个自己的字符串数据和一个投票的计数器。而question模型中，仅有投票的名字以及更新的时间。
Field具体的参数请在使用时留意并自行查询。

## 迁移

修改setting.py，在其中APPS中添加：
```
'polls.apps.PollsConfig',
```
运行下述以载入对象。

```python manage.py makemigrations polls```


自动数据库迁移：
```
python manage.py sqlmigrate polls 0001
python manage.py migrate
```

>迁移是非常强大的功能，它能让你在开发过程中持续的改变数据库结构而不需要重新删除和创建表 - 它专注于使数据库平滑升级而不会丢失数据。

## 管理员
创建管理员账号：
```
python manage.py createsuperuser
```
运行服务器：
```
python manage.py runserver
```
载入[这个链接](http://127.0.0.1:8000/admin/)以查看管理员账号。

## 重要！载入管理功能

在admin.py中修改
```python
from django.contrib import admin

from .models import Question

admin.site.register(Question)
```

执行以上两步会使你可以在管理员界面中看见相应的东西。

## 更多关于视图的内容

视图的添加是在polls/views.py文件中。格式为：
```python
def results(request, question_id):
    response = "You're looking at the results of question %s."
    return HttpResponse(response % question_id)
```

返回值为HttpResponse作为返回字符串。与之同时需要将相关信息载入urls.py。
```python
 path('<int:question_id>/results/', views.results, name='results'),
```

>当某人请求你网站的某一页面时 —— 比如说， "/polls/34/" ，Django 将会载入 mysite.urls 模块，因为这在配置项 ROOT_URLCONF 中设置了。然后 Django 寻找名为 urlpatterns 变量并且按序匹配正则表达式。在找到匹配项 'polls/'，它切掉了匹配的文本（"polls/"），将剩余文本 ——"34/"，发送至 'polls.urls' URLconf 做进一步处理。
>

## 显示最近的问题：
在views.py中添加：
```python
from django.http import HttpResponse

from .models import Question


def index(request):
    latest_question_list = Question.objects.order_by('-pub_date')[:5]
    output = ', '.join([q.question_text for q in latest_question_list])
    return HttpResponse(output)

```

## 使用模板
在polls目录下创建templates目录。在其中再建一个polls目录，在其中放入.html文档，如index.html

写入index.html文档。如：
```html
{% if latest_question_list %}
    <ul>
    {% for question in latest_question_list %}
        <li><a href="/polls/{{ question.id }}/">{{ question.question_text }}</a></li>
    {% endfor %}
    </ul>
{% else %}
    <p>No polls are available.</p>
{% endif %}
```
其中的变量直接调用views.py中的变量。
然后更新views.py以使用html：
```python
from django.shortcuts import render
from .models import Question
def index(request):
    latest_question_list = Question.objects.order_by('-pub_date')[:5]
    context = {'latest_question_list': latest_question_list}
    return render(request, 'polls/index.html', context)
```
> The render() function takes the request object as its first argument, a template name as its second argument and a dictionary as its optional third argument. It returns an HttpResponse object of the given template rendered with the given context.
> 

## 去除硬编码

为了防止重复编写因为id不一样的不同界面：
```html
<li><a href="{% url 'detail' question.id %}">{{ question.question_text }}</a></li>
```
这里的url后面链接的名字是在urls.py中定义的名字。

## 一个很强的示例
vite()的界面，在views.py中：
```python
from django.http import HttpResponse, HttpResponseRedirect
from django.shortcuts import get_object_or_404, render
from django.urls import reverse

from .models import Choice, Question
# ...
def vote(request, question_id):
    question = get_object_or_404(Question, pk=question_id)
    try:
        selected_choice = question.choice_set.get(pk=request.POST['choice'])
    except (KeyError, Choice.DoesNotExist):
        # Redisplay the question voting form.
        return render(request, 'polls/detail.html', {
            'question': question,
            'error_message': "You didn't select a choice.",
        })
    else:
        selected_choice.votes += 1
        selected_choice.save()
        # Always return an HttpResponseRedirect after successfully dealing
        # with POST data. This prevents data from being posted twice if a
        # user hits the Back button.
        return HttpResponseRedirect(reverse('polls:results', args=(question.id,)))
```

函数解释：
- request.POST['choice']。类似字典，返回choice对应的id