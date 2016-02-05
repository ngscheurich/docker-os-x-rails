# Setup notes for Docker +  Rails on OS X

## Prerequisites

- [Homebrew](http://brew.sh/)
- [VirtualBox](https://www.virtualbox.org/wiki/Downloads)

## Install Docker

First, make sure your Homebrew is up to date and install the `docker-compose`
package. This will install Docker and its related binaries:

```shell
$ brew update
$ brew install docker-compose
```

Create a new Docker virtual machine that is configured for use with VirtualBox
(we’ll call the VM *default*). When the command completes, make sure that the
new VM is up and running:

```shell
$ docker-machine create --driver=virtualbox default
$ docker-machine ls
```

Run the following command to export the environment variables that Docker needs:

```shell
eval "$(docker-machine env default)"
```

> You should also add that command to your shell config so that the environment
variables are available in future sessions.

At this point you should verify that Docker can connect to the daemon and access the VM you made:

```shell
$ docker info
```

If all is well, give the Docker `hello-world` image a go:

```shell
$ docker run hello-world
```

This should pull the `hello-world` image from Docker Hub and run it.

Congrats, Docker is all set up!

## Dockerize a Rails app

### Create an app image
Create a Dockerfile with the following contents in your app’s root directory.
This will define a new Docker image for your app:

```
# Base this image on the official Ruby 2.2.3 image
FROM ruby:2.2.3

# Update Apt sources and install build tools
RUN apt-get update -qq && apt-get install -y build-essential

# Install dependencies for PostgreSQL
RUN apt-get install -y libpq-dev

# Install dependencies for Nokogiri
RUN apt-get install -y libxml2-dev libxslt1-dev

# Install dependencies for capybara-webkit
RUN apt-get install -y libqt4-webkit libqt4-dev xvfb

# Install a JavaScript runtime
RUN apt-get install -y nodejs

# Create a place for the Rails app to live and then change to it
ENV APP_DIR /railsapp
RUN mkdir $APP_DIR
WORKDIR $APP_DIR

# Install gems in the image
ADD Gemfile* $APP_DIR/
RUN bundle install

# Copy app files to image
ADD . $APP_DIR
```

Now, build the Docker image you defined by running the following command from
your app’s root directory:

```shell
$ docker build -t username/rails:latest .
```

> In the above command, `username` is your image’s author, `rails` is your image’s
name, and `latest` is your image’s author. You can change these to whatever you
like, or just omit the `-t` option.

If all went well, running `docker images` should show your newly created image
(as well as the `ruby` and `hello-world` images, if you’ve been following along).

### Define services

We’ll be using `docker-compose` to create services for our app. Docker services
let us easily spin up new Docker containers and link them together in a hassle-free
way. Create a `docker-compose.yml` file with the following contents in your app’s
root directory:

```yaml
# Define a service for the database
db:
  # Use the official PostgreSQL image as a base
  image: postgres:9.4.5
  # Expose default PostgreSQL port to the host
  ports:
    - "5432:5432"

# Define a service for the Rails app
web:
  # Specify path to the app’s Dockerfile
  build: .
  # Default command for `docker-compose up`
  command: bin/rails server --port 3000 --binding 0.0.0.0
  # Expose Rails server port to the host
  ports:
    - "3000:3000"
  # Link to the container in the db service
  links:
    - db
  # Mount the app directory on our host to the web service container
  volumes:
    - .:/railsapp
```

Now, edit your Rails database config to reference the *db* service.
For an app called *myapp*, your `config/database.yml` should look
somthing like:

```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: 5
  username: postgres
  host: db

development:
  <<: *default
  database: myapp_development

test:
  <<: *default
  database: myapp_test
```

> Setting the default host to simply *db* just “works” because docker-compose
creates an `/etc/hosts` entry in the *web* container so that the *db* container’s
IP address is aliased to *db*. Magic!

### Initial setup
Once these two configuration files are in place, you need to create and set up
the database:

```shell
$ docker-compose run web rake db:create db:setup
```

> `rake db:create db:setup` from the above command is what is sent to the *web*
container to process. Any command will work here—for instance, we could
`docker-compose run web /bin/bash` to get an interactive shell.

### Run the app

Now that the database is in place, we can bring the app up by simply:

```shell
$ docker-compose up
```

Running this command will bring the app services (*web* and *db*) up if necessary and then
process the command we specified for the *web* service, i.e., start the Rails server.

You should now be able to access your app on port 3000 on your currently-running Docker VM’s
IP address. If your Docker VM is called *default*, you can find its IP by running:

```shell
$ docker-machine ip default
```

> It’s probably a good idea to alias the Docker VM’s IP to something like *docker* in your hosts
file so you can just hit up http://docker:3000.

*Voilà!*

## Notes
* This guide is largely based on [*Rails on Docker* by thoughtbot](https://robots.thoughtbot.com/rails-on-docker)
* Handy Docker cheat sheet: [https://github.com/wsargent/docker-cheat-sheet](https://github.com/wsargent/docker-cheat-sheet)

---

[(ↄ) Copyleft]() Nicholas Gunther Scheurich under the [GPLv2](https://github.com/ngscheurich/docker-os-x-rails/blob/master/LICENSE)
