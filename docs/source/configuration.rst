.. _config:

Configuration
=============

sqlfluff accepts configuration either through the command line or
through configuration files. There is *rough* parity between the
two approaches with the exception that *templating* configuration
must be done via a file, because it otherwise gets slightly complicated.

For details of what's available on the command line check out
the :ref:`cliref`.

For file based configuration *sqlfluff* will look for the following
files in order. Later files will (if found) will be used to overwrite
any vales read from earlier files.

- *setup.cfg*
- *tox.ini*
- *pep8.ini*
- *.sqlfluff*

Within these files, they will be read like an `cfg file`_, and *sqlfluff*
will look for sections which start with *sqlfluff*, and where subsections
are delimited by a semicolon. For example the *jinjacontext* section will
be indicated in the section started with *[sqlfluff:jinjacontext]*.

.. _`cfg file`: https://docs.python.org/3/library/configparser.html

Nesting
-------

**Sqlfluff** uses **nesting** in it's configuration files, with files
closer *overriding* (or *patching*, if you will) values from other files.
That means you'll end up with a final config which will be a patchwork
of all the values from the config files loaded up to that path.
You don't **need** any config files to be present to make *sqlfluff*
work. If you do want to override any values though sqlfluff will use
files in the following locations in order, with values from later
steps overriding those from earlier:

0. *[...and this one doesn't really count]* There's a default config as
   part of the sqlfluff package. You can find this below, in the
   :ref:`defaultconfig` section.
1. It will look in the user's home directory (~), for any of the
   filenames above in the main :ref:`config` section. If
   multiple are present, they will *patch*/*override* eachother
   in the order above.
2. It will look for the same files in the current working directory.
3. *[if parsing a file in a subdirectory of the current working directory]*
   It will look for the same files in every subdirectory between the
   current working dir and the file directory.
4. It will look for the same files in the directory containing the file
   being linted.

This whole structure leads to efficient configuration, in particular
in projects which utilise a lot of complicated templating.

.. _templateconfig:

Templating Configuration
------------------------

When thinking about templating there are two different kinds of things
that a user might want to fill into a templated file, *variables* and
*functions/macros*. Currently *functions* aren't implemented in any
of the templaters.

Variable Templating
^^^^^^^^^^^^^^^^^^^

Variables are available in the *jinja* and *python* templaters. By default
the templating engine will expect variables for templating to be available
in the config, and the templater will be look in the section corresponding
to the context for that templater. By convention, the config for the *jinja*
templater is found in the *sqlfluff:templater:jinja:context* section and the
config for the *python* templater is found in the
*sqlfluff:templater:python:context* section.

For example, if passed the following *.sql* file:

.. code-block:: jinja

    SELECT {{ num_things }} FROM {{ tbl_name }} WHERE id > 10 LIMIT 5

...and the following configuration in *.sqlfluff* in the same directory:

.. code-block:: cfg

    [sqlfluff:templater:jinja:context]
    num_things=456
    tbl_name=my_table

...then before parsing, the sql will be transformed to:

.. code-block:: sql

    SELECT 456 FROM my_table WHERE id > 10 LIMIT 5

.. note::

    If there are variables in the template which cannot be found in
    the current configuration context, then this will raise a `SQLTemplatingError`
    and this will appear as a violation without a line number, quoting
    the name of the variable that couldn't be found.

Macro Templating
^^^^^^^^^^^^^^^^

Macros (which also look and feel like *functions* are available only in the
*jinja* templater. Similar to `Variable Templating`_, these are specified in
config files, what's different in this case is how they are named. Similar to
the *context* section above, macros are configured seperately in the *macros*
section of the config. Consider the following example.

If passed the following *.sql* file:

.. code-block:: jinja

    SELECT {{ my_macro(6) }} FROM some_table

...and the following configuration in *.sqlfluff* in the same directory (note
the tight control of whitespace):

.. code-block:: cfg

    [sqlfluff:templater:jinja:macros]
    a_macro_def = {% macro my_macro(something) %}{{something}} + {{something * 2}}{% endmacro %}

...then before parsing, the sql will be transformed to:

.. code-block:: sql

    SELECT 6 + 12 FROM some_table

Note that in the code block above, the variable name in the config is
*a_macro_def*, and this isn't apparently otherwise used anywhere else.
Broadly this is accurate, however within the configuration loader this will
still be used to overwrite previous *values* in other config files. As such
this introduces the idea of config *blocks* which could be selectively
overwritten by other configuration files downsteam as required. Some such
blocks are provided by default as `Builtin Macro Blocks`_ to assist with
common use cases.

.. note::

    Throughout the templating process **whitespace** will still be treated
    rigourously, and this includes **newlines**. In particular you may choose
    to provide your *dummy* macros in your configuration with different to
    the actual macros you may be using in production.

    **REMEMBER:** The purpose of providing the option of macros is to *enable*
    the parsing of templated sql without it being a blocker. It shouldn't
    be a requirement that the *templating* is accurate - only so far as that
    is required to enable the *parsing* and *linting* to be helpful.

Builtin Macro Blocks
^^^^^^^^^^^^^^^^^^^^

One of the main use cases which inspired *sqlfluff* as a a project was `dbt`_.
It uses jinja templating extensively and leads to some users maintaining large
repositories of sql files which could potentially benefit from some linting.

*Sqlfluff* anticipates this use case and provides some built in macro blocks
in the `Default Configuration`_ which assist in getting started with `dbt`_
projects. In particular it provides mock objects for:

* *ref*: The mock version of this provided simply returns the model reference
  as the name of the table. In most cases this is sufficient.
* *config*: A regularly used macro in `dbt`_ to set configuration values. For
  linting purposes, this makes no difference and so the provided macro simply
  returns nothing.

.. note::

    If there are other builtin macros which would make your life easier,
    consider submitting the idea (or even better a pull request) on `github`_.

.. _`dbt`: https://www.getdbt.com/
.. _`github`: https://www.github.com/alanmcruickshank/sqlfluff

.. _defaultconfig:

Default Configuration
---------------------

The default configuration is as follows, note the `Builtin Macro Blocks`_ in
section *[sqlfulff:templater:jinja:macros]* as referred to above.

.. literalinclude:: ../../src/sqlfluff/default_config.cfg
   :language: cfg
   :linenos:
