require 'yaml'
require 'fileutils'
require 'rubygems'
require 'bundler'
require 'logger'

require 'rspec/core/rake_task'

require_relative './redmine'

class LocalRakeTask < RSpec::Core::RakeTask
  def spec_command
    "ruby -S bundle exec #{super}"
  end
end

class RMRakeTask < LocalRakeTask
  def run_task(*args)
    Bundler.with_clean_env { Dir.chdir(RM.path) { super }}
  end
end

def force?
  !!ENV['FORCE']
end

RM = Redmine.new

task default: [:install, :update, :compile, :spec]

desc 'Run all specs (alias for spec:all)'
task spec: 'spec:all'

namespace :spec do
  desc 'Run unit, plugin and browser specs'
  task all: [:unit, :plugin, :browser]

  desc 'Run unit specs (Testing isolated dashboard components)'
  LocalRakeTask.new(:unit) do |t|
    t.pattern    = ENV['SPEC'] || 'spec/unit/**/*_spec.rb'
    t.ruby_opts  = '-Ispec/unit -Ilib'
    t.rspec_opts = '--color --backtrace'
    t.rspec_opts << " --seed #{ENV['SEED']}" if ENV['SEED']
  end

  desc 'Run plugin specs (Testing within redmine application)'
  RMRakeTask.new(:plugin) do |t|
    t.pattern    = ENV['SPEC'] || "#{RM.path}/spec/plugin/**/*_spec.rb"
    t.ruby_opts  = "-I#{RM.path}/spec/plugin"
    t.rspec_opts = '--color --backtrace'
    t.rspec_opts << " --seed #{ENV['SEED']}" if ENV['SEED']
  end

  desc 'Run browser specs'
  RMRakeTask.new(:browser) do |t|
    t.pattern    = ENV['SPEC'] || "#{RM.path}/spec/browser/**/*_spec.rb"
    t.ruby_opts  = "-I#{RM.path}/spec/browser"
    t.rspec_opts = '--color --backtrace'
    t.rspec_opts << " --seed #{ENV['SEED']}" if ENV['SEED']
  end
end
task ci: :spec

desc 'Setup project environment (alias for redmine:install)'
task install: %w(redmine:install)

desc 'Update project environment (alias for redmine:update)'
task update: %w(redmine:update)

desc 'Start local redmine server (aliases: `s`)'
task server: :install do |_, args|
  RM.bx %w(rails server), args
end
task s: 'server'

desc <<-DESC.gsub(/^ {2}/, '')
  Cleanup project directory. This removes all installed
  redmines as well as precompiled assets.
DESC
task :clean do
  %w(tmp).each do |dir|
    FileUtils.rm_rf dir if File.directory?(dir)
  end
end

task :compile do
  Redmine.exec %w(make install-deps)
  Redmine.exec %w(make min)
end

namespace :redmine do
  desc <<-DESC.gsub(/^ {4}/, '')
    Download Redmine. That includes exporting SVN tag,
    linking plugin and plugin specs and do necessary
    changed to Redmine\'s Gemfile.
  DESC
  task :download do
    if File.exist?(File.join(RM.path, '.downloaded')) && !force?
      puts "Redmine #{RM.version} already downloaded. "\
           'Use `redmine:clean` or FORCE=1 to force redownloaded.'
    else
      RM.clean
      RM.exec %w(svn export --quiet --force), RM.svn_url, '.'
      RM.exec %w(ln -s), Dir.pwd, 'plugins/redmine_dashboard'
      RM.exec %w(mkdir -p), 'public/plugin_assets'
      RM.exec %w(ln -s), File.join(Dir.pwd, 'assets'),
        'public/plugin_assets/redmine_dashboard_linked'
      RM.exec %w(ln -s), File.join(Dir.pwd, 'spec'), '.'

      # Adjust capybara version requirements as redmine locks to ~> 2.1.0
      # but rspec 3 requires >= 2.2
      RM.exec %w(sed -i -e),
        "s/.*gem [\"']capybara[\"'].*/gem 'capybara', '~> 2.3'/g", 'Gemfile'

      FileUtils.touch File.join(RM.path, '.downloaded')
    end
  end

  desc <<-DESC.gsub(/^ {4}/, '')
    Configure Redmine. A database config will be generated
    using mysql2 gem and the `rdb_development` database for
    the `development` environment. Already existing
    configuration will be preserved except for the `test`
    environment.
  DESC
  task config: :download do
    config = {}
    if File.exist? File.join(RM.path, 'config/database.yml')
      begin
        config = YAML.load_file File.join(RM.path, 'config/database.yml')
      rescue => e
        warn e
      end
    end

    config['test'] = RM.database_config(:test)
    %w(production development).each do |env|
      config[env] = RM.database_config(env) unless config[env]
    end

    File.open(File.join(RM.path, 'config/database.yml'), 'w') do |f|
      f.write YAML.dump config
    end
  end

  desc <<-DESC.gsub(/^ {4}/, '')
    Install Redmine. This task will run `bundle install`,
    generate secret token, create databases as well as
    migrate and prepare them. This task will only run once
    unless forced. Use `update` for updating after new gems
    or database migrations.
  DESC
  task install: :config do
    if File.exist?(File.join(RM.path, '.installed')) && !force?
      puts "Redmine #{RM.version} already installed. Use `redmine:clean` to "\
           'delete redmine and reinstall or FORCE=1 to force install steps.'
    else
      Rake::Task['redmine:bundle'].invoke

      RM.bx %w(rake generate_secret_token)
      RM.bx %w(rake db:create:all)

      Rake::Task['redmine:migrate'].invoke
      Rake::Task['redmine:prepare'].invoke

      FileUtils.touch File.join(RM.path, '.installed')
    end
  end

  desc <<-DESC.gsub(/^ {4}/, '')
    Update Redmine. This runs `bundle install` and migrate
    and prepare databases.
  DESC
  task update: [:install, :bundle, :migrate, :prepare]

  task :bundle do
    tries = 0
    begin
      RM.exec %w(rm -f Gemfile.lock)
      RM.exec %w(bundle install --without rmagick)
    rescue
      STDERR.puts 'bundle install failed. Retry...'
      retry if (tries += 1) < 5
    end
  end

  task :migrate do
    RM.bx %w(rake db:migrate)
    RM.bx %w(rake redmine:plugins:migrate)
  end

  task :prepare do
    RM.bx %w(rake db:test:prepare)
  end

  desc 'Clean redmine directory'
  task :clean do
    RM.clean
  end

  task :exec, [:cmd] do |t, args|
    puts RM.bx args.cmd
  end
end
