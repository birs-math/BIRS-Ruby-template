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
$ vi .gitignore
.env
*.swp
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

## Basic App Configuration
We'll be using postgresql, redis and sidekiq, so add the necessary gems to `myapp/Gemfile`.
```bash
$ vi myapp/Gemfile
  ...
 +gem 'unicorn', '~> 6.1.0'
 +gem 'pg', '~> 1.3.5'
 +gem 'sidekiq', '~> 6.4.2'
 +gem 'redis-rails', '~> 5.0.2'
```

Add a DRY database config to `myapp/config/database.yml`
```yaml
development:
  url: <%= (ENV.has_key?('DATABASE_URL') ? ENV['DATABASE_URL'] : '').gsub('?', '_development?') %>

test:
  url: <%= (ENV.has_key?('DATABASE_URL') ? ENV['DATABASE_URL'] : '').gsub('?', '_test?') %>

staging:
  url: <%= (ENV.has_key?('DATABASE_URL') ? ENV['DATABASE_URL'] : '').gsub('?', '_staging?') %>

production:
  url: <%= (ENV.has_key?('DATABASE_URL') ? ENV['DATABASE_URL'] : '').gsub('?', '_production?') %# SQLite. Versions 3.8.0 and up are supported.
```

Create a `myapp/config/secrets.yml`
```yaml
development: &default
  secret_key_base: <%= ENV['SECRET_TOKEN'] %>

test:
  <<: *default

staging:
  <<: *default

production:
  <<: *default
```

Add a basic application config to `myapp/config/application.rb`a
```ruby
  ...
module Workshops
  class Application < Rails::Application
    # Initialize configuration defaults for originally generated Rails version.
    config.load_defaults 7.1

    # Please, add to the `ignore` list any other `lib` subdirectories that do
    # not contain `.rb` files, or that should not be reloaded or eager loaded.
    # Common ones are `templates`, `generators`, or `middleware`, for example.
    config.autoload_lib(ignore: %w(assets tasks))

    config.log_level = :debug
    config.log_tags  = [:subdomain, :uuid]
    config.logger    = ActiveSupport::TaggedLogging.new(Logger.new(STDOUT))

    config.cache_store = :redis_store, ENV['CACHE_URL'],
                         { namespace: 'myapp:cache' }

    config.active_job.queue_adapter = :sidekiq
  end
end
```

Create a unicorn config `myapp/config/unicorn.rb`
```ruby
i# Heavily inspired by GitLab:
# https://github.com/gitlabhq/gitlabhq/blob/master/config/unicorn.rb.example

worker_processes ENV['WORKER_PROCESSES'].to_i
listen ENV['LISTEN_ON']
timeout 30
preload_app true
GC.respond_to?(:copy_on_write_friendly=) && GC.copy_on_write_friendly = true

check_client_connection false

before_fork do |server, worker|
  defined?(ActiveRecord::Base) && ActiveRecord::Base.connection.disconnect!

  old_pid = "#{server.config[:pid]}.oldbin"
  if old_pid != server.pid
    begin
      sig = (worker.nr + 1) >= server.worker_processes ? :QUIT : :TTOU
      Process.kill(sig, File.read(old_pid).to_i)
    rescue Errno::ENOENT, Errno::ESRCH
    end
  end
end

after_fork do |server, worker|
  defined?(ActiveRecord::Base) && ActiveRecord::Base.establish_connection
end
```

Create a sidekiq initializer `myapp/config/initializers/sidekiq.rb`
```ruby
sidekiq_config = { url: ENV['JOB_WORKER_URL'] }

Sidekiq.configure_server do |config|
  config.redis = sidekiq_config
end

Sidekiq.configure_client do |config|
  config.redis = sidekiq_config
end
```

Make sure we allow connections from ourselves in `myapp/config/environments/development.rb`
```ruby
  ...
 +config.hosts << myapp
```

Create an example environment variable file dot-env.example
```bash
# Replace this value with the output of `rake secret`
# Production value MUST be protected.
SECRET_TOKEN=ASamplePasswordGoesHere

WORKER_PROCESSES=1
LISTEN_ON=0.0.0.0:3000
DATABASE_URL=postgresql://dbuser:dbpass@postgres:5432/myapp?encoding=utf8&pool=5&timeout=5000
CACHE_URL=redis://redis:6379/0
JOB_WORKER_URL=redis://redis:6379/0
```

