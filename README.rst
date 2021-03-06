===============================
Etcaetera
===============================

.. image:: https://badge.fury.io/py/etcaetera.png
    :target: http://badge.fury.io/py/etcaetera
    
.. image:: https://travis-ci.org/oleiade/etcaetera.png?branch=master
        :target: https://travis-ci.org/oleiade/etcaetera

.. image:: https://pypip.in/d/etcaetera/badge.png
        :target: https://crate.io/packages/etcaetera?version=latest

What?
=====

Etcaetera is the simplest way for you to load your application configuration from multiple sources.

It exposes a single **Config** object to which you add prioritized sources adapters (env, files, cmdline, modules...).

Once you call ``load`` method on it: your settings are loaded from your adapters in the right order, all your configuration is stored in the **Config** object.

You're **done**.



Why?
====

Managing a large application configuration sources can be a pain in the neck.

Command line, files, system environment, modules, the settings you seek come from a lot of mixed sources.

They all have a different accessing mode and merging them can sometimes seem impossible !

Etcaetera provides you with a simple and unified way to handle all the complexity in a single place.

Installation
============

With pip
--------

.. code-block:: bash

    $ pip install etcaetera

With setuptools
---------------

.. code-block:: bash

    $ git clone git@github.com:oleiade/etcaetera
    $ cd etcaetera
    $ python setup.py install


Usage
=====

Dive
----

A real world example is worth a thousand words

.. code-block:: python

    >>> from etcaetera.config import Config
    >>> from etcaetera.adapters import Defaults, Overrides, Env, File

    # Let's create a new configuration object
    >>> config = Config()

    # And create a bunch of adapters
    >>> env_adapter = Env(keys=["MY_FIRST_SETTING", "MY_SECOND_SETTING"])
    >>> python_file_adapter = File('/etc/my/python/settings.py')
    >>> json_file_adapter = File('/etc/my_json_settings.json')
    >>> module_adapter = Module(os)
    >>> overrides = Overrides({"MY_FIRST_SETTING": "my forced value"})

    # Let's register them
    >>> config.register([env_adapter, python_file_adapter, json_file_adapter, module_adapter, overrides])

    # Load configuration
    >>> config.load()


    # And that's it
    >>> print config
    {
        "MY_FIRST_SETTING": "my forced value",
        "MY_SECOND_SETTING": "my second value",
        "FIRST_YAML_SETTING": "first yaml setting value found in yaml settings",
        "FIRST_JSON_SETTING": "first json setting value found in json settings",
        ...
    }


Config object
-------------

The config object is the central place for your whole application settings. It loads your adapters in the order you've registered them, and updates itself using it's data.

Please note that **Defaults** adapter will always be loaded first, and **Overrides** will always be loaded last.

