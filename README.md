# Sample Ruby App

These notes take you through the process of spinning up a new ruby app with
docker-compose. They've been cobbled together from various sources (mostly
a [post on
semaphoreci](https://semaphoreci.com/community/tutorials/dockerizing-a-ruby-on-rails-application)
and the [ruby dockerhub](https://hub.docker.com/_/ruby)) to suit my needs and
should represent a number of "good practices", if not necessarily "best
practices". The objective is to provice a useful development environment and
a reliable and easy to deploy production environment.

## Bootstrap the app
We'll put everything inside a single repository, including the supporting
docker and docker-compose infrastructure as well as the applicaiton code. 
```bash
$ mkdir example-app && cd example-app
$ git init .
```


`rails` can bootstap a new application, but doing so assumes we have rails
installed somewhere. We'll create a throwaway container to do that for us.
Create `Dockerfile.rails`
```
# Dockerfile.rails
FROM ruby:3.1.2 AS rails-toolbox

# Default directory
ENV INSTALL_PATH /opt/app
RUN mkdir -p $INSTALL_PATH

# Install rails
RUN gem install rails bundler
#RUN chown -R user:user /opt/app
WORKDIR /opt/app

# Run a shell
CMD ["/bin/sh"]
```

Build the `rails-toolbox` image with `docker build -t rails-toolbox
Dockerfile.rails`. Then use it to bootstrap the app
```bash
$ docker run -it -v $PWD:/opt/app rails-toolbox rails new --skip-bundle myapp
```

That should create `myapp` inside the `example-app` directory and populate it
with a rails app. It will also create a repository _inside_ the app directory,
which we don't want, so we will remove that.
```bash
$ rm -rf myapp/.git
```

Commit everything we have up to this point.

##


## TODO

[ ] Formalize production and development image differences

