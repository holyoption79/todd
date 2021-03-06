User-Defined Testlets
================================

One of the most important original design principles for ToDD was the ability for users to easily define their own testing. Indeed, this has become one of ToDD's biggest advantages over other testing solutions, both open-source and commercial.

The idea is to allow the user to use any testing application (provided it is available on the system on which the ToDD agent is running.  If the user writes a script to wrap around an existing application, the testlet should handle or pass along the input/arguments as well as parse any output from the underlying application. Naturally, a testlet can perform tests itself, so the user can also write a totally self-contained testing program, provided it follows the testlet standard, which is documented below.

Referring to a Testlet
----------------------

When you want to run a certain testlet, you refer to it by name. There are a number of `testlets built-in to ToDD <nativetestlets/nativetestlets.html>`_ and are therefore reserved:

* http
* bandwidth
* ping
* portknock

Provided it has a unique name, and that it is executable (pre-compiled binary, Python script, bash script, etc.) then it can function as a testlet. Early testlets were actually just bash scripts that wrapped around existing applications like iperf or ping, and simply parsed their output.

.. NOTE::
    All native testlets maintain their own documentation. Please view the links at the top of `Native Testlets <nativetestlets/nativetestlets.html>`_ for more information about these testlets, such as what arguments they require, and a sample of their output.


Check Mode
----------
Each testlet must support a "check mode". This is sort of a "pre-test" check to ensure the testlet can be run. When running in "check mode", a testlet should test it's own ability to run a "real" test, such as sending traffic to localhost to ensure it can use the network stack.

For instance, when the ToDD agent runs the "ping" testlet in check mode, it would invoke it like this:

.. code-block:: text

    ping check

That said, the ToDD Server will distribute testrun instructions to the agents in two phases:

* Install - run the referenced testlet in check mode, then record all of the parameters for the intended test in the agent's cache
* Execute - run the installed testrun instruction

Input
-----
Obviously, testing application vary greatly in terms of their input. Some testing applications use certain command-line arguments or flags, and others aren't even configured via the command-line.

The idea of a ToDD testlet is to standardize this input so that any testing application can be run identically. This was a very useful concept early in ToDD's life, as the very first testlets were simple bash scripts that wrapped existing applications like ``ping`` and ``iperf``, and passed around the required arguments to make them conform to the standard we'll discuss now.

All testlets must follow the following standard input:

.. code-block:: text

    ./testletname < target > < args >

* "target" - this is always the first parameter. ``todd-agent`` will spin up N instances of a testlet, where N equals the number of targets that a given agent has been instructed to test against. So, the testlet must accept the target's IP address or hostname as it's first argument.
* "args" - any arguments required by the testlet. These could be arguments required by the testlet itself, or they could be arguments required by an underlying application that the testlet is wrapping. This depends on the testlet implementation.

Output
------
The output for every testlet is a single-level JSON object, which contains key-value pairs representing the metrics gathered for that testlet.

Since the ToDD agent is responsible for executing a testlet, it will watch ``stdout``, which is where ``todd-agent`` will output this JSON object. This makes testlets a very flexible method of performing tests; since it only needs to output these metrics as JSON to stdout, the testlet can be written in any language, as long as they support the input and output standardized in this document.

A sample JSON object that the "ping" testlet will provide is shown below:

.. code-block:: text

    {
        "avg_latency_ms": 27.007,
        "packet_loss_percentage": 0
    }

.. NOTE::
    The ToDD agent does not have an opinion on the values contained in the keys or values for this JSON object, or how many k/v pairs there are - only that it is valid JSON, and is a single level object (no nested objects, lists, etc). It also doesn't care about the datatype for these metrics. In this case, the first metric is a float, and the second is an integer. Both are passed as-is to the TSDB, or presented to the user.

The JSON document shown above contains the metrics for a single testlet run, which means that this is relevant to only a single target, run by a single ToDD agent. The ToDD agent will receive this output once for each target in the testrun, and submit this entire dataset up to the ToDD server via the ``comms`` system when finished.

The ToDD Server will also aggregate each agent's report to a single metric document for the entire testrun, so that it's easy to see the metrics for each source-to-target relationship for a testrun.

Installing a User-Defined Testlet
---------------------------------

Installing a user-defined testlet is easy, once you've written it to conform to the standards above. All you need to do is place it in the directory ``/opt/todd/server/assets/testlets`` on the server running ``todd-server``, and ToDD will ensure that the testlet is deployed to the agents.

.. NOTE::

    When ToDD was first released, testlets were all written in bash and packaged with ToDD by embedding them in some clever Go code, and then "unpacked" into this directory. So, you might see some testlets in this directory that you didn't write, like ``iperf``, or ``http``. This is a temporary measure, and will not exist in a future version of ToDD, once those testlets are implemented in Go.

If you want to make changes to these testlets, just overwrite the file in this directory - ToDD will detect those changes and continually make sure the agents are updated.