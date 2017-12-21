PonyUp!
=======

Version ``0.1.1``
                 

Migrations for ponyorm

Very simple migrations.
-----------------------

Install
~~~~~~~

This still a first beta version, but you can test it already with the
following shell command:

.. code:: sh

    $ pip install pony_up

Getting started
~~~~~~~~~~~~~~~

.. code:: python

    from pony_up import migrate

    # to be able to bind the database with your information,
    # just create a function for it:
    def bind_func(db):
        db.bind('database type', host='localhost', user='root', passwd='1234secure', db='test1')
        # https://docs.ponyorm.com/api_reference.html#Database.bind
        db.generate_mapping(create_tables=True)
        # https://docs.ponyorm.com/api_reference.html#Database.generate_mapping
    # end def


    db = migrate(bind_func, folder_path="examples/migrations, python_import="examples.migrations")

The updates are applied as soon as ``migrate`` is called. It will return
``db``, being the latest schema.

Your File schema
~~~~~~~~~~~~~~~~

-  ``migrations/``: The migrations are in here

   -  ``v{number}/``: for example “v0”

      -  ``__init__.py``: needs to import the model and also migrate if
         present.
      -  ``model.py``: model of version ``{number}``
      -  ``migrate.py``: the script which updates from ``{number}`` to
         ``{number+1}``

   -  ``v{number}.py``:
      A file is possible too, if it has the attribute ``model`` with a
      function ``register_database`` (calling
      ``model.register_database(db)``)
      and optionally a ``migrate`` attribute with function ``do_update``
      (will call ``migrate.do_update(db)``)

The required functions
~~~~~~~~~~~~~~~~~~~~~~

``model.py``
            

.. code:: python

    def register_database(db):

..

    In this function should be your ``orm.Entity`` subclasses.

Arguments: - ``db`` - The database to register entities to.

``migrate.py``
              

.. code:: python

    def do_update(db, old_db=None):

..

    Here you write code which changes stuff in the database

| Arguments: - ``db`` - The latest schema. - ``old_db`` - This can have
  3 different types:
| - ``pony.orm.Database`` A database schema of the previous migration
  step (Would be **v0** if we are at **v1**. See *Fig 1*),
| - ``True`` if the provided ``db`` database is old, and this version
  didn’t introduced a new schema. (See **v2** in *Fig 1*)
| - ``None`` if there was no previous step (The first migration, e.g.
  **v0**)

Info graphic
~~~~~~~~~~~~
.. image:: https://cloud.githubusercontent.com/assets/2737108/25397889/3a75eca2-29ea-11e7-9527-0bb3cc1412ef.png
    :alt: *Fig 1. Migrations*

FAQ
~~~

How to use
''''''''''

    See above, or have a look at the example.

Can I contribute?
'''''''''''''''''

    | Please do!
    | Report issues, suggest features, or even submit code!

I don’t like using ``db.{EntityName}``.
'''''''''''''''''''''''''''''''''''''''

I have used the file ``database.py`` before, to include all my objects,
and still like to use the existing import statements. I imported:

.. code:: python

    from database import {EntityName}

or even import all the database entities with the wildcard import

.. code:: python

    from database import *

..

    | You should move the entity definitions in ``database.py`` into a
      migrations step (``v0.model`` perhaps), and replace the file
      content with ``db = migrate(...)``, like seen above.
    | Now you can add the following lines after said
      ``db = migrate(...)`` part:

    .. code:: python

        # register the tables to this module
        __all__ = ["db"]
        for t_name, t_clazz in db.entities.items():
            globals()[t_name] = t_clazz
            __all__.append(t_name)
        # end for

My application with the migration will run multible times at the same time
''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''

    | You need to deploy some sort of locking, because else two clients
      trying to modify the same tables would end in a disaster.
    |     If you use postgres, you can use `Advisory Locks <https://www.postgresql.org/docs/9.1/static/explicit-locking.html#ADVISORY-LOCKS>`_. (Also see this `blog post with examples <https://hashrocket.com/blog/posts/advisory-locks-in-postgres>`_).

    | Request a lock before the ``db = migrate(...)``, and release it
      afterwards:

    .. code:: python

        import psycopg2
        con = psycopg2.connect(host=POSTGRES_HOST, user=POSTGRES_USER, password=POSTGRES_PASSWORD, database=POSTGRES_DB)
        cur = con.cursor()
        # requesting database update lock
        cur.execute("SELECT pg_advisory_lock(85,80);")  # update lock (ascii: 85,80 = UP)


        # run the migration
        db = migrate(...)


        # releasing lock after database update
        cur.execute("SELECT pg_advisory_unlock(85,80);")  # update lock (ascii: 85,80 = UP)
        res = cur.fetchone()
        if not isinstance(res[0], bool) or not res[0]:
            # True = success
            # Fail => false or no bool
            raise ValueError("Could not release update lock, lock was not held (Advisory Lock 85,80)")
        # end if

I like the script above, but it should just terminate instead of waiting
''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''

    Replace the ``cur.execute("SELECT pg_advisory_lock(85,80);")`` part
    above with:

    .. code:: python

        # requesting database update lock
        cur.execute("SELECT pg_try_advisory_lock(85,80);")  # update lock (ascii: 85,80 = UP)
        res = cur.fetchone()
        if not isinstance(res[0], bool) or not res[0]:
            # True = success
            # Fail => false or no bool
            raise ValueError("Currently already upgrading. (Advisory Lock 85,80)")
        # end if

    | With that your script will raise an exception (and probably
      terminate) if the database is already being upgraded somewhere
      else.
    | Note: in a webserver (flask, django, …) environment this is
      probably not wanted. Like, a Nginx server would keep running, and
      uWSGI would spam the log with
      ``no python application found, check your startup logs for errors``.

Where does the name come from?
''''''''''''''''''''''''''''''

    | Because of the library ``Pony ORM``, the verb ``to pony up`` and
      this tool doing ``updates``!
    | Got it? Yeah, what a sick joke! Tell your Grandma, too!

Who is best pony?
'''''''''''''''''

    Definitely **Littlepip**! (see `Fallout: Equestria <http://falloutequestria.wikia.com/wiki/Fallout:_Equestria>`_)


Why is this FAQ getting stupid now?
'''''''''''''''''''''''''''''''''''

    lel.
