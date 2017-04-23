# encoding: utf-8
#!/usr/bin/env ruby

require 'html-proofer'

task :default => [:test]

task :test do
    HTMLProofer.new("./_site", {:allow_hash_href => true, :disable_external => true, :empty_alt_ignore => true, :extension => ".html"}).run
end
