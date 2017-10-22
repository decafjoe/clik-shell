
============
 clik-shell
============

clik-shell is a tiny glue library between clik_ and cmd_::

  from clik import app
  from clik_shell import DefaultShell


  @app
  def myapp():
      yield


  # ... subcommands for myapp ...


  @myapp
  def shell():
      yield
      DefaultShell(myapp).cmdloop()

See :ref:`the quickstart <quickstart>` for more documentation on what
clik-shell can do.

.. _clik: https://clik.readthedocs.io
.. _cmd: https://docs.python.org/3/library/cmd.html

.. toctree::
   :maxdepth: 2

   quickstart
   api
   internals
   changelog
