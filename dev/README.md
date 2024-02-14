```
       ____                  __          ____          .-.
      / __ \____  _____ ____/ /_  ____  / / /_    .--./ /      _.---.,
     / /_/ / __ `/ ___/ ___/ __ \/ __ \/ / __/     '-,  (__..-`       \
    / ____/ /_/ (__  |__  ) /_/ / /_/ / / /_          \                |
   /_/    \__,_/____/____/_,___/\____/_/\__/           `,.__.   ^___.-/
                                                         `-./ .'...--`
  The open source password manager for teams                `'
  (c) 2024 Passbolt SA
  https://www.passbolt.com
```

# Setting up a working development environment using docker

**⚠️  Note:** These instructions are for setting a functional development environment only. We're exposing here sensitive ports and mounting code volumes for rapid development and debugging of the server stack. Refer to the [installation guide](https://help.passbolt.com/hosting/install) if you want to use Passbolt securely to share passwords with your team.

## Prerequisites
  - [Git](https://git-scm.com/)
  - [Docker](https://docs.docker.com/get-docker/) v20 or newer

## Fork & configure passbolt_api

1. Clone the [passbolt/passbolt_api](https://github.com/passbolt/passbolt_api) repo
Fork the Passbolt API repository. Please read [Fork a repo](https://docs.github.com/en/get-started/quickstart/fork-a-repo?tool=webui) if you've never done this before.

2. Clone the forked repository onto your local machine:
```bash
git clone git@github.com:<YOUR_FORK_HERE>/passbolt_api.git
```

3. Copy the default app configuration file (`app.default.php`) into a new one(`app.php`) for passbolt_api (the new file will be used by the passbolt server)
```bash
cp config/app.default.php config/app.php
```

4. Install composer dependencies, a new vendor directory will be created with all the required libraries
```
docker run --rm --interactive --tty --volume $PWD:/app composer install --ignore-platform-reqs
```

## Spin up development stack using docker compose

1. Clone the [passbolt_docker](https://github.com/passbolt/passbolt_docker) repository:
```bash
git clone https://github.com/passbolt/passbolt_docker.git
```

2. Copy the `.env.example` file into `.env` and replace the `PATH_TO_PASSBOLT_API` variable with the path to the passbolt_api repository on your machine.

Example:
```
PASSBOLT_API_PATH="/Users/maki/code/passbolt_api"
```

3. Spin-up the docker-compose containers (mariadb, passbolt server, etc.) required for passbolt API development
```bash
docker-compose -f dev/docker-compose-dev.yaml up -d
```

4. Create the first user (the administrator) by replacing the below command with your own data. More details [here](https://help.passbolt.com/hosting/install/ce/docker).
```bash
docker-compose -f dev/docker-compose-dev.yaml exec passbolt /bin/bash -c \
  'su -m -c "/var/www/passbolt/bin/cake passbolt register_user -u <email> \
  -f <first name>  -l <last name>  -r admin" -s /bin/sh www-data'
```

For example:
```
docker-compose -f dev/docker-compose-dev.yaml exec passbolt /bin/bash -c \
  'su -m -c "/var/www/passbolt/bin/cake passbolt register_user -u john@passbolt.local \
  -f John  -l Doe  -r admin" -s /bin/sh www-data'
```

Above command will return a link to setup your first administrator. But before that you'll have to add a new host entry in your `/etc/hosts` file for "passbolt.local" domain.

For example:
```
127.0.0.1   passbolt.local
```

That's all!

# Enable POSTGRES

A postgres service will also be stared if you use the `pgsql` profile:
```
docker-compose -f dev/docker-compose-dev.yaml --profile pgsql up
```

The default env variables are defined in `dev/env/pgsql.env`.

# Setup LDAP

1. Add an entry for `ldap.local` inside your /etc/hosts file:
```
127.0.0.1   ldap.local
```
2. Run docker compose with the `--profile ldap` parameter
3. Visit http://localhost:8080/ using your web browser, and login with the following credentials:
  - Login DN:  `cn=admin,dc=example,dc=org`
  - Password:  `admin`
4. Click the Import button and upload the [LDAP init file](./ldap/init.ldiff)
5. Navigate to https://passbolt.local, login and put the following configuration under Administration > User Directory:
  - Directory Type:  `Open Ldap`
  - Server url:      `ldap://openldap:389`
  - Username:        `cn=admin,dc=example,dc=org`
  - Password:        `admin`
  - Domain:          `ldap.local`
  - Base DN:         `dc=example,dc=org`
6. Click the button to test the configuration, ensure the dummy data has been processed and click the button to save the settings

**Note:** If you get an "Internal Server Error" while testing the configurations and you are using the php debug mode, set the debug flag to false (e.g. in passbolt.php) and try again.

# Setup xDebug

In order to setup xDebug with an IDE or code editor, please use dev/Dockerfile or docker-compose/docker-compose-dev.yaml to spin up a development stack, which already contains xDebug configured to run within the Passbolt server.
You will then have to configure your IDE to connect to xDebug. Below are the steps required for a few IDEs:

## Visual Studio Code

1. From the Extensions tab, install the "PHP Debug" extension from the "Xdebug" publisher
2. In the "Run and Debug" tab, click the gear icon at the very top of the panel to "Open 'launch.json'"
3. Under "configurations", add a new JSON object with the following content:
```
{
  "name": "Listen for Xdebug on Docker",
  "type": "php",
  "request": "launch",
  "port": 9003,
  "pathMappings": {
    "/var/www/passbolt": "${workspaceFolder}"
  }
},
```
4. Check for errors by adding `xdebug_info(); die();` to the Passbolt `webroot/index.php` file and visiting the Passbolt server root page. If you don't see anything under the "Diagnosis" section, you can remove this change and start using xDebug
5. In the "Run and Debug" tab, select the debug profile we added in "launch.json" ("Listen for Xdebug on Docker") and click the green arrow to connect to xDebug

In case the "${workspaceFolder}" value is not mapped correctly (this seems to be the case on MacOS), you can provide the full path of the open workspace folder:
* Either manually: `<ABSOLUTE_PATH_TO_WORKSPACE_FOLDER>`
* Or if you store all your code fodlers in a common code place:
  - `<ABSOLUTE_PATH_TO_COMMON_CODE_FOLDER>/${workspaceFolderBasename}` OR
  - `${workspaceRoot}` (deprecated in vscode but still works, though only works in single-workspace setup)

## PHPStorm

1. Configure your IDE so that it can properly connect with Docker: under Settings/Preferences -> Build, Execution, Deployment -> Docker. Here is a tutorial: https://www.jetbrains.com/help/phpstorm/docker.html#enable_docker
2. Then, under Settings/Preferences -> PHP -> Debug, in the "External connections" section, make sure the "Break at first line in PHP scripts" checkbox is unchecked
3. Thereafter, we need to configure a PHP server, which can be done by going to File > Settings > PHP > Servers. Click on the plus sign twice to create two servers:
  - The first server should be set to xDebug: Name="Docker - Passbolt", Host="passbolt.local", Port="9003", Debugger="Xdebug", ProjectFiles="<PATH_TO_PASSBOLT_API_REPO>", AbsolutePath="/var/www/passbolt"
  - The second server should be set to the web server (passbolt.local): Name="passbolt.local", Host="passbolt.local", Port="443", Debugger="Xdebug", ProjectFiles="<PATH_TO_PASSBOLT_API_REPO>", AbsolutePath="/var/www/passbolt"
4. Save and close the Settings/Preferences window
5. At the top of the main window, click the "Add Configuration..." button, then "Add new..." > "PHP Remote Debug"
6. Check the "Filter debug connection by IDE key", and fill in the form with: Name="Docker - Passbolt", Server="Docker - Passbolt", IDEKey="docker_passbolt"
7. At the top of the main window, click the "Listen for PHP debug connection" button and start a debugging session

**Note:** If you want to debug tests, you will need to properly setup the PHPUnit library under Settings/Preferences -> PHP -> Test Frameworks by adding a configuration and setting the path to phpunit.phar to "/var/www/passbolt/vendor/bin/phpunit" (the docker path mapping must be setup for it to work with this path)
