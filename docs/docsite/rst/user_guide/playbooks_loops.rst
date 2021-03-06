.. _playbooks_loops:

Loops
=====

Often you'll want to do many things in one task, such as create a lot of users, install a lot of packages, or
repeat a polling step until a certain result is reached.

This chapter is all about how to use loops in playbooks.

.. contents:: Topics

.. _standard_loops:

Standard Loops
``````````````

To save some typing, repeated tasks can be written in short-hand like so::

    - name: add several users
      user:
        name: "{{ item }}"
        state: present
        groups: "wheel"
      loop:
         - testuser1
         - testuser2

If you have defined a YAML list in a variables file, or the 'vars' section, you can also do::

    loop: "{{ somelist }}"

The above would be the equivalent of::

    - name: add user testuser1
      user:
        name: "testuser1"
        state: present
        groups: "wheel"
    - name: add user testuser2
      user:
        name: "testuser2"
        state: present
        groups: "wheel"

.. note:: Before 2.5 Ansible mainly used the ``with_<lookup>`` keywords to create loops, the `loop` keyword is basically analogous to ``with_list``.


Some plugins like, the yum and apt modules can take lists directly to their options, this is more optimal than looping over the task.
See each action's documentation for details, for now here is an example::

   - name: optimal yum
     yum:
       name: "{{list_of_packages}}"
       state: present

   - name: non optimal yum, not only slower but might cause issues with interdependencies
     yum:
       name: "{{item}}"
       state: present
     loop: "{{list_of_packages}}"

Note that the types of items you iterate over do not have to be simple lists of strings.
If you have a list of hashes, you can reference subkeys using things like::

    - name: add several users
      user:
        name: "{{ item.name }}"
        state: present
        groups: "{{ item.groups }}"
      loop:
        - { name: 'testuser1', groups: 'wheel' }
        - { name: 'testuser2', groups: 'root' }

Also be aware that when combining :doc:`playbooks_conditionals` with a loop, the ``when:`` statement is processed separately for each item.
See :ref:`the_when_statement` for an example.

To loop over a dict, use the ``dict2items`` :ref:`dict_filter`::

    - name: create a tag dictionary of non-empty tags
      set_fact:
        tags_dict: "{{ (tags_dict|default({}))|combine({item.key: item.value}) }}"
      loop: "{{ tags|dict2items }}"
      vars:
        tags:
          Environment: dev
          Application: payment
          Another: "{{ doesnotexist|default() }}"
      when: item.value != ""

Here, we don't want to set empty tags, so we create a dictionary containing only non-empty tags.


.. _complex_loops:

Complex loops
`````````````

Sometimes you need more than what a simple list provides, you can use Jinja2 expressions to create complex lists:
For example, using the 'nested' lookup, you can combine lists::

    - name: give users access to multiple databases
      mysql_user:
        name: "{{ item[0] }}"
        priv: "{{ item[1] }}.*:ALL"
        append_privs: yes
        password: "foo"
      loop: "{{ ['alice', 'bob'] |product(['clientdb', 'employeedb', 'providerdb'])|list }}"

.. note:: ``with_`` loops are actually a combination of things ``with_`` + ``lookup()``, even ``items`` is a lookup. ``loop`` can be used in the same way as shown above.


Using lookup vs query with loop
```````````````````````````````

In Ansible 2.5 a new jinja2 function was introduced named :ref:`query`, that offers several benefits over ``lookup`` when using the new ``loop`` keyword.

This is better described in the lookup documentation. However, ``query`` provides a simpler interface and a more predictable output from lookup plugins, ensuring better compatibility with ``loop``.

In certain situations the ``lookup`` function may not return a list which ``loop`` requires.

The following invocations are equivalent, using ``wantlist=True`` with ``lookup`` to ensure a return type of a list::

    loop: "{{ query('inventory_hostnames', 'all') }}"

    loop: "{{ lookup('inventory_hostnames', 'all', wantlist=True) }}"


.. _do_until_loops:

Do-Until Loops
``````````````

.. versionadded:: 1.4

Sometimes you would want to retry a task until a certain condition is met.  Here's an example::

    - shell: /usr/bin/foo
      register: result
      until: result.stdout.find("all systems go") != -1
      retries: 5
      delay: 10

The above example runs the shell module iteratively until the module's result has "all systems go" in its stdout or the task has
been retried for 5 times with a delay of 10 seconds. The default value for "retries" is 3 and "delay" is 5.

The task returns the results returned by the last task run. The results of individual retries can be viewed by -vv option.
The registered variable will also have a new key "attempts" which will have the number of the retries for the task.

.. note:: If the ``until`` parameter isn't defined, the value for the ``retries`` parameter is forced to 1.

Using register with a loop
``````````````````````````

After using ``register`` with a loop, the data structure placed in the variable will contain a ``results`` attribute that is a list of all responses from the module.

Here is an example of using ``register`` with ``loop``::

    - shell: "echo {{ item }}"
      loop:
        - "one"
        - "two"
      register: echo

This differs from the data structure returned when using ``register`` without a loop::

    {
        "changed": true,
        "msg": "All items completed",
        "results": [
            {
                "changed": true,
                "cmd": "echo \"one\" ",
                "delta": "0:00:00.003110",
                "end": "2013-12-19 12:00:05.187153",
                "invocation": {
                    "module_args": "echo \"one\"",
                    "module_name": "shell"
                },
                "item": "one",
                "rc": 0,
                "start": "2013-12-19 12:00:05.184043",
                "stderr": "",
                "stdout": "one"
            },
            {
                "changed": true,
                "cmd": "echo \"two\" ",
                "delta": "0:00:00.002920",
                "end": "2013-12-19 12:00:05.245502",
                "invocation": {
                    "module_args": "echo \"two\"",
                    "module_name": "shell"
                },
                "item": "two",
                "rc": 0,
                "start": "2013-12-19 12:00:05.242582",
                "stderr": "",
                "stdout": "two"
            }
        ]
    }

