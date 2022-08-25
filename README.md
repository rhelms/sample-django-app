# Sample Django Application

This is a sample Django application that mimicks a development environment and is being used to assist with
replication of issues experienced in PyCharm 2022.2.1.

## Build

Build the docker container
```bash
cd docker
docker compose build
```

Run the container.

Note: we do our development with containers running in a local swarm. My initial run will assume a local swarm, but 
I will attempt to replication using Docker Compose as well.

Alter `docker-compose.yml` to select appropriate available ports for your machine. By default, this project uses 8203 for web and 2225 for SSH.

Also uncomment the `volumes` clause that would allow the project `src` directory to be mounted into the container. 
**This is the preferred mode of operation and does not work with PyCharm 2022.2.1.**

## Deploy under Docker Swarm

Via local Swarm
```bash
docker stack deploy -c docker-compose.yml sample
```

### Test Deploy

Test the Django Server natively
```bash
docker ps | grep sample-web
docker exec -it <container id> bash
./manage.py migrate
./manage.py runserver 0.0.0.0:8000
```

Access the app via http://127.0.0.1:8203, nominating which ever port you specified in the `docker-compose.yml` file.

You should see the default Django install screen, with "The install worked successfully! Congratulations!" across the middle.

## PyCharm Setup under Docker Swarm via SSH and mounted source

### Interpreter

Now to set up an SSH Interpreter using the system distribution.

Access File | Settings | Project: sample-django-app | Python Interpreter and select Add Interpreter | On SSH...

* SSH Connection: New
* Host: 127.0.0.1
* Port: 2225
* Username: root
* Password: root 
* Check "Save password"

Introspection completes successfully. Select Next

Select System Interpreter

* Leave interpreter as default (/usr/bin/python3)
* Wipe Sync directory (because we don't want to actually sync anything)

Interpreter will display. Select Apply.

Go to Build, Execution, Deployment | Deployment
* Select the recently created server. It should have a name like "root@127.0.0.1:2225 password".
* Update Root path to be /opt/sample
* Leave "Use Rsync doe download/upload/sync" unselected

Go to Mappings
* Append "/src" to the Local Path (i.e. /home/myuser/work/sample-django-app/src)
* Change Deployment path to /

At this point, the File Transfer tab will show files being transfered. 

**This is a part of the problem. I already have the volume mounted into the container and don't want to sync any files. I only want to have a mapping for debugging purposes.**

But let's press on.

### Django Server

Set up a Django Server configuration

Select Edit Configurations

Select + and Django Server
* Name: Sample Server
* Host: 0.0.0.0
* Port: 8000
* Python interpreter: <Select the remote interpreter just created. Mine was called "Remote Python 3.9.2 (/usr/bin/python3)" by default>
* Working directory: <Select the src directory of the project (i.e. /home/myuser/work/sample-django-app/src)>

Select Fix to set up Django support. You will get a new Settings dialog at the Languages & Frameworks > Django section

* Django project root: /home/myuser/work/sample-django-app/src
* Settings: sample/settings.py
* Leave everything else blank. Select OK

The dialog will close, and you will be returned to the Run/Debug Configurations dialog.

At this point, the "Error: Django is not importable in this environment" message will show at the bottom.

You may need to reselect the Python Interpreter

Select Apply and then OK

### Django Server Test

Select the Run icon to run the Sample Server configuration.
The Edit Configuration dialog will show due to the "Error: Django is not importable in this environment" issue.
* Select Run
* Select Continue Anyway

This is where replication of the issue is coming unstuck. When I open the Edit Configuration for the Django Server, the interpreter 
is being reset back to my default system interpreter. If I select the appropriate remote interpreter, the Apply option does
not become available and any attempt to run just uses my local interpreter.

As it turns out, when the initial sync was done, all of the files under `src` were reduced to a zero size.

## Deploy under Docker Swarm (no mounted drive)

You may need to repull the repo to restore the files that were reduced to zero size.

Ensure the `docker-compose.yml` file has the `volumes` section commented out. This time we will try using rsync in 
an attempt to show the issue with being unable to stop the Django Server once started.

To start with a clean slate, remove the existing Interpreter, Deployment and SSH Configuration associated with this deployment.

Via local Swarm
```bash
docker stack deploy -c docker-compose.yml sample
```

This time we have no source code in our container (unless we copy it to `docker/sample/src` first, which is what we do
in our build scripts), so we can't do a native Django Server test until after the interpreter has been set up and the
initial rsync is done.

Follow the previous [Interpreter](#interpreter) instructions, except map the local project `src` to `/opt/sample` and enable rsync.

Now we can test the server directly in the container.

```bash
python3 manage.py migrate
python3 manage.py runserver 0.0.0.0:8000
```

We've had to switch to specifying `python3` because the default rsync settings do not preserve executable permissions on `manage.py`.

Verify that the website is accessible via http://127.0.0.1:8203.

Alter the previous Django Server to nominate the new Remote Interpreter or delete the previous and set up a new Remote Interpreter.

Again, replication of the issue has come unstuck due to not being able to save the Remote Interpreter to the Django Server configuration.

