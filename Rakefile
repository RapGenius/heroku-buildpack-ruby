require "fileutils"
require "tmpdir"

S3_BUCKET_NAME  = "heroku-buildpack-ruby"
VENDOR_URL      = "https://s3.amazonaws.com/#{S3_BUCKET_NAME}"

def s3_tools_dir
  File.expand_path("../support/s3", __FILE__)
end

def s3_upload(tmpdir, name)
  sh("#{s3_tools_dir}/s3 put #{S3_BUCKET_NAME} #{name}.tgz #{tmpdir}/#{name}.tgz")
end

def vendor_plugin(git_url, branch = nil)
  name = File.basename(git_url, File.extname(git_url))
  Dir.mktmpdir("#{name}-") do |tmpdir|
    FileUtils.rm_rf("#{tmpdir}/*")

    Dir.chdir(tmpdir) do
      sh "git clone #{git_url} ."
      sh "git checkout origin/#{branch}" if branch
      FileUtils.rm_rf("#{name}/.git")
      sh("tar czvf #{tmpdir}/#{name}.tgz *")
      s3_upload(tmpdir, name)
    end
  end
end

def in_gem_env(gem_home, &block)
  old_gem_home = ENV['GEM_HOME']
  old_gem_path = ENV['GEM_PATH']
  ENV['GEM_HOME'] = ENV['GEM_PATH'] = gem_home.to_s

  yield

  ENV['GEM_HOME'] = old_gem_home
  ENV['GEM_PATH'] = old_gem_path
end

desc "install ree"
task "ree:install", :version do |t, args|
  full_version   = args[:version] || '1.8.7-2012.02'
  full_name      = "ruby-enterprise-#{full_version}"
  version        = '1.8.7'
  major_ruby     = '1.8'
  rubygems       = '1.8.24'
  name           = "ruby-#{version}"
  usr_dir        = "usr"
  rubygems_dir   = "#{full_name}/rubygems"
  Dir.mktmpdir("ruby-") do |tmpdir|
    Dir.chdir(tmpdir) do |dir|
      FileUtils.rm_rf("#{tmpdir}/*")

      sh "curl https://rubyenterpriseedition.googlecode.com/files/#{full_name}.tar.gz -s -o - | tar zxf -"

      Dir.chdir("#{full_name}/source") do |source_dir|
        sh "curl -o 34ba44f94a62c63ddf02a045b6f4edcd6eab4989.patch https://github.com/Genius/rubyenterpriseedition187-330/commit/34ba44f94a62c63ddf02a045b6f4edcd6eab4989.patch"
        sh "curl -o 5384967a015be227e16af7a332a50d45e14ed0ad.patch https://github.com/Genius/rubyenterpriseedition187-330/commit/5384967a015be227e16af7a332a50d45e14ed0ad.patch"
        sh "curl -o tcmalloc_declare_memalign_volatile.patch https://raw.githubusercontent.com/skottler/rubyenterpriseedition187-330/tcmalloc-patch/patches/tcmalloc_declare_memalign_volatile.patch"
        sh "patch -p1 < 34ba44f94a62c63ddf02a045b6f4edcd6eab4989.patch"
        sh "patch -p1 < 5384967a015be227e16af7a332a50d45e14ed0ad.patch"
        sh "patch -p1 < tcmalloc_declare_memalign_volatile.patch"
      end

      # build ruby
      if major_ruby == "1.8"
        output  = "ruby-build-#{version}"
        prefix  = "/tmp/ruby-#{version}"
          build_command = [
            "mv #{full_name} #{prefix}",
            "cd #{prefix}",
            "./installer --auto /app/vendors/#{output} --no-dev-docs --no-tcmalloc",
          ]
          build_command = build_command.join(" && ")

          sh build_command
      end
    end
  end
end