Subsequent loops over the registered variable to inspect the results may look like::

    - name: Fail if return code is not 0
      fail:
        msg: "The command ({{ item.cmd }}) did not have a 0 return code"
      when: item.rc != 0
      loop: "{{ echo.results }}"

During iteration, the result of the current item will be placed in the variable::

    - shell: echo "{{ item }}"
      loop:
        - one
        - two
      register: echo
      changed_when: echo.stdout != "one"



Looping over the inventory
``````````````````````````

If you wish to loop over the inventory, or just a subset of it, there are multiple ways.
One can use a regular ``loop`` with the ``ansible_play_batch`` or ``groups`` variables, like this::

    # show all the hosts in the inventory
    - debug:
        msg: "{{ item }}"
      loop: "{{ groups['all'] }}"

    # show all the hosts in the current play
    - debug:
        msg: "{{ item }}"
      loop: "{{ ansible_play_batch }}"

There is also a specific lookup plugin ``inventory_hostnames`` that can be used like this::

    # show all the hosts in the inventory
    - debug:
        msg: "{{ item }}"
      loop: "{{ query('inventory_hostnames', 'all') }}"

    # show all the hosts matching the pattern, ie all but the group www
    - debug:
        msg: "{{ item }}"
      loop: "{{ query('inventory_hostnames', 'all:!www') }}"

More information on the patterns can be found on :doc:`intro_patterns`

.. _loop_control:

Loop Control
````````````

.. versionadded:: 2.1

In 2.0 you are again able to use loops and task includes (but not playbook includes). This adds the ability to loop over the set of tasks in one shot.
Ansible by default sets the loop variable ``item`` for each loop, which causes these nested loops to overwrite the value of ``item`` from the "outer" loops.
As of Ansible 2.1, the ``loop_control`` option can be used to specify the name of the variable to be used for the loop::

    # main.yml
    - include_tasks: inner.yml
      loop:
        - 1
        - 2
        - 3
      loop_control:
        loop_var: outer_item

    # inner.yml
    - debug:
        msg: "outer item={{ outer_item }} inner item={{ item }}"
      loop:
        - a
        - b
        - c

.. note:: If Ansible detects that the current loop is using a variable which has already been defined, it will raise an error to fail the task.

.. versionadded:: 2.2

When using complex data structures for looping the display might get a bit too "busy", this is where the ``label`` directive comes to help::

    - name: create servers
      digital_ocean:
        name: "{{ item.name }}"
        state: present
      loop:
        - name: server1
          disks: 3gb
          ram: 15Gb
          network:
            nic01: 100Gb
            nic02: 10Gb
            ...
      loop_control:
        label: "{{ item.name }}"

This will now display just the ``label`` field instead of the whole structure per ``item``, it defaults to ``{{ item }}`` to display things as usual.

.. versionadded:: 2.2

Another option to loop control is ``pause``, which allows you to control the time (in seconds) between execution of items in a task loop.::

    # main.yml
    - name: create servers, pause 3s before creating next
      digital_ocean:
        name: "{{ item }}"
        state: present
      loop:
        - server1
        - server2
      loop_control:
        pause: 3

.. versionadded:: 2.5

If you need to keep track of where you are in a loop, you can use the ``index_var`` option to loop control to specify a variable name to contain the current loop index.::

    - name: count our fruit
      debug:
        msg: "{{ item }} with index {{ my_idx }}"
      loop:
        - apple
        - banana
        - pear
      loop_control:
        index_var: my_idx

.. versionadded:: 2.8

As of Ansible 2.8 you can get extended loop information using the ``extended`` option to loop control. This option will expose the following information.

==========================  ===========
Variable                    Description
--------------------------  -----------
``ansible_loop.allitems``   The list of all items in the loop
``ansible_loop.index``      The current iteration of the loop. (1 indexed)
``ansible_loop.index0``     The current iteration of the loop. (0 indexed)
``ansible_loop.revindex``   The number of iterations from the end of the loop (1 indexed)
``ansible_loop.revindex0``  The number of iterations from the end of the loop (0 indexed)
``ansible_loop.first``      ``True`` if first iteration
``ansible_loop.last``       ``True`` if last iteration
``ansible_loop.length``     The number of items in the loop
``ansible_loop.previtem``   The item from the previous iteration of the loop. Undefined during the first iteration.
``ansible_loop.nextitem``   The item from the following iteration of the loop. Undefined during the last iteration.
==========================  ===========

::

      loop_control:
        extended: yes

Migrating from with_X to loop
`````````````````````````````

.. include:: shared_snippets/with2loop.txt


.. seealso::

   :doc:`playbooks`
       An introduction to playbooks
   :doc:`playbooks_reuse_roles`
       Playbook organization by roles
   :doc:`playbooks_best_practices`
       Best practices in playbooks
   :doc:`playbooks_conditionals`
       Conditional statements in playbooks
   :doc:`playbooks_variables`
       All about variables
   `User Mailing List <https://groups.google.com/group/ansible-devel>`_
       Have a question?  Stop by the google group!
   `irc.freenode.net <http://irc.freenode.net>`_
       #ansible IRC chat channel
