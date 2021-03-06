# frozen_string_literal: true

require 'rubygems'
require 'bundler'
require 'bundler/gem_tasks'

Bundler.setup(:default, :test)

require 'rubocop/rake_task'

RuboCop::RakeTask.new

task default: [:rubocop, :spec]

require 'rspec/core'
require 'rspec/core/rake_task'
RSpec::Core::RakeTask.new(:spec) do |spec|
  pattern = ARGV[1] || 'spec/**/*_spec.rb'
  spec.pattern = FileList[pattern]
  spec.verbose = false
end

desc "builds a gem"
task build: :update_asset_version do
  `gem build rack-mini-profiler.gemspec 1>&2`
end

desc "compile sass"
task compile_sass: :copy_files do
  require "sassc"
  scss = File.read("lib/html/includes.scss")
  css = SassC::Engine.new(scss).render
  File.write("lib/html/includes.css", css)
end

desc "update asset version file"
task update_asset_version: [:compile_sass, :write_vendor_js] do
  require 'digest/md5'
  h = []
  Dir.glob('lib/html/*.{js,html,css,tmpl}').each do |f|
    h << Digest::MD5.hexdigest(::File.read(f))
  end
  File.open('lib/mini_profiler/asset_version.rb', 'w') do |f|
    f.write \
"# frozen_string_literal: true
module Rack
  class MiniProfiler
    ASSET_VERSION = '#{Digest::MD5.hexdigest(h.sort.join(''))}'
  end
end\n"
  end
end

@mini_racer_context = nil
desc "generate vendor asset file"
task :write_vendor_js do
  require 'mini_racer'
  require 'nokogiri'

  dot_js = File.read(File.expand_path("../lib/html/dot.1.1.2.min.js", __FILE__))
  html = File.read(File.expand_path("../lib/html/includes.tmpl", __FILE__))

  templates = {}
  Nokogiri::HTML(html).css('script[type="text/x-dot-tmpl"]').each do |node|
    id = node["id"]
    raise "Each template must have a unique id" if !id || id.size == 0 || templates.key?(id)
    templates[id] = node.content
  end

  @mini_racer_context ||= MiniRacer::Context.new
  @mini_racer_context.eval(dot_js)
  templates_js = "MiniProfiler.templates = {};\n"

  templates.each do |k, v|
    template = v.gsub('`', '\\`')
    compiled = @mini_racer_context.eval <<~JS
      doT.compile(`#{template}`).toString()
    JS
    templates_js += <<~JS
      MiniProfiler.templates["#{k}"] = #{compiled}
    JS
  end

  pretty_print = File.read(File.expand_path("../lib/html/pretty-print.js", __FILE__))
  content = <<~JS
    /**
      THIS FILE IS AUTOMATICALLY GENERATED BY THE `write_vendor_js` RAKE TASK.
      DON'T EDIT THIS FILE BY HAND; CHANGES WILL BE OVERRIDEN.
    **/

    "use strict";
    #{templates_js}
    #{pretty_print}
    MiniProfiler.loadedVendor = true;
  JS
  path = File.expand_path("../lib/html/vendor.js", __FILE__)
  FileUtils.touch(path)
  File.write(path, content)
end

desc "Start Sinatra server for client-side development"
task :client_dev do
  require 'listen'

  regexp = /(vendor\.js|includes\.css)$/
  listener = Listen.to(File.expand_path("lib/html", __dir__)) do |modified|
    next if modified.all? { |m| m =~ regexp }
    print("Assets change detected; updating ASSET_VERSION constant and recompiling templates... ")
    rake_task = Rake.application[:update_asset_version]
    rake_task.all_prerequisite_tasks.each(&:reenable)
    rake_task.reenable
    rake_task.invoke
    puts "Done"
  rescue => err
    puts "\nError occurred: #{err.inspect}"
  end
  listener.start
  pid = spawn("cd website && BUNDLE_GEMFILE=Gemfile bundle exec rackup")
  Process.wait(pid)
rescue Interrupt
  listener.stop
end

desc "copy files from other parts of the tree"
task :copy_files do
  # TODO grab files from MiniProfiler/UI
end
