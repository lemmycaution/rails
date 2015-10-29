require 'rake'
require RAKEVERSION == '0.8.0' ? 'rake/gempackagetask' : 'rubygems/package_task'
$:.unshift File.expand_path('..', __FILE__)
require File.expand_path('../railslts-version/lib/railslts-version', __FILE__)

BRANCH = '3-0-lts'
SUB_PROJECT_PATHS = %w(activesupport activemodel actionpack actionmailer activeresource activerecord railties railslts-version)
ALL_PROJECT_PATHS = ['.', *SUB_PROJECT_PATHS]

fail = lambda { |message|
  STDERR.puts "\e[31m#{message}\e[0m" # red
  exit(1)
}

run = lambda { |command|
  puts "\e[35m#{command}\e[0m" # pink
  result = ENV['DRY_RUN'] || system(command)
  result or fail.call("Failed to execute `#{command}`")
  true
}

rails_gemspec = eval(File.read('rails.gemspec'))
Gem::PackageTask.new(rails_gemspec) do |p|
  p.gem_spec = rails_gemspec
end

namespace :railslts do

  desc 'Run tests for Rails LTS compatibility'
  task :test do

    puts '', "\033[44m#{'activesupport'}\033[0m", ''
    system('cd activesupport && rake test')

    puts '', "\033[44m#{'actionmailer'}\033[0m", ''
    system('cd actionmailer && rake test')

    puts '', "\033[44m#{'actionpack'}\033[0m", ''
    system('cd actionpack && rake test')

    puts '', "\033[44m#{'activemodel'}\033[0m", ''
    system('cd activemodel && rake test')

    puts '', "\033[44m#{'activerecord (mysql)'}\033[0m", ''
    system('cd activerecord && rake test_mysql')

    db_path = '/tmp/lts-test-db'
    FileUtils.mkdir_p(db_path)
    puts '', "\033[44m#{'activerecord (sqlite3)'}\033[0m", ''
    system("cd activerecord && DB_PATH=#{db_path} rake test_sqlite3")

    puts '', "\033[44m#{'activerecord (postgres)'}\033[0m", ''
    system('cd activerecord && rake test_postgresql')

    puts '', "\033[44m#{'activeresource'}\033[0m", ''
    system('cd activeresource && rake test')

    puts '', "\033[44m#{'railties'}\033[0m", ''
    system('cd railties && TMP_PATH=/tmp/lts-test-app rake test')

    puts '', "\033[44m#{'railslts-version'}\033[0m", ''
    system('cd railslts-version && rake test')

  end

  namespace :gems do

    # Clean previous .gem files in pkg/ folder of root and sub-projects
    task :delete do
      ALL_PROJECT_PATHS.each do |project|
        pkg_folder = "#{project}/pkg"
        puts "Emptying packages folder #{pkg_folder}..."
        FileUtils.mkdir_p(pkg_folder)
        run.call("rm -f #{pkg_folder}/*.gem")
      end
    end

    # Call :package task in sub-projects
    task :package_all do
      ALL_PROJECT_PATHS.each do |project|
        run.call("cd #{project} && rake package")
      end
    end

    # Clean up building artifacts left by :package tasks
    task :clean_building_artifacts do
      ALL_PROJECT_PATHS.each do |project|
        pkg_folder = "#{project}/pkg"
        puts "Deleting building artifacts from #{pkg_folder}..."
        run.call("rm -rf #{pkg_folder}/*.tgz") # TGZ
        run.call("rm -rf #{pkg_folder}/*.zip") # ZIP
        run.call("rm -rf #{pkg_folder}/*/")    # Folder
      end
    end

    # Move *.gem packages from sub-projects's pkg to root's pkg for easier releasing
    task :consolidate do
      SUB_PROJECT_PATHS.each do |project|
        pkg_folder = "#{project}/pkg"
        gem_path = "#{pkg_folder}/#{project}-#{RailsLts::VERSION::STRING}.gem"
        puts "Moving .gem from #{gem_path} to pkg ..."
        File.file?(gem_path) or fail.call("Not found: #{gem_path}")
        consolidated_pkg_folder = 'pkg'
        FileUtils.mkdir_p(consolidated_pkg_folder)
        FileUtils.mv(gem_path, consolidated_pkg_folder)
      end
    end

    desc 'Builds *.gem packages for distribution without Git'
    task :build => [:delete, :package_all, :consolidate, :clean_building_artifacts] do
      puts 'Done.'
    end

  end

  desc 'Updates the LICENSE file in individual sub-projects'
  task :update_license do
    require 'date'
    last_change = Date.parse(`git log -1 --format=%cd`)
    ALL_PROJECT_PATHS.each do |project|
      next if project == 'railslts-version' # has no LICENSE file
      license_path = "#{project}/LICENSE"
      puts "Updating license #{license_path}..."
      File.exists?(license_path) or fail.call("Could not find license: #{license_path}")
      license = File.read(license_path)
      license.sub!(/ before(.*?)\./ , " before #{(last_change + 10).strftime("%B %d, %Y")}.") or fail.call("Couldn't find timestamp.")
      File.open(license_path, "w") { |w| w.write(license) }
    end
  end

  namespace :customer do

    task :ensure_ready do
      jobs = [
        "Did you update the version in railslts-version/lib/railslts-version.rb (currently #{RailsLts::VERSION::STRING})?",
        'Did you update the LICENSE files using `rake railslts:update_license?',
        'Did you commit and push your changes, as well as the changes by the Rake tasks mentioned above?',
        'Did you build static gems using `rake railslts:gems:build` (those are not pushed to Git)?',
        'Did you activate key forwarding for *.gems.makandra.de?',
        "We will now publish the Rails LTS #{RailsLts::VERSION::STRING} for customers. Ready?",
      ]

      puts

      jobs.each do |job|
        print "#{job} [y/n] "
        answer = STDIN.gets
        puts
        unless answer.strip == 'y'
          $stderr.puts 'Aborting. Nothing was released.'
          puts
          exit
        end
      end
    end

    task :push_to_git_repo do
      %w[c23 c42 c32 c24].each do |hostname|
        fqdn = "#{hostname}.gems.makandra.de"
        puts "\033[1mUpdating #{fqdn}...\033[0m"
        command = "cd /var/www/railslts && git fetch origin #{BRANCH}:#{BRANCH}"
        run.call "ssh deploy-gems_p@#{fqdn} '#{command}'"
        puts 'Done.'
      end

      puts 'Gems pushed to customer Git repo.'
      puts "Now run `git clone -b #{BRANCH} https://gems.makandra.de/railslts #{BRANCH}-test-checkout`"
      puts 'and make sure your commits are present.'
    end

    task :push_to_gem_server do
      password = `read -s -p "Enter password for railslts-gems-admin.makandra.de: " password; echo $password`.chomp
      server_url = "https://admin:#{password}@railslts-gems-admin.makandra.de"
      gem_paths = Dir.glob['pkg/*.gem']
      gem_paths.size == ALL_PROJECT_PATHS.size or fail.call("Expected #{ALL_PROJECT_PATHS.size} .gem files, but only got #{gem_paths.inspect}")
      gem_paths.each do |gem_path|
        puts "Publishing #{gem_path}"
        run.call("gem push #{gem_path} --host #{server_url}")
      end
    end

    desc "Publish Rails LTS #{RailsLts::VERSION::STRING} for customers"
    task :release => [:ensure_ready, :push_to_git_repo, :push_to_gem_server]

  end

  namespace :community do

    task :push_to_git_repo do
      existing_remotes = `git remote`
      unless existing_remotes.include?('community')
        run.call('git remote add community git@github.com:makandra/rails.git')
      end
      run.call('git fetch community')

      puts 'We will now publish the following changes to GitHub:'
      puts
      run.call("git log --oneline community/#{BRANCH}..HEAD")
      puts

      puts 'Do you want to proceed? [y/n]'
      answer = STDIN.gets
      puts
      unless answer.strip == 'y'
        $stderr.puts 'Aborting. Nothing was released.'
        puts
        exit
      end

      run.call("git push community #{BRANCH}")
      puts 'Gems pushed to community github repo.'
      puts "Check https://github.com/makandra/rails/tree/#{BRANCH} and make sure your commits are present"
    end

    desc "Publish Rails LTS #{RailsLts::VERSION::STRING} for community subscribers"
    task :release => :push_to_git_repo

  end

end
