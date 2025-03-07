# Configure OPAL
Both OPAL-Server and OPAL client provide multiple ways to load configuration:
 - via environement variables (prefixed with 'OPAL_')
 - via command line values
 - via a '.env' or '.ini' file.

You can combine configuration sources; be aware of the ovveride rules:
default < .env file < env variable < command line
i.e. the settings in the env file ovveride the defaults, but both env-variables and command line values override them.

## What are the configuration variables available
To view the configuration settings for both OPAL-SERVER and OPAL-CLIENT you can do one of:
  - Run the server or client as a CLI - and use the '--help' command.
   - using the '--help' on specific commands such as 'run' will provide more information
  - Look at the configuration code itself:
    - [Common config](https://github.com/permitio/opal/blob/master/opal_common/config.py) : values available for both server and client
    - [Server config](https://github.com/permitio/opal/blob/master/opal_server/config.py) : values available only for the server
    - [Client config](https://github.com/permitio/opal/blob/master/opal_client/config.py) : values available only for the client

## Configuration architecture
OPAL's configuration is based on our very own `Confi` module, which in turn is based on [Decouple](https://pypi.org/project/python-decouple/), and adds complex value parsing with Pydantic, and command line arguments via Typer/Click.


## Configuring logs
OPAL supports to log out puts - STDERR/STDOUT and a log file review the variables prefixed with `LOG_` for the options.
