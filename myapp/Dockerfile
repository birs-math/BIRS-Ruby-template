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
