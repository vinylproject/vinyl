The vinyl project (aka async django)
---------------
*currently at alfa stage*

The vinyl project is an initiative (unofficial) to continue adding the 
async support to django. The main task is porting the **django orm** - the main 
remaining obstacle in the way.

Goals:
- async-only (native asynchrony)
- be compatible with django models, not break existing code
- be close to classic django in terms of API

**A model manager**

The main entry point to the magic is the model manager, which you need to 
add alongside 
the relular one (`objects`, likely).

```python
class M(models.Model):
  ...
  vinyl = VinylManager()
```

The default database for vinyl is `'vinyl_default'`. `vinyl` provides the 
respective backends. So, here is what you should have in the `settings.py`:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'PASSWORD': 'postgres',
        'USER': 'postgres',
        'NAME': 'mydb',
    },
    'vinyl_default': {
        'ENGINE': 'vinyl.postgresql_backend',
        'PASSWORD': 'postgres',
        'USER': 'postgres',
        'NAME': 'mydb',
    },
}
```

Now you can try the magic:

```python
await M.vinyl.all()
ob = await M.vinyl.get()
await ob.related_set.all()
ob.related_obj = related_obj
await ob.save()
```

`M.objects.all()` will keep working, like the rest of django.

**Model instances (aka the objects). Insert and update separated**

Like django itself, vinyl builds a lot of its API around the model 
instances (CRUD operations, related attributes, etc). But since the API is 
different from django (namely, it is async), vinyl uses a different class for 
the model:

```
In [1]: (await M.vinyl.first()).__class__
Out[1]: vinyl.model.M
```

You can access the class of the vinyl model through `M.vinyl.model`, but 
generally you don't have to. With vinyl, you do not directly instantiate the 
models. When making a query, you get the instantiated objects already, and 
for inserts you should use

```python
obj = await M.vinyl.create(**kwargs)
```

In other words, update and insert are separated. `Model.save` can only be 
used for an update. Not only a more explicit API is better in my opinion, 
but, as a side effect from this, the model saving code got much 
cleaner.

**Lazy attributes, prefetch_related and the like**

In django, the related attributes are lazy. If they have been prefetched 
somehow, you get the cached values, otherwise the query is made. In vinyl, 
lazy attributes are gone. The old-style access (with `await`, of course) 
will always lead to a query:

```python
await obj.related_obj  # hits the database
                       # will not be cached automatically
```
To get the prefetched values one should use 
dictionary access:

```python
obj['related_obj']
obj['related_set']
```

Again, making the API more explicit is only a plus, in my opinion.

**What is supported**

Currently, almost all of django API is supported one way or another, 
with a few exceptions:

- no signals
- no chunked fetching
- autocommit is turned off

I am thinking about making the rules of model inheritance more strict too, so as
to only support the case where all models in the inheritance chain share 
the same primary key. This would simplify the logic of CRUD operations (and 
their bulk variants).

Of the databases, **PostgreSql** amd **MySql/MariaDb** are supported. I used 
the psycopg3 driver instead of the asyncpg, but I think, I will make some 
benchmarks in the nearest future. 

As in django, the database support is provided by database backends, so can 
be contributed easily.

Also, since vinyl uses django models, you have the django admin available as 
well as the migrations.

**Plans for release**

- moving to beta, adding tests
- psycopg or asyncpg?
- Not to increase the scope

**Bonus: how was this made possible?**

First of all, django turned out to be pretty extensible itself. Here I want to 
mention database backends, compilers, ability to use a 
non-default database.

Second, there are things specific to an orm in general and to django in 
particular, that make porting easier. Here I will draw a simplified picture. 

For the read-access, we usually use querysets. These are lazy, so are not 
evaluated until you decide so. One would naturally want to do that in the 
`__await__` 
method. It turned out pretty easy to pre-evaluate a queryset, and then launch 
the usual procedure, reusing the compiler from the pre-evaluation step.

Next, there is write access: CRUD operations and the like. These generally 
execute some SQL, not being interested in the result. I mean, we need to know 
whether it was a success or not, but do not need to fetch the results, by 
doing, for example, `cursor.fetchall()`. These operations can simply store all 
the queries they need to make and then to execute them in deferred fashion.

As a result, we get an async-native implementation, which is a much better 
solution than other hacky tricks. An example for the latter can be 
sqlalchemy, which resorted to greenlets for the similar purpose. 