def install_gem(gem_name, version)
  name = "#{gem_name}-#{version}"
  Dir.mktmpdir("#{gem_name}-#{version}") do |tmpdir|
    Dir.chdir(tmpdir) do |dir|
      in_gem_env(tmpdir) do
        sh("unset RUBYOPT; gem install #{gem_name} --version #{version} --no-ri --no-rdoc --env-shebang")
        sh("rm #{gem_name}-#{version}.gem")
        sh("rm -rf cache/#{gem_name}-#{version}.gem")
        sh("tar czvf #{tmpdir}/#{name}.tgz *")
        s3_upload(tmpdir, name)
      end
    end
  end
end

desc "update plugins"
task "plugins:update" do
  vendor_plugin "http://github.com/heroku/rails_log_stdout.git", "legacy"
  vendor_plugin "http://github.com/pedro/rails3_serve_static_assets.git"
  vendor_plugin "http://github.com/hone/rails31_enable_runtime_asset_compilation.git"
end

desc "install vendored gem"
task "gem:install", :gem, :version do |t, args|
  gem     = args[:gem]
  version = args[:version]

  install_gem(gem, version)
end

desc "generate ruby versions manifest"
task "ruby:manifest" do
  require 'rexml/document'
  require 'yaml'

  document = REXML::Document.new(`curl https://#{S3_BUCKET_NAME}.s3.amazonaws.com`)
  rubies   = document.elements.to_a("//Contents/Key").map {|node| node.text }.select {|text| text.match(/^(ruby|rbx|jruby)-\\\\d+\\\\.\\\\d+\\\\.\\\\d+(-p\\\\d+)?/) }

  Dir.mktmpdir("ruby_versions-") do |tmpdir|
    name = 'ruby_versions.yml'
    File.open(name, 'w') {|file| file.puts(rubies.to_yaml) }
    sh("#{s3_tools_dir}/s3 put #{S3_BUCKET_NAME} #{name} #{name}")
  end
end

namespace :buildpack do
  require 'netrc'
  require 'excon'
  require 'json'
  require 'time'
  require 'cgi'
  require 'git'
  require 'fileutils'
  require 'digest/md5'
  require 'securerandom'

  def connection
    @connection ||= begin
      user, password = Netrc.read["api.heroku.com"]
      Excon.new("https://#{CGI.escape(user)}:#{password}@buildkits.herokuapp.com")
    end
  end

  def latest_release
    @latest_release ||= begin
      buildpack_name = "heroku/ruby"
      response = connection.get(path: "buildpacks/#{buildpack_name}/revisions")
      releases = JSON.parse(response.body)

      # {
      #   "tar_link": "https://codon-buildpacks.s3.amazonaws.com/buildpacks/heroku/ruby-v84.tgz",
      #   "created_at": "2013-11-06T18:55:04Z",
      #   "published_by": "richard@heroku.com",
      #   "id": 84
      #  }
      releases.map! do |a|
        a["created_at"] = Time.parse(a["created_at"])
        a
      end.sort! { |a,b| b["created_at"] <=> a["created_at"] }
      releases.first
    end
  end

  def new_version
    @new_version ||= "v#{latest_release["id"] + 1}"
  end

  def git
    @git ||= Git.open(".")
  end

  desc "increment buildpack version"
  task :increment do
    version_file = './lib/language_pack/version'
    require './lib/language_pack'
    require version_file

    if LanguagePack::Base::BUILDPACK_VERSION != new_version
      stashes = nil

      if git.status.changed.any?
        stashes = Git::Stashes.new(git)
        stashes.save("WIP")
      end

      File.open("#{version_file}.rb", 'w') do |file|
      file.puts <<-FILE
require "language_pack/base"

# This file is automatically generated by rake
module LanguagePack
  class LanguagePack::Base
    BUILDPACK_VERSION = "#{new_version}"
  end
