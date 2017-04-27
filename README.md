# A Concise Introduction to Django

**[Nathan Schneider](http://nathan.cl)**  
2016-04-30

Django is a powerful and popular web framework built on Python. It has extensive [documentation](https://docs.djangoproject.com/en/1.9/). I have compiled the notes below mostly by working through the [tutorial](https://docs.djangoproject.com/en/1.9/intro/). Most of the code examples are from the tutorial's sample web poll app.

# Basic architecture

A **project** builds a website out of **apps** (pluggable components). At its core, each project/app consists of:

1. **Models** (models.py): Python objects backed by a database (SQLite by default). Each model is defined as a Python class, and automatically built into a database table. The structure of the class determines the database table schema. From a model, we can query and update its **objects** (instances/records).

2. **URLconfs** (urls.py): when the user navigates to a URL, it is matched against regex patters in a module called the URLconf to decide which view will handle the HTTP request, possibly extracting portions of the URL to pass as variables to the view. There is a project-level URLconf along with app-level URLconfs; each app is associated with a URL prefix that is recognized at the project level and then stripped off for processing the rest of the URL at the app level.

3. **Views** (views.py): functions/classes that handle HTTP requests, typically querying/updating models and providing data to templates (or issuing an HTTP error). Views handle directly navigated URLs as well as form submissions (GET or POST).

4. **Templates** (templates/appname/templatename.html): these **render** information in client-side form (primarily, HTML). The data passed into the template for rendering is called the **context**. (Templates do not access models directly, only what is provided in the context.) The .html file must follow a template language ([Django's default template language](https://docs.djangoproject.com/en/1.9/topics/templates/#the-django-template-language)).

In the tutorial example, all models, views, and templates are at the app level, but there may be scenrios in which some should be at the project level. 

Each project also encapsulates command-line utilities and an admin web portal.


# Details

## project admin CLI

* create stub project: `django-admin startproject <sitename>`. in the project:
  - outer directory with manage.py, which provides a developer CLI
  - inner directory: Python package, including
    * settings.py
    * urls.py: associate URL regexes with views
* manage.py commands: `python manage.py`...
  - `runserver [port]`: start a dev version of the project on localhost. code changes will trigger an automatic restart.
  - `startapp <appname>`: create stub app within the project
  - `makemigrations <appname>`: prepare modifications to database schema given changes to models. these are stored in migrations/ subdirectory of the app with a name like 0001_initial.py
  - `migrate`: carry out pending migrations, creating/updating database schemas for apps in `INSTALLED_APPS`
  - `shell`: interactive Python shell for the project.

## app-level files
  - views.py: HTTP handlers—process requests and produce responses (a page or an error such as `Http404`)
    ```py
    from django.http import HttpResponse
    from django.shortcuts import get_object_or_404, get_list_or_404, render
    from django.template import loader
    from .models import Question
    def index_simple(request):
        return HttpResponse("Hello, world. You're at the polls index.")
      
    def detail_simple(request, question_id): # question_id is an arg from a URL regex group
        return HttpResponse("You're looking at question %s." % question_id)
    
    def index(request):
        latest_question_list = Question.objects.order_by('-pub_date')[:5]
        template = loader.get_template('polls/index.html')
        context = {'latest_question_list': latest_question_list}
        return render(request, 'polls/index.html', context)
        # equivalently: return HttpResponse(template.render(context, request))
    
    def detail(request, question_id):
        # try Question.objects.get(pk=question_id)
        question = get_object_or_404(Question, pk=question_id)
        return render(request, 'polls/detail.html', {'question': question})
    ```
  - urls.py: URLs within the app (appended to project part of URL)
   
    ```py
    from django.conf.urls import url
    from . import views
    urlpatterns = [
        url(r'^$', views.index, name='index'),
        url(r'^(?P<question_id>[0-9]+)/$', views.detail, name='detail'),
    ]
    ```
    * project urls.py can `include('appname.urls')`
    * app urls.py can specify a namespace by setting `app_name`
  - models.py: define classes to serve as database tables and class fields (holding a `models.Field` subclass) to serve as columns
    ```py
    from django.db import models
    class Question(models.Model):
        question_text = models.CharField(max_length=200)
        pub_date = models.DateTimeField('date published')
        # database column names: question_text, pub_date
        # 'date published' is a human-readable name
        # form fields will be validated to enforce specifications
        
    class Choice(models.Model):
        question = models.ForeignKey(Question, on_delete=models.CASCADE)
        # foreign key: links to a record in another table
        choice_text = models.CharField(max_length=200)
        votes = models.IntegerField(default=0)
    ```
    * project settings.py needs to register the app in `INSTALLED_APPS`. Then `migrate` will know to take care of database operations for the app's models.
    * `id` field added automatically as a primary key. `pk` is an alias for the primary key.
    * it is recommended to define a `__str__()` method for each model class
    * can explore the models and data with `manage.py shell`:
    ```py
    # Import the model classes we just wrote.
    >>> from polls.models import Question, Choice
    # access the records
    >>> Question.objects.all()
    # query with constraints
    >>> Question.objects.filter(id=3)
    >>> Question.objects.filter(question_text__startswith='What')
    # chaining of fields
    >>> Choice.objects.filter(question__question_text__startswith='What') # all choices whose question has text starting with...
    # Can instantiate a Question, and call .save() on it to add a record.
    # Because Choice uses Question as a foreign key, we can reverse-access from a question q with:
    >>> q.choice_set
    # and can even create new Choice records via a Question record:
    >>> c = q.choice_set.create(choice_text='asdf', votes=3)
    # a record can also be deleted
    >>> c.delete()
    ```

## web admin

database admin interface at /admin: edit data (not schema) for models registered in admin.py:

```py
from django.contrib import admin

# Register your models here.
from .models import Question, Choice

admin.site.register(Question)
admin.site.register(Choice)
```

## templates

templates go in `templates/<appname>` subdirectory of app
  - [template language ref](https://docs.djangoproject.com/en/1.9/topics/templates/#the-django-template-language)
  - the view is responsible for passing data (**context**) to the template
  - polls/templates/polls/index.html:
      ```html
    {% if latest_question_list %}
    <ul>
    {% for question in latest_question_list %}
        <li><a href="{% url 'detail' question.id %}">{{ question.question_text }}</a></li>
    {% endfor %}
    </ul>
    {% else %}
    <p>No polls are available.</p>
    {% endif %}
      ```
  - `{% url <urlname> <args> %}` to construct a URL based on urls.py
    * arguments may be positional XOR keyword
    * `{% url <urlname> <args> as v %}` stores the URL in a variable
    * trying to construct a nonexistent URL will cause an error unless `as` syntax is used
    * if urls.py is namespaced with an `app_name`, then use `{% url <appname>:<urlname> <args> %}`
    * the `url` command is called a *tag*. [url tag ref](https://docs.djangoproject.com/en/1.9/ref/templates/builtins/#url)


- templates can be composed with the [`include` tag](https://docs.djangoproject.com/en/1.9/ref/templates/builtins/#include): `{% include "foo/bar.html" %}`
    * included template is rendered with the same context as the including template
- **[template inheritance](https://docs.djangoproject.com/en/1.9/ref/templates/language/#template-inheritance):** a template can define a skeleton with named blocks to be filled in by extending (inheriting) templates.
    * base template:
    ```html
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <link rel="stylesheet" href="style.css" />
        <title>{% block title %}My amazing site{% endblock %}</title>
    </head>
    ```
    * child template—must begin with `extends`, can override contents of parent blocks:
    ```html
    {% extends "base.html" %}
    
    {% block title %}My amazing blog{% endblock %}
    ``` 
    * `{{ block.super }}` inserts the parent block's content
    * `endblock`s may optionally be named: `{% endblock title %}`
- control structure template tags: `for`/`endfor`, `if`/`elif`/`else`/`endif`
- **[template filters](https://docs.djangoproject.com/en/1.9/ref/templates/builtins/#ref-templates-builtins-filters)** modify the variable being rendered. E.g.:
    * `{{ value|lowercase }}`
    * `{{ value|length }}`
    * `{{ value|join:", " }}`
    * default in case a value is falsy: `{{ value|default:"&gt;:-(" }}`
- `{# this is a template comment #}`
- template variables are [autoescaped](https://docs.djangoproject.com/en/1.9/ref/templates/language/#automatic-html-escaping) by default, but this can be turned off for individual variables (`{{ value|safe }}`) or blocks
    * strings passed to filter literals are not autoescaped
- [more on templates](https://docs.djangoproject.com/en/1.9/topics/templates/)

## static files

Under an app directory, `static/<appname>/static` is the default home for the app's static files—images, JS, CSS, etc.

Templates must first call `{% load staticfiles %}` and will then be able to construct static file paths, e.g.: `{% static 'polls/style.css' %}`
* As with the `url` tag, the `static` tag supports an `as` variable: `{% static 'polls/style.css' as stylesheet %}`

[more on static files](https://docs.djangoproject.com/en/1.9/intro/tutorial06/)

## forms

example from [[1]](https://docs.djangoproject.com/en/1.9/intro/tutorial04/)

```html
<h1>{{ question.question_text }}</h1>

{% if error_message %}<p><strong>{{ error_message }}</strong></p>{% endif %}

<form action="{% url 'polls:vote' question.id %}" method="post">
{% csrf_token %}
{% for choice in question.choice_set.all %}
    <input type="radio" name="choice" id="choice{{ forloop.counter }}" value="{{ choice.id }}" />
    <label for="choice{{ forloop.counter }}">{{ choice.choice_text }}</label><br />
{% endfor %}
<input type="submit" value="Vote" />
</form>
```

Processing this form, of course, requires a view. We add to views.py:

```py
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

[more on forms](https://docs.djangoproject.com/en/1.9/topics/forms/)

## Avoiding race conditions

Code like

```py
o = # get a record from the database
o.myattribute += val
o.save()
```

(for example, in the `vote` view above) risks triggering a race condition if the database changes between the time the field value is looked up, and the time the new value is saved.

There is a [solution](https://docs.djangoproject.com/en/1.9/ref/models/expressions/#avoiding-race-conditions-using-f) in the form of delayed execution: `F(fieldname)` can be used as shorthand for SQL to lookup the value of a field, to be used in assigning a derived value to the same field or another field.

## Generic Views

Views can be written as classes, and Django provides base classes for common view patterns. Of interest here are the ones that involve rendering a template—[`TemplateView`](https://docs.djangoproject.com/en/1.9/ref/class-based-views/base/) and its subclasses. These render a template with some context, a single object or set of objects from a specified model.

When providing the view to the `url()` function in urls.py, the class method `as_view()` must be called to create an instance of the view.

### [`DetailView`](https://docs.djangoproject.com/en/1.9/ref/class-based-views/generic-display/#detailview): show information about a single object.

Look up an object (of a specified model) by its primary key (`pk`) and render it in `template_name` (defaults to `<appname>/<modelname>_detail.html`).

```py
class ResultsView(generic.DetailView):
    model = Question
    template_name = 'polls/results.html' # default: 'polls/question_detail.html'
    # default: context_object = 'question'
```

Corresponding urls.py pattern:

```py
url(r'^(?P<pk>[0-9]+)/results/$', views.ResultsView.as_view(), name='results'),
```

### [`ListView`](https://docs.djangoproject.com/en/1.9/ref/class-based-views/generic-display/#listview): show information about multiple objects.

Look up a set of objects (of a specified model) and render them in `template_name` (defaults to `<appname>/<modelname>_list.html`).
  - `get_queryset()` can be overloaded to customize the selection and ordering of objects
  - Other methods defined by the `MultipleObjectMixin` are also, presumably, overloadable. E.g., `get_ordering()` and methods for pagination.
  
```py
class IndexView(generic.ListView):
    template_name = 'polls/index.html' # default: 'polls/question_list.html'
    context_object_name = 'latest_question_list' # default: question_list
    
    def get_queryset(self):
        """Return the last five published questions."""
        return Question.objects.order_by('-pub_date')[:5]
```

### Other generic views 

are available for [redirects](https://docs.djangoproject.com/en/1.9/ref/class-based-views/base/#redirectview), [forms/editing](https://docs.djangoproject.com/en/1.9/ref/class-based-views/generic-editing/), and [date-based data](https://docs.djangoproject.com/en/1.9/ref/class-based-views/generic-date-based/) such as archives.

## Other topics

* [intro to testing](https://docs.djangoproject.com/en/1.9/intro/tutorial05/), [more on testing](https://docs.djangoproject.com/en/1.9/topics/testing/)
* [pagination](https://docs.djangoproject.com/en/1.9/topics/pagination/)
* [user authentication](https://docs.djangoproject.com/en/1.9/topics/auth/)
* [sending email](https://docs.djangoproject.com/en/1.9/topics/email/)

## Extensions of interest

* [Mezzanine](http://mezzanine.jupo.org/) CMS/blogging platform
  - [mezzanine-pagedown](https://pypi.python.org/pypi/mezzanine-pagedown) for Markdown support