.. code-block:: python

    >>> from etcaetera.config import Config

    # You can provide defaults to Config at initialization, whether as a Defaults object,
    # or as a dict.
    >>> config = Config({"abc": "123"})

    >>> print config
    {
        "ABC": "123  # every Config keys will automatically be uppercased
    }

    # When you register adapters to it, they are not immediately evaluated.
    >>> config.register(Env(["USER", "PWD"])
    >>> assert "USER" not in config
    True
    >>> assert "PWD" not in config
    True
    >>> config.register(Overrides({"abc": "do re mi"})
    >>> assert config["ABC"] != "do re mi"
    True

    # Whenever you call load, adapters are evaluated and your config
    # values are updated accordingly
    >>> config.load()
    >>> print config
    {
        "ABC": "do re mi",
        "USER": "your user",
        "PWD": "/current/working/directory"
    }


Adapters
--------

Adapters are the interfaces with configuration sources. They load settings from their custom source type,
and they expose them as a normalized dict to *Config* objects.

Right now, etcaetera provides the following adapters:
    * *Defaults*: sets some default settings
    * *Overrides*: overrides the config settings values
    * *Env*: extracts configuration values from system environment
    * *File*: extracts configuration values from a file. Accepted format are: json, yaml, python module file (see *File adapter* section for more details)
    * *Module*: extracts configuration values from a python module. Like in django, only uppercased variables will be matched

In a close future, etcaetera may provide adapters for:
    * *Argv* argparse format support: would load settings from an argparser parser attributes
    * *File* ini format support: would load settings from an ini file

Defaults adapter
~~~~~~~~~~~~~~~~

Defaults adapter provides your configuration object with default values.
It will always be evaluated first when ``Config.load`` method is called.
You can whether provide defaults values to *Config* as a *Defaults* object
or as a dictionary.

.. code-block:: python

    >>> from etcaetera.adapter import Defaults

    # Defaults adapter provides default configuration settings
    >>> defaults = Defaults({"ABC": "123"})
    >>> config = Config(defaults)

    >>> print config
    {
        "ABC": "123"
    }

Overrides adapter
~~~~~~~~~~~~~~~~~

The Overrides adapter overrides *Config* object values with it's own values.
It will always be evaluated last when the ``Config.load`` method is called.

.. code-block:: python

    >>> from etcaetera.adapter import Overrides

    # The Overrides adapter helps you set overriding configuration settings.
    # When registered over a Config objects, it will always be evaluated last.
    # Use it if you wish to force some config values.
    >>> overrides_adapter = Overrides({"USER": "overrided value"})
    >>> config = Config({
        "USER": "default_value",
        "FIRST_SETTING": "first setting value"
    })

    >>> config.register(overrides_default)
    >>> config.load()

    >>> print config
    {
        "USER": "overrided user",
        "FIRST_SETTING": "first setting value"
    }



Env adapter
~~~~~~~~~~~

The Env adapter loads settings from your system environement.
It should come with a list of keys to fetch. If you don't provide the keys yourself,
the parent *Config* object will automatically provide it's own.

.. code-block:: python

    >>> from etcaetera.adapter import Env

    # You can provide keys to be fetched by the adapter at construction
    >>> env = Env(keys=["USER", "PATH"])

    # Or whenever you call load over it. They will be merged
    # with those provided at initialization.
    >>> env.load(keys=["PWD"])

    >>> print env.data
    {
        "USER": "user extracted from environment",
        "PATH": "path extracted from environment",
        "PWD": "pwd extracted from environment"
    }

File adapter
~~~~~~~~~~~~

The File adapter will load the configuration settings from a file.
Supported formats are json, yaml and python module files. Every key-value pairs
stored in the pointed file will be loaded in the *Config* object it is registered to.


Python module files
```````````````````

The Python module files should be in the same format as the Django settings files. Only uppercased variables
will be loaded. Any python data structures can be used.

*Here's an example*

*Given the following settings.py file*

.. code-block:: bash

    $ cat /my/settings.py
    FIRST_SETTING = 123
    SECOND_SETTING = "this is the second value"
    THIRD_SETTING = {"easy as": "do re mi"}
    ignored_value = "this will be ignore"

*File adapter output will look like this*:

.. code-block:: python

    >>> from etcaetera.adapter import File

    >>> file = File('/my/settings.py')
    >>> file.load()

    >>> print file.data
    {
        "FIRST_SETTING": 123,
        "SECOND_SETTING": "this is the second value",
        "THIRD_SETTING": {"easy as": "do re mi"}
    }

Serialized files (aka json and yaml)
````````````````````````````````````

*Given the following json file content*:

.. code-block:: bash

    $ cat /my/json/file.json
    {
        "FIRST_SETTING": "first json file extracted setting",
        "SECOND_SETTING": "second json file extracted setting"
    }

*The File adapter output will look like this*:

.. code-block:: python

    >>> from etcaetera.adapter import File

    # The File adapter awaits on a file path at construction.
    # All you have to do then, is to let the magic happen
    >>> file = File('/my/json/file.json')
    >>> file.load()

    >>> print file.data
    {
        "FIRST_SETTING": "first json file extracted setting",
        "SECOND_SETTING": "second json file extracted setting"
    }


Module adapter
~~~~~~~~~~~~~~

The Module adapter will load settings from a python module. It emulates the django
settings module loading behavior, so that every uppercased locals of the module is matched.

**Given a mymodule.settings module looking this**:

.. code-block:: python

    MY_FIRST_SETTING = 123
    MY_SECOND_SETTING = "abc"

**Loaded module data will look like this**:

.. code-block:: python

    >>> from etcaetera.adapter import Module

    # It will extract all of the module's uppercased local variables
    >>> module = Module(mymodule.settings)
    >>> module.load()

    >>> print module.data
    {
        MY_FIRST_SETTING = 123
        MY_SECOND_SETTING = "abc"
    }


Contribute
==========

Please read the `Contributing <https://github.com/oleiade/etcaetera/blob/develop/CONTRIBUTING.rst>`_ instructions

If you are lazy, here's a summary:

1. Found a bug? Want to add a feature? Check for open issues or open a fresh one to start a discussion about it.
2. Fork the repository, and start making your changes
3. Write some tests showing you fixed the current bug or your feature works as expected
4. Fasten your seatbelt, and send a pull request to the *develop* branch.


License
=======
The MIT License (MIT)

Copyright (c) 2014 Théo Crevon

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
the Software, and to permit persons to whom the Software is furnished to do so,
subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

