require 'rubygems'
require 'json'

class Armature
  METADATA_FILE = "metadata.json"

  ATTRIBUTES = [ "arm", "title", "champion", "organization", "effort", "revision", "requires-arms", "project", "issues", "implementation", "discussion" ]
  REQUIRED_ATTRS = [ "arm", "title", "champion", "revision", "project" ]
  ARRAYS = [ "organization", "requires-arms", "issues" ]

  attr_reader :directory

  def [](name)
    @attributes[name]
  end

  def []=(name, value)
    @attributes[name] = value
  end

  def initialize(directory)
    @attributes = {}

    @directory = directory
  end

  def load
    data = parse()

    data.each do |key, value|
      raise "Invalid Armature attribute '#{key}'" unless ATTRIBUTES.include?(key)

      # Make sure all of our list attributes are in the form of an array.
      if ARRAYS.include?(key)
        value = Array(value)
      end

      self[key] = value
    end
  end

  def metadata_file
      File.join(directory, Armature::METADATA_FILE)
  end

  def parse
    raise "No #{Armature::METADATA_FILE} for #{directory}" unless File.exist?(metadata_file)

    begin
      JSON.parse(File.read(metadata_file))
    rescue => detail
      raise "Could not parse metadata file #{metadata_file} for #{directory}: #{detail}"
    end
  end

  def validate
    raise "No title for #{self.inspect}" unless self["title"]
    raise "No number for #{self.inspect}" unless self["arm"]
    print "Validating #{self['title']} - #{self['arm']}"
    REQUIRED_ATTRS.each do |key|
      raise "No #{key} attribute specified" unless v = self[key] and v != ""
    end
    puts ""
  end
end

task :validate do
  Dir.entries(".").find_all { |d| d =~ /^arm-[\d]+\./ }.each do |d|
    begin
      arm = Armature.new(d)
      arm.load
      arm.validate
    rescue => detail
      puts "Could not parse #{d}: #{detail}"
    end
  end
end

task :gh => [:armsite, :gh_pages] do
end

require 'bundler/setup'
require 'grancher/task'
Grancher::Task.new(name = 'gh_pages') do |g|
  g.branch = 'gh-pages'
#  g.push_to = 'origin'
  g.message = 'Updated armatures'
  g.directory '_armsite'
  g.file 'jekyll/_config.yml', '_config.yml'
end

task :armsite do
  PRESITE="_armsite"
  FileUtils.rm_rf(PRESITE)
  FileUtils.mkdir_p(PRESITE)
  puts "Generating _presite"
  Dir.entries(".").find_all { |d| d =~ /^arm-[\d]+\./ }.each do |d|
    begin
      arm = Armature.new(d)
      arm.load
#      arm.validate
      presite_arm_dir = File.join(PRESITE, d)
      puts "Generating arm in #{presite_arm_dir}"
      Dir.mkdir(presite_arm_dir) unless File.directory?(presite_arm_dir)
      Dir.entries(d).find_all { |f| f =~ /\.md$/ }.each do |f|
        puts "Reading: #{f}"
        File.open(File.join(d, f), "r") do |fh|
          presite_file = File.join(presite_arm_dir, f)
          puts "Generating arm file #{presite_file}"
          File.open(presite_file, "w") do |fh2|
            fh2.puts("---")
            fh2.puts("layout: page")
            fh2.puts("title: #{arm['title']}")
            fh2.puts("arm: #{arm['arm']}")
            fh2.puts("champion: #{arm['champion']}")
            fh2.puts("revision: #{arm['revision']}")
            fh2.puts("project: #{arm['project']}")
            fh2.puts("implementation: #{arm['implementation']}")
            fh2.puts("requires-arms: #{arm['requires-arms']}")
            fh2.puts("issues: #{arm['issues']}")
            fh2.puts("main-page: true") if f =~ /^index\.md$/
            fh2.puts("---")
            fh2.puts
            # copy over the full file, but rewrite internal links
            # to md file to .html files
            fh.readlines.each do |line|
              fh2.puts line.gsub(/\((\w+)\.md\)/, '(\1.html)' )
            end
          end
        end
      end
      # finally copy over all the top level pages
      Dir.entries(".").find_all { |f| f =~ /\.md$/ }.each do |f|
        puts "copying #{f}"
        FileUtils.cp(f, File.join(PRESITE, f))
      end
    rescue => detail
      puts "Could not parse #{d}: #{detail}"
      puts detail.backtrace
    end
  end
end