Edit the `myapp/Dockerfile`. The main things of note here are that we're relying on build stages to separate out some of the garbage
```bash
# Dockerfile development version
ARG RUBY_VERSION=3.1.2
FROM registry.docker.com/library/ruby:$RUBY_VERSION AS base

WORKDIR /rails

FROM base as build


# Install gem and yarn build dependencies
RUN curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg -o /root/yarn-pubkey.gpg && apt-key add /root/yarn-pubkey.gpg
RUN echo "deb https://dl.yarnpkg.com/debian/ stable main" > /etc/apt/sources.list.d/yarn.list
RUN apt-get update && apt-get install -y --no-install-recommends libvips pkg-config nodejs yarn

# Install gems
COPY Gemfile Gemfile.lock ./

RUN bundle install && \
   rm -rf ~/.bundle/ "${BUNDLE_PATH}"/ruby/*/cache "${BUNDLE_PATH}"/ruby/*/bundler/gems/*/.git && \
    bundle exec bootsnap precompile --gemfile

# Copy application code
COPY . .
RUN yarn install

# Precompile bootsnap code for faster boot times
RUN bundle exec bootsnap precompile app/ lib/

FROM base

RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y curl libsqlite3-0 libvips && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

COPY --from=build /usr/local/bundle /usr/local/bundle
COPY --from=build /rails /rails

ENTRYPOINT ["/rails/bin/docker-entrypoint"]

CMD ["bundle", "exec", "unicorn", "-c", "config/unicorn.rb"]
```

Create a `.dockerignore` to avoid things we don't want cluttering the build.
```bash
.git
.dockerignore
.env
workshops/node_modules/
workshops/vendor/bundle/
workshops/tmp/
```

Create a simple reverse-proxy config for nginx
```bash
# reverse-proxy.conf

server {
    listen 8080;
    server_name example.org;

    location / {
        proxy_pass http://myapp:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Create a compose file to bring these pieces together. `nginx` and `redis` can run from images, and our app and sidekiq will run from the same image. At the top level of the directory structure, create `compose.yml`
```yml
services:

  postgres:
    image: postgres:14.2
    environment:
      POSTGRES_USER: wsuser
      POSTGRES_PASSWORD: wspass
    ports: 
      - '5432:5432'
    volumes:
      - myapp-postgres:/var/lib/postgresql/data

  redis:
    image: redis:7.0
    ports:
      - '6379:6379'
    volumes:
      - myapp-redis:/var/lib/redis/data

  app:
    build:
      context: myapp
    # For development, volume mount code
    volumes:
      - ./myapp:/rails
    links:
      - postgres
      - redis
    ports:
      - '3000:3000'
    env_file:
      - .env

  sidekiq:
    build:
      context: myapp
    command: bundle exec sidekiq 
    links:
      - postgres
      - redis
    env_file:
      - .env

  nginx:
    image: nginx:1.25
    volumes:
      - ./reverse-proxy.conf:/etc/nginx/conf.d/reverse-proxy.conf:ro
    links:
      - app
    ports:
      - '8080:8080'

volumes:
  myapp-postgres:
  myapp-redis:
```

Copy the sample environment to `.env`, ready for docker-compose. Without having rake available yet, we will need to generate aour own random value
```bash
$ NEW_SECRET=$(LC_ALL=C tr -dc 'A-Za-z0-9' </dev/urandom | head -c 64; echo)
$ sed "s/SECRET_KEY=.*/SECRET_KEY=${NEW_SECRET}/g" dot-env.example > .env
```

Our app currently doesn't have a `Gemfile.lock` which is required for our build. We can generate one as so
```bash
$ docker run --rm -v "$PWD/myapp":/usr/src/app -w /usr/src/app ruby:3.1.2 bundle install
```

Now we should be ready to build the image
```bash
$ docker compose build
```

For a new app, we need to initialze the database
```bash
$ docker compose run app rake db:reset
$ docker compose run app rake db:migrate
```

That's it for now, commit

## TODO

[ ] Formalize production and development image differences
[ ] Replace unicorn with puma
[ ] Tidy up environment variables - particularly the DB stuff
