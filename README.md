# Using netbox-docker
1. Navigate to a directory and use `git clone -b release https://github.com/netbox-community/netbox-docker.git`
2. In the root of the new folder (netbox-docker), create `plugin_requirements.txt`, `Dockerfile-Plugins`, `docker-compose.override.yml`, `entrypoint.sh`
3. In `plugin_requirements.txt`, list the plugins you would like to use
	1. For external plugins, list the name (e.g. `netbox-bgp`)
	2. For local plugins, list the path (`/opt/netbox/plugins/plugin-name`)
4. Copy the following into `Dockerfile-Plugins`

```Dockerfile
FROM netboxcommunity/netbox:latest
COPY ./plugin_requirements.txt /opt/netbox/
COPY plugins/ /opt/netbox/plugins/
RUN /opt/netbox/venv/bin/pip install --no-warn-script-location -r /opt/netbox/plugin_requirements.txt

# These lines are only required if your plugin has its own static files.
COPY configuration/configuration.py /etc/netbox/config/configuration.py
COPY configuration/plugins.py /etc/netbox/config/plugins.py
RUN SECRET_KEY="dummydummydummydummydummydummydummydummydummydummy" /opt/netbox/venv/bin/python /opt/netbox/netbox/manage.py collectstatic --no-input
COPY entrypoint.sh /entrypoint.sh
RUN chmod -R ugo+wx /opt/netbox/venv/lib/python3.11/site-packages/

ENTRYPOINT [ "/entrypoint.sh" ]
CMD [ "/opt/netbox/launch-netbox.sh" ]
```

5. Copy the following into `docker-compose.override.yml`

```YAML
services:
	netbox:
		image: netbox:latest-plugins
		ports:
			- 8000:8080
		build:
			context: .
			dockerfile: Dockerfile-Plugins
	netbox-worker:
		image: netbox:latest-plugins
		build:
			context: .
			dockerfile: Dockerfile-Plugins
	netbox-housekeeping:
		image: netbox:latest-plugins
		build:
			context: .
			dockerfile: Dockerfile-Plugins
	postgres:
		ports:
			- 5432:5432
```

6. Copy the following into `entrypoint.sh`

```BASH
#!/bin/bash

# Activate the virtual environment
source /opt/netbox/venv/bin/activate

# Run a Python script to import PLUGINS and make migrations
/opt/netbox/venv/bin/python <<EOF
import sys
from subprocess import call

# Add the directory containing plugins.py to the search path
plugins_dir = '/etc/netbox/config'
sys.path.append(plugins_dir)

# Import the PLUGINS variable
from plugins import PLUGINS

# Run makemigrations for each plugin
for plugin in PLUGINS:
	call(['/opt/netbox/venv/bin/python', '/opt/netbox/netbox/manage.py', 'makemigrations', plugin])

# Clean up by removing the directory from sys.path if necessary
sys.path.remove(plugins_dir)
EOF

/opt/netbox/venv/bin/python /opt/netbox/netbox/manage.py migrate

/opt/netbox/venv/bin/python <<EOF
import sys
from subprocess import call

# Add the directory containing plugins.py to the search path
plugins_dir = '/etc/netbox/config'
sys.path.append(plugins_dir)

# Import the PLUGINS variable
from plugins import PLUGINS

# Run makemigrations for each plugin
for plugin in PLUGINS:
	call(['/opt/netbox/venv/bin/python', '/opt/netbox/netbox/manage.py', 'reindex', plugin])

# Clean up by removing the directory from sys.path if necessary
sys.path.remove(plugins_dir)
EOF

/opt/netbox/venv/bin/python /opt/netbox/netbox/manage.py reindex

# Execute the main command (e.g., start NetBox)
exec "$@"
```

7. Then run `chmod +x entrypoint.sh`
8. In env/netbox.env, add the following lines

```
DEBUG=true
DEVELOPER=true
```

8. In env/postgres.env, add `DB_PORT=5432`
9. Create a folder `plugins` in the `netbox-docker` directory

## To Create a Plugin
2. Create the plugin folder in `plugins`
3. Add `plugin_name` to configuration/plugins.py
4. Add `/opt/netbox/plugins/plugin-name` to plugin_requirements.txt
5. See [NetBox Plugin Tutorial](https://github.com/netbox-community/netbox-plugin-tutorial)
6. To verify proper installation, go to Admin > System in the NetBox dashboard and you should see the plugin listed under Plugins

> [!important]
> When creating `__init__.py`, use `from netbox.plugins import PluginConfig` instead of `from extras.plugins import PluginConfig`
> 
> Additionally, when creating templates, make sure to create MANIFEST.in to the plugin's root directory with the line `recursive-include plugin_name/templates *.html`

> [!note]
> There is no need to manually perform migrations or run `setup.py`; the Dockerfile should take care of that.

## To Run the Server
1. Use `docker compose build`
2. Use `docker compose up -d`
	1. The `-d` means detached
3. Go to `localhost:8000` in your browser

> [!note]
> The first time you run the app, you must use `docker compose exec netbox python manage.py createsuperuser` to create an admin user. The email is not required