end
FILE
      end

      git.add "#{version_file}.rb"
      git.commit "bump to #{new_version}"

      stashes.pop if stashes

      puts "Bumped to #{new_version}"
    else
      puts "Already on #{new_version}"
    end
  end

  def changelog_entry?
    File.read("./CHANGELOG.md").split("\n").any? {|line| line.match(/^## #{new_version}/) }
  end

  desc "check if there's a changelog for the new version"
  task :changelog do
    if changelog_entry?
      puts "Changelog for #{new_version} exists"
    else
      puts "Please add a changelog entry for #{new_version}"
    end
  end

  def github_remote
    @github_remote ||= git.remotes.detect {|remote| remote.url.match(%r{heroku/heroku-buildpack-ruby.git$}) }

  end

  def git_push_master
    puts "Pushing master"
    git.push(github_remote, 'master', false)
    $?.success?
  end

  desc "update master branch"
  task :git_push_master do
    git_push_master
  end

  desc "stage a tarball of the buildpack"
  task :stage do
    Dir.mktmpdir("heroku-buildpack-ruby") do |tmpdir|
      Git.clone(File.expand_path("."), 'heroku-buildpack-ruby', path: tmpdir)
      Dir.chdir(tmpdir) do |dir|
        streamer = lambda do |chunk, remaining_bytes, total_bytes|
          File.open("ruby.tgz", "w") {|file| file.print(chunk) }
        end
        Excon.get(latest_release["tar_link"], :response_block => streamer)
        Dir.chdir("heroku-buildpack-ruby") do |dir|
          sh "tar xzf ../ruby.tgz .env"
          sh "tar czf ../buildpack.tgz * .env"
        end

        @digest = Digest::MD5.hexdigest(File.read("buildpack.tgz"))
      end

      filename = "buildpacks/#{@digest}.tgz"
      puts "Writing to #{filename}"
      FileUtils.mkdir_p("buildpacks/")
      FileUtils.cp("#{tmpdir}/buildpack.tgz", filename)
      FileUtils.cp("#{tmpdir}/buildpack.tgz", "buildpacks/buildpack.tgz")
    end
  end

  def multipart_form_data(buildpack_file_path)
    body = ''
    boundary = SecureRandom.hex(4)
    data = File.open(buildpack_file_path)

    data.binmode if data.respond_to?(:binmode)
    data.pos = 0 if data.respond_to?(:pos=)

    body << "--#{boundary}" << Excon::CR_NL
    body << %{Content-Disposition: form-data; name="buildpack"; filename="#{File.basename(buildpack_file_path)}"} << Excon::CR_NL
    body << 'Content-Type: application/x-gtar' << Excon::CR_NL
    body << Excon::CR_NL
    body << File.read(buildpack_file_path)
    body << Excon::CR_NL
    body << "--#{boundary}--" << Excon::CR_NL

    {
      :headers => { 'Content-Type' => %{multipart/form-data; boundary="#{boundary}"} },
      :body => body
    }
  end

  task :publish do
    buildpack_name = "heroku/ruby"
    puts "Publishing #{buildpack_name} buildpack"
    resp = connection.post(multipart_form_data("buildpacks/buildpack.tgz").merge(path: "/buildpacks/#{buildpack_name}"))
    puts resp.status
    puts resp.body
  end

  desc "tag a release"
  task :tag do
    tagged_version =
      if @new_version.nil?
        "v#{latest_release["id"]}"
      else
        new_version
      end

    git.add_tag(tagged_version)
    puts "Created tag #{tagged_version}"

    puts "Pushing tag to remote #{github_remote}"
    git.push(github_remote, nil, true)
  end

  desc "release a new version of the buildpack"
  task :release do
    Rake::Task["buildpack:increment"].invoke
    raise "Please add a changelog entry for #{new_version}" unless changelog_entry?
    raise "Can't push to master" unless git_push_master
    Rake::Task["buildpack:stage"].invoke
    Rake::Task["buildpack:publish"].invoke
    Rake::Task["buildpack:tag"].invoke
  end
end

begin
  require 'rspec/core/rake_task'

  desc "Run specs"
  RSpec::Core::RakeTask.new(:spec) do |t|
    t.rspec_opts = %w(-fs --color)
    #t.ruby_opts  = %w(-w)
  end
  task :default => :spec
rescue LoadError => e
end
