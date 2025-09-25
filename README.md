# OpenMoxie
<p>
<img src="./site/static/hive/openmoxie_logo.svg" width="200" height="200">
Welcome!  You may be here looking for a solution to run your Embodied Moxie Robot
should their cloud infrastructure shut down.  If so, you are in the right place!  Some
of you may already be concerned this is going to be complicated, and for those who
are looking to just install and run something, we'll cover that first.
</p>

## What is OpenMoxie

A local network hub that Moxie can connect with and a local network service that provides the critical messaging
and services to make Moxie function to the extent it can using the internal software of the robot.  Much of the
Moxie content is supported like Daily Missions, Reading, and Wild Workout; but some of the newer modules like Ocean
Explorer, Animal Faces, and Story Maker are missing.  On the plus side, you will be able to control the schedule,
exclude modules your child dislikes, and write your own simple conversations to have with Moxie.

There's no parent app, all configuration is done through the web interface to OpenMoxie.

## What You Need

1.  You need a computer of some kind (PC, Linux, Mac, Raspberry Pi5) on the same wireless network you intend to use
your Moxie on.  This wireless network should be reasonable secure and should have a firewall, which is mostly what
home networks look like these days.
2. You need An account with OpenAI and credits to pay for Speech-to-Text and Chat
3. You need to install Docker (Docker Desktop https://www.docker.com/products/docker-desktop/ for example), don't buy any license as this is for personal use

# Quick Start

Unless you want to make changes, or your platform isn't supported, you should not need to clone this
repository.  There are images available on Docker hub for common PC platforms.

1. Download and install Docker to the target machine
2. Create a directory somewhere, for instance an `OpenMoxie` folder in your home directory
3. Download and save the latest [docker-compose.yml](./docker-compose.yml) and save it inside that directory
4. Open a terminal window in that directory
5. run `docker-compose pull` (this downloads the latest images)
6. run `docker-compose up -d` (this starts OpenMoxie to run in the background)
7. Visit http://localhost:8001/hive

Once running, the URL above should bring you to the setup page.  Details on setting up an OpenAI
account can be found there.  Also, Docker Desktop can be used to stop and restart this without returning
to the terminal, but a `local/work` directory will be created holding the database and log files for OpenMoxie.

Do take a look at the [Moxie Overview](doc/MoxieOverview.md) to learn more about the schedule and settings.

# Slow Start

Still reading?  Ah, yes.  You are my kind of people. If you want to tinker, change things, debug, or
just look under the hood a bit; you should clone the project.

## Components

Currently this project contains the following:

* Django app for basic web services and database (using sqlite3 bindings)
* pymqtt based service code to handle device
* A simple MQTT based STT provider using OpenAI Whisper
* A simple MQTT based remote chat service using single prompt inferences from OpenAI

## Telehealth Studio

The Telehealth Studio gives you a richer puppeteering surface for recorded performances. From the dashboard choose
**Telehealth Studio** next to a device to:

* Toggle Telehealth mode on the robot.
* Send auto-generated markup using the familiar mood/intensity sliders.
* Paste raw markup strings to chain gestures, pauses, screen graphics, and sound effects without losing timing control.

When running inside Docker alongside the rest of the OpenMoxie stack:

1. Build and start the full stack, including MQTT, from the project root:
   ```
   docker-compose up --build -d
   ```
2. Visit `http://localhost:8001/hive` and sign in.
3. From **Devices** choose **Telehealth Studio** for the target robot.
4. Press **Start Telehealth Mode** to wake the robot into Telehealth/puppet mode, then use either form to send output.

If you are already running the standard containers you can refresh just the Django service after pulling these changes:

```
docker-compose up --build -d server
```

The studio reuses the existing MQTT broker (`mqtt` service) and remote chat pipeline, so no extra processes are required.

## Code 

## Building Your Own Docker

The Dockerfile and docker_compose.yml provide a direct means to install and run the complete
system inside a container, including an MQTT broker with a self-signed certificate Moxie's should
be able to join.

1. Clone project
2. Install docker
3. run `docker-compose up --build -d`
4. Visit http://localhost:8001/hive
5. Depending on your PC firewall, expose port 8883

## Running Directly

You can run all the python project code locally, however you will still need an MQTT broker to coordinate
the networking between Moxie and OpenMoxie services.  For linux-based development, I installed mosquitto and
ran it natively.  It is probably simpler for most to use run the local docker version, which requires some
light editing.

1. Clone project
2. Install dependencies
```
python3 -m pip install -r requirements.txt
```
3. Make initial migrations
```
python3 site/manage.py makemigrations
```
4. Run initial migration
```
python3 site/manage.py migrate
```
5. Create a superuser 
```
python site/manage.py createsuperuser
```
6. Run the initial data import
```
python3 site/manage.py init_data
```
7. Edit `site\openmoxie\settings.py` and edit this block to point to localhost for mqtt.  Alternately, you can edit your local /etc/hosts file to include `127.0.0.1       mqtt`
```
MQTT_ENDPOINT = {
    'host': 'localhost',
    'port': 8883,
    'project': 'openmoxie',
    'cert_required': False,
}
```
9. Start the MQTT Broker
```
docker-compose up -d mqtt
```
8. Run the service (Note: no-reload is currently required to prevent the mqtt supervisor from being created twice for some reason.)
```
python3 site/manage.py runserver --noreload
```

Once it is running, you may visit http://localhost:8000/hive for the dashboard.

# MQTT Broker - Direct Install

I have been running this all from a headless Ubuntu system using mosquitto MQTT as a broker and configured it for anonymous
access, which is generally a bad idea when the services are running with payment attached.  But I wanted to share my setup
in case anyone wanted to replicate it as a starting point on their own system.

```
sudo apt-get install mosquitto
```

My site configuration for mosquitto in `/etc/mosquitto/conf.d/openmoxie.conf`
```
listener 8883
cafile /etc/letsencrypt/live/domain.com/chain.pem
keyfile /etc/letsencrypt/live/domain.com/privkey.pem
certfile /etc/letsencrypt/live/domain.com/cert.pem
allow_anonymous true
```

I'm using Let's Encrypt with my apache instance, and have allowed ACL permission for mosquitto to read them for it's SSL connections and
will be using a virtual host proxy to route openmoxie subdomain to the django port, so all external webtraffic is nicely encrypted.  The
config for reference.

```
<IfModule mod_ssl.c>
<VirtualHost *:443>
        ServerAdmin webmaster@localhost
        ServerName openmoxie.domain.com

        ErrorLog ${APACHE_LOG_DIR}/openmoxie_error.log
        CustomLog ${APACHE_LOG_DIR}/openmoxie_access.log combined
        SSLEngine on
        ProxyRequests Off
        ProxyPreserveHost On
        ProxyPass / http://localhost:8000/
        ProxyPassReverse / http://localhost:8000/

Include /etc/letsencrypt/options-ssl-apache.conf
SSLCertificateFile /etc/letsencrypt/live/domain.com/fullchain.pem
SSLCertificateKeyFile /etc/letsencrypt/live/domain.com/privkey.pem
</VirtualHost>
</IfModule>
```
