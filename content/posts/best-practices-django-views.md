---
title: "Best Practices for Django REST Framework Views"
date: 2022-09-28T21:24:58+01:00
draft: false
featured_image: '/images/drf-logo.png'
---

Django REST Framework has two methods for writing APIs - function based views (similar for people coming from other frameworks like Flask or FastAPI), or class-based views (which is more like the patterns in the ASP.NET and Spring frameworks). Class-based views seem to have won out within the Django community, and so here we will focus on those, but even after making this choice, developers still need to decide which type of class-based view is appropriate. 

# APIView and GenericAPIView

Reading [the tutorial](https://www.django-rest-framework.org/tutorial/3-class-based-views/), the first thing a new developer in Django will encounter is the 'APIView' type of view. The developer can implement a method for each HTTP verb that they are interested in handling. The downside of this is that the developer needs to potentially implement *multiple* APIView classes in order to model a resource. For example, for a RESTful API representing a blog, you would generally want to be able to retrieve both a list of blog posts (usually with some key details like the ID, title and perhaps some other metadata like the date) at a location like `/api/blog`, as well as a specific blog post at `/api/blog/<id>`. To do this, you'd need to write two separate classes, each with their own GET action:
```python3
# views.py

class BlogListView(APIView):
    def get(self, *args, **kwargs):
        serializer = BlogListSerializer(Blog.objects.all(), many=True)
        return Response(serializer.data)

class BlogDetailView(APIView):
    def get(self, id, *args, **kwargs):
        instance = get_object_or_404(Blog, id)
        serializer = BlogDetailSerializer(instance)
        return Response(serializer.data)
```

In practice, the boilerplate of these methods is often unnecessary if the logic is simple, and if the data for the respose can be constructed from a single Django QuerySet (potentially with annotations, filtering, etc). One of the beauties of Django (and similar frameworks like Ruby on Rails) is that you can get a lot of the CRUD methods implemented almost for free, and only implement more complicated logic only when you *need* to do so. For example, you could get all of the CRUD endpoints set up and working simply by instantiating the following classes:

```python3
# views.py
from rest_framework.views import ListCreateAPIView, RetrieveUpdateDestroyAPIView

class BlogListCreateView(ListCreateAPIView):
    queryset = Blog.objects.all()
    serializer = BlogListSerializer

class BlogRetrieveUpdateDestroyView(RetrieveUpdateDestroyAPIView)
    queryset = Blog.objects.all()
    serializer = BlogDetailSerializer
```

A few things to note here:
* We've used specific DRF "Concrete view classes" here. There's a whole list of these in the [Django documentation](https://www.django-rest-framework.org/api-guide/generic-views/#concrete-view-classes). You should pick with care which actions you actually want to provide.
* We add attributes to the class specifying the serializer and the queryset. These are not the only attributes it is possible to specify, but they are the most important and regularly used ones. This is because the concrete view classes inherit from `GenericAPIView` rather than APIView. This is a class that is designed to work directly with Django models, whereas using APIView directly is more appropriate when you want to write an endpoint that has no reference to the Django ORM at all. This makes a difference when generating an OpenAPI schema, as when serializers are specified on a class inheriting from GenericAPIView, the schema will be automatically generated based on the specified serializers.
* We've used a different serializer for creating/listing and retrieving/updating/deleting. This is a really common and important pattern. Generally, the information you display in a list should be a reduced subset of what is fetched when retrieving a single instance of a resource. Therefore, the serializer that you use to turn a Django model instance should almost always be different.

## ViewSets

ViewSets specifically aim to solve the problem of having to write multiple APIViews. Instead of writing a 'get' method twice, you write a 'retrieve' method to get a specific instance, and a 'list' method to get all instances. Similarly, rather than writing a 'post' method, you would write a 'create' method, and for 'put' a 'update' and 'patch' a 'partial_update' method. These are all magically translated to the appropriate endpoint location (e.g. `/api/blog` for list, `/api/blog/1` for retrieve) by a ['router'](https://www.django-rest-framework.org/api-guide/routers/). For example, you could write:

```python3
from rest_framework.viewsets import ViewSet

class BlogViewSet(ViewSet):
    def list(self, request):
        queryset = Blog.objects.all()
        serializer_class = BlogListSerializer(queryset, many=True)
        return Response(serializer.data)

    def retrieve(self, request, pk=None):
        instance = get_object_or_404(Blog, pk)
        serializer_class = BlogDetailSerializer(instance)
        return Response(serializer.data)
```

As before, we don't generally need to implement these boilerplate methods; we can inherit from mixin classes that provide the methods for us:

```python3
from rest_framework.viewsets import GenericViewSet
from rest_framework.mixins import (
    CreateModelMixin,
    ListModelMixin,
    RetrieveModelMixin,
    UpdateModelMixin,
    DestroyModelMixin
)

class BlogViewSet(CreateModelMixin,
                  ListModelMixin,
                  RetrieveModelMixin,
                  UpdateModelMixin,
                  DestroyModelMixin,
                  GenericViewSet):
    queryset = Blog.objects.all
    serializer_class = BlogSerializer
```

A few more things to note here, however:
* Like with APIView we end up with a class that inherits from GenericViewSet which is designed to work with the ORM and serializers rather than ViewSet when using Mixins. 
* Rather than inheriting each mixin individually, you can inherit from 'collections' of these - for e.g. [ModelViewSet](https://www.django-rest-framework.org/api-guide/viewsets/#modelviewset) or [ReadOnlyModelViewSet](https://www.django-rest-framework.org/api-guide/viewsets/#readonlymodelviewset). In practice, I prefer to specifically choose the subset mixins I actually need.
* This is very much the way that the documentation pushes you to write a viewset. However, we've only been able to specify a *single* serializer, despite the fact we might need to use multiple ones.
* The Mixins are not actually unique to the ViewSet - they can be used in the ordinary APIView type methods, but you would for e.g. need to call the 'list' or 'retrieve' method directly from your 'get' method.

# Customising Behaviour

As we noted in the previous example, there are often times when you need to customise the behaviour - in that example, it was in having different serializers for different actions, but we can foresee examples like where an authenticated user might get a different set of results to an unauthenticated user. In these cases, there are effectively two options:
* We go back to writing our own boilerplate methods, and give up one the ones provided by DRF.
* We don't specify properties like 'queryset' and 'serializer' when we inherit from APIView or ViewSets. We instead provide a method `get_<attribute_name>` that provides that attribute dynamically, for e.g. based on the user or other information we can garner from the request.

My experience has been that the best option here *really* depends on the case at hand. For serializers for example, I've found this pattern to be easy to use and understand for people new to the code base:

```python3
class BlogViewSet(ViewSet):
    ... 
    def get_serializer_class(self, request):
        # Specify any serializers that are 'custom' for end points
        options = {
            'retrieve': BlogRetrieveSerializer,
            'list': BlogListSerializer
        }
        # Return the appropriate serializer for the method if it exists, otherwise return the default:
        return options.get(request.method, BlogCreateUpdateSerializer)
```

For querysets, I've generally found it easier to go back to writing a boilerplate method, or even splitting these out into a seperate services methods in order to allow better code reuse (particularly where a queryset might be used both for an API and Django serving pages) and testing:

```python3
# services.py

def get_complicated_blog_objects():
    return Blog.objects.prefetch_related(
        "author__firstname",
        "author__lastname"
    ).filter(
        date__gte=datetime.date.today()-datetime.timedelta(days=30)
    ).annotate(
        author_name=F("author__firstname")+F("author__lastname")
    )

# views.py
class BlogViewSet(ViewSet):
    queryset = get_complicated_blog_objects()
```

There are a lot of other methods like the `get_queryset` and `get_serializer` that can be found on GenericAPIView and GenericViewSet and their derived classes, and the best way I've found to work through the multiple inheritance heirarchy is with the great little [Classy DRF](https://www.cdrf.co/) website which shows the ancestors and all attributes and methods for a given class.
 
## What circumstances should I use an APIView, a GenericAPIView, a ViewSet or a GenericViewSet?

Some rules of thumb I use day-to-day:
* GenericViewSet and GenericAPIView (and classes that make use of them) should only be used where there is a correspondence directly to something directly representable as a single QuerySet in the Django ORM. If you are for e.g. going off to another service to get some data which you return to the client, or are combining multiple querysets across objects to construct a response, then APIView or ViewSet are the better choice.
* Does my API correspond 1-2-1 with a Django ORM model and actions on it? If so, I generally use a ViewSet to avoid having to write two classes. A case where I might use an APIView instead might be where for e.g. I calculate some aggregate statistics (counts of objects, most recent object, etc.) where I only need a simple `get` method. I think this is easier to understand, because for such actions, generally 'list' and 'retrieve' as language around this doesn't really make sense.
