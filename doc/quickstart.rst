
.. _quickstart:

============
 Quickstart
============

clik-shell makes it easy to add an interactive command shell to your
clik_ application.

.. _clik: https:://clik.readthedocs.io


Example Program
===============

.. highlight:: python

Here's the program we'll be working with::

  from clik import app

  @app
  def myapp():
      """Example application for clik-shell."""
      yield
      print('myapp')

  @myapp
  def foo():
      """Print foo."""
      yield
      print('foo')

  @myapp
  def bar():
      """Print bar."""
      yield
      print('bar')

  @myapp
  def baz():
      """A subcommand with subcommands."""
      yield
      print('baz')

  @baz
  def spam():
      """Print spam."""
      yield
      print('spam')

  @baz
  def ham():
      """Print ham."""
      yield
      print('ham')

  @baz
  def eggs():
      """Print eggs."""
      yield
      print('eggs')

  if __name__ == '__main__':
      myapp.main()


Add Shell Subcommand
====================

Add a new subcommand that makes use of
:class:`clik_shell.DefaultShell`::

  from clik_shell import DefaultShell

  @myapp
  def shell():
      """Interactive command shell for my application."""
      yield
      DefaultShell(myapp).cmdloop()

.. highlight:: bash

That's it! The example application now has an interactive command
shell::

  $ ./example.py shell
  myapp
  myapp> help

  Documented commands (type help <topic>):
  ========================================
  EOF  bar  baz  exit  foo  help  quit  shell

  myapp> help foo
  usage: foo [-h]

  Print foo.

  optional arguments:
    -h, --help  show this help message and exit

  myapp> help baz
  usage: baz [-h] {spam,ham,eggs} ...

  A subcommand with subcommands.

  optional arguments:
    -h, --help       show this help message and exit

  subcommands:
    {spam,ham,eggs}
      spam           Print spam.
      ham            Print ham.
      eggs           Print eggs.

  myapp> foo
  foo
  myapp> baz
  usage: baz [-h] {spam,ham,eggs} ...
  baz: error: the following arguments are required: {spam,ham,eggs}

  myapp> qux
  error: unregonized command: qux (enter ? for help)

  myapp> baz spam
  baz
  spam
  myapp> exit

  $


Intended Usage
==============

.. highlight:: python

In practice, the base shell is designed to be subclassed::

  class Shell(DefaultShell):
      def __init__(self):
          super(Shell, self).__init__(myapp)

  @myapp
  def shell():
      """Interactive command shell for my application."""
      yield
      Shell().cmdloop()

:class:`DefaultShell <clik_shell.DefaultShell>` is a subclass of
:class:`Cmd <cmd.Cmd>`, so subclasses of :class:`DefaultShell
<clik_shell.DefaultShell>` can make use of everything in :class:`Cmd
<cmd.Cmd>`. This is useful for things like customizing the prompt and
adding introductory text::

  class Shell(DefaultShell):
      intro = 'Welcome to the myapp shell. Enter ? for a list of commands.\n\n'
      prompt = '(myapp)% '

.. highlight:: bash

With those updates::

  $ ./example.py shell
  myapp
  Welcome to the myapp shell. Enter ? for a list of commands.


  (myapp)%


Excluding Commands from the Shell
=================================

.. highlight:: bash

As implemented, the ``shell`` command is available from within the
shell::

  $ ./example.py shell
  myapp
  myapp> ?

  Documented commands (type help <topic>):
  ========================================
  EOF  bar  baz  exit  foo  help  quit  shell

  myapp> shell
  myapp> exit

  myapp> exit

  $

.. highlight:: python

This works, but isn't the desired behavior. There's no reason for
users to start a "subshell." For this case,
:func:`clik_shell.exclude_from_shell` is available::

  from clik_shell import DefaultShell, exclude_from_shell

  @exclude_from_shell
  @myapp
  def shell():
      """Interactive command shell for my application."""
      yield
      Shell().cmdloop()

.. highlight:: bash

Now users cannot call ``shell`` from within the shell::

  $ ./example.py shell
  myapp
  myapp> ?

  Documented commands (type help <topic>):
  ========================================
  EOF  bar  baz  exit  foo  help  quit

  myapp> shell
  error: unregonized command: shell (enter ? for help)
  
  myapp> exit

  $

Note that :func:`exclude_from_shell <clik_shell.exclude_from_shell>`
is not limited to the shell command itself -- it may be used on any
subcommand to exclude that subcommand from the shell interface.


Shell-Only Commands
===================

To create a command that is available only in the shell, define a new
``do_*`` method as outlined in the :mod:`cmd` documentation::

  import subprocess

  class Shell(DefaultShell):
      def do_clear(self, _):
          """Clear the terminal screen."""
          yield
          subprocess.call('clear')


Base Shell Classes
==================

:class:`DefaultShell <clik_shell.DefaultShell>` adds a few commonly
desired facilities to the default command loop:

* ``exit`` and ``quit`` commands to exit the shell
* ``EOF`` handler, which exits the shell on :kbd:`Ctl-D`
* ``KeyboardInterrupt`` handler, which exits the shell on :kbd:`Ctl-C`
* :meth:`cmd.Cmd.emptyline` override to a no-op (by default it runs
  the last command entered)

If you want to implement these facilities yourself, subclass
:class:`clik_shell.BaseShell` instead of the default shell. The base
shell defines only three methods on top of :class:`cmd.Cmd`:

* :meth:`__init__ <clik_shell.BaseShell.__init__>`, which dynamically
  generates the ``do_*`` and ``help_*`` methods
* :meth:`default <clik_shell.BaseShell.default>`, which overrides the
  default :meth:`cmd.Cmd.default` implementation in order to hack in
  support for hyphenated command names (see below)
* :meth:`error <clik_shell.BaseShell.error>`, which is called when a
  command exits with a non-zero code


Hyphenated Commands
===================

:mod:`cmd` does not natively support commands with hyphenated names --
commands are defined by creating a ``do_*`` method and methods may not
have hyphens in them. Due to this constraint, there's not much
clik-shell can do but work around it as best as possible:

* For the purpose of defining methods, all hyphens are converted to
  underscores -- so ``my-subcommand`` becomes ``my_subcommand``
* A hook is added to :meth:`cmd.Cmd.default` to recognize
  ``my-subcommand`` and redirect it to ``my_subcommand``

Le sigh. This sucks because:

* The underscore names aren't the "real" command names
* The hyphen names don't show up in the help documentation
* In theory someone could define ``my-subcommand`` **and**
  ``my_subcommand``, which totally breaks this scheme (in practice,
  anyone who designs a CLI where those two commands do different
  things deserves to have their app broken)

But, I mean, at least ``my-subcommand`` doesn't bail out. And that's
the *only* reason the workaround was implemented. Otherwise it's a
pretty ugly wart on an otherwise reasonably-designed API.
