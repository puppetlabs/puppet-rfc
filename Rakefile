require 'rubygems'
require 'json'

class Armature
  METADATA_FILE = "metadata.json"

  ATTRIBUTES = [ "arm", "title", "champion", "organization", "effort", "revision", "requires-arms", "project", "issues", "implementation" ]
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
  Dir.entries(".").find_all { |d| d =~ /^arm-[\d+]\./ }.each do |d|
    begin
      arm = Armature.new(d)
      arm.load
      arm.validate
    rescue => detail
      puts "Could not parse #{d}: #{detail}"
    end
  end
end
