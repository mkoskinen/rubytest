source "https://rubygems.org"

ruby file: ".ruby-version"

gem "rails", "~> 8.0"

# Download-heavy: lots of pure-Ruby sub-gems
gem "aws-sdk-s3"
gem "stripe"
gem "sentry-rails"

# Compile-heavy: native extensions
gem "nokogiri"
gem "pg"
gem "bcrypt"
gem "image_processing"
gem "ffi"
gem "grpc"

# Realistic Rails app shape
gem "devise"
gem "sidekiq"
gem "redis"

group :development, :test do
  gem "rspec-rails"
  gem "factory_bot_rails"
  gem "faker"
end
