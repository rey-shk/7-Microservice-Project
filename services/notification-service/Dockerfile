FROM ruby:3.3-slim

WORKDIR /app

# Install native dependencies required for pg gem compilation
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
 && rm -rf /var/lib/apt/lists/*

# Install ruby gems
COPY Gemfile Gemfile.lock ./
RUN bundle install

# Copy application source
COPY . .

EXPOSE 8086

CMD ["bundle", "exec", "ruby", "app.rb"]
