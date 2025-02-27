============
tap-redshift
============


`Singer <https://singer.io>`_ tap that extracts data from a `Redshift <https://aws.amazon.com/documentation/redshift/>`_ database and produces JSON-formatted data following the Singer spec.


Forked to symon-ai/tap-redshift to fix build problems
=====================================================
* tap-redshift could not find ``pytz`` library. Solved by naming ``pytz`` in setup.py.
* tap-redshift could not find path to ``pg_config``. Solved by importing ``psycopg2-binary`` instead of ``psycopg2``.

Usage
=====
tap-redshift assumes you have a connection to Redshift and requires Python 3.6+.

Step 1: Create a configuration file
-----------------------------------
When you install tap-redshift, you need to create a ``config.json`` file for the database connection.

The json file requires the following attributes;

* ``host``
* ``port``
* ``dbname``
* ``user``
* ``password``
* ``start_date`` (Notation: yyyy-mm-ddThh:mm:ssZ)

And optional attributes;

* ``schema``
* ``ssl`` (set to "true" to require SSL)

Example:

.. code-block:: json

    {
        "host": "REDSHIFT_HOST",
        "port": "REDSHIFT_PORT",
        "dbname": "REDSHIFT_DBNAME",
        "user": "REDSHIFT_USER",
        "password": "REDSHIFT_PASSWORD",
        "start_date": "REDSHIFT_START_DATE",
        "schema": "REDSHIFT_SCHEMA",
        "ssl": "true"
    }


Step 2: Discover what can be extracted from Redshift
----------------------------------------------------
The tap can be invoked in discovery mode to get the available tables and columns in the database.
It points to the config file created to connect to redshift:

.. code-block:: shell

    $ tap-redshift --config config.json -d

A full catalog tap is written to stdout, with a JSON-schema description of each table. A source
table directly corresponds to a Singer stream.

Redirect output from the tap's discovery mode to a file so that it can be modified when the tap is
to be invoked in sync mode.

.. code-block:: shell

    $ tap-redshift -c config.json -d > catalog.json

This runs the tap in discovery mode and copies the output into a ``catalog.json`` file.

A catalog contains a list of stream objects, one for each table available in your Redshift schema.

Example:

.. code-block:: json

    {
        "streams": [
            {
                "tap_stream_id": "sample-dbname.public.sample-name",
                "stream": "sample-stream",
                "database_name": "sample-dbname",
                "table_name": "public.sample-name"
                "schema": {
                    "properties": {
                        "id": {
                            "minimum": -2147483648,
                            "inclusion": "automatic",
                            "maximum": 2147483647,
                            "type": [
                                "null",
                                "integer"
                            ]
                        },
                        "name": {
                            "maxLength": 255,
                            "inclusion": "available",
                            "type": [
                                "null",
                                "string"
                            ]
                        },
                        "updated_at": {
                            "inclusion": "available",
                            "type": [
                                "string"
                            ],
                            "format": "date-time"
                        },
                    },
                    "type": "object"
                },
                "metadata": [
                    {
                        "metadata": {
                            "selected-by-default": false,
                            "selected": true,
                            "is-view": false,
                            "table-key-properties": ["id"],
                            "schema-name": "sample-stream",
                            "valid-replication-keys": [
                                "updated_at"
                            ]
                        },
                        "breadcrumb": [],
                    },
                    {
                        "metadata": {
                            "selected-by-default": true,
                            "sql-datatype": "int2",
                            "inclusion": "automatic"
                        },
                        "breadcrumb": [
                            "properties",
                            "id"
                        ]
                    },
                    {
                        "metadata": {
                            "selected-by-default": true,
                            "sql-datatype": "varchar",
                            "inclusion": "available"
                        },
                        "breadcrumb": [
                            "properties",
                            "name"
                        ]
                    },
                    {
                        "metadata": {
                            "selected-by-default": true,
                            "sql-datatype": "datetime",
                            "inclusion": "available",
                        },
                        "breadcrumb": [
                            "properties",
                            "updated_at"
                        ]
                    }
                ]
            }
        ]
    }


Step 3: Select the tables you want to sync
------------------------------------------
In sync mode, ``tap-redshift`` requires a catalog file to be supplied, where the user must
have selected which streams (tables) should be transferred. Streams are not selected by default.

For each stream in the catalog, find the ``metadata`` section. That is the section you will modify
to select the stream and, optionally, individual properties too.

The stream itself is represented by an empty breadcrumb.

Example:

.. code-block:: json

    "metadata": [
        {
            "breadcrumb": [],
            "metadata": {
                "selected-by-default": false,
                ...
            }
        }
    ]

You can select it by adding ``"selected": true`` to its metadata.

Example:

.. code-block:: json

    "metadata": [
        {
            "breadcrumb": [],
            "metadata": {
                "selected": true,
                "selected-by-default": false,
                ...
            }
        }
    ]

The tap can then be invoked in sync mode with the properties catalog argument:

Example (paired with ``target-datadotworld``)

.. code-block:: shell

    tap-redshift -c config.json --catalog catalog.json | target-datadotworld -c config-dw.json


Step 4: Sync your data
----------------------
There are two ways to replicate a given table. FULL_TABLE and INCREMENTAL.
FULL_TABLE replication is used by default.

Full Table
++++++++++
Full-table replication extracts all data from the source table each time the tap is invoked without
a state file.

Incremental
+++++++++++
Incremental replication works in conjunction with a state file to only extract new records each
time the tap is invoked i.e continue from the last synced data.

To use incremental replication, we need to add the ``replication_method`` and ``replication_key``
to the streams (tables) metadata in the ``catalog.json`` file.

Example:

.. code-block:: json

    "metadata": [
        {
            "breadcrumb": [],
            "metadata": {
                "selected": true,
                "selected-by-default": false,
                "replication-method": "INCREMENTAL",
                "replication-key": "updated_at",
                ...
            }
        }
    ]

We can then invoke the tap again in sync mode. This time the output will have ``STATE`` messages
that contains a ``replication_key_value`` and ``bookmark`` for data that were extracted.

Redirect the output to a ``state.json`` file. Normally, the target will echo the last STATE after
it has finished processing data.

Run the code below to pass the state into a ``state.json`` file.

Example:

.. code-block:: shell

    tap-redshift -c config.json --catalog catalog.json | \
        target-datadotworld -c config-dw.json > state.json

The ``state.json`` file should look like;

.. code-block:: json

    {
        "currently_syncing": null,
        "bookmarks": {
            "sample-dbname.public.sample-name": {
                "replication_key": "updated_at",
                "version": 1516304171710,
                "replication_key_value": "2013-10-29T09:38:41.341Z"
            }
        }
    }

For subsequent runs, you can then invoke the incremental replication passing the latest state in order to limit data only to what has been modified since the last execution.

.. code-block:: shell

    tail -1 state.json > latest-state.json; \
    tap-redshift \
        -c config-redshift.json \
        --catalog catalog.json \
	    -s latest-state.json | \
                target-datadotworld -c config-dw.json > state.json


Running locally
===============


Ensure poetry is installed on your machine. 

* This command will return the installed version of poetry if it is installed.
.. code-block:: shell
    poetry --version

* If not, install poetry using the following commands (from https://python-poetry.org/docs/#installation):
.. code-block:: shell
    curl -sSL https://install.python-poetry.org | python3 -
    PATH=~/.local/bin:$PATH


Within the `tap-redshift` directory, install dependencies:
.. code-block:: shell
    poetry install

Then run the tap:
.. code-block:: shell
    poetry run tap-redshift <options>

Copyright &copy; 2019 Stitch
