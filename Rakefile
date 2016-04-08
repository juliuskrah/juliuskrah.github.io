# encoding: utf-8

require "html/proofer"

task :default => [:test]

task :test do
    HTML::Proofer.new("./_site", {
        :allow_hash_href => true,
	:disable_external => true,
	:empty_alt_ignore => true,
	:ext => ".html"
    }).run
end
