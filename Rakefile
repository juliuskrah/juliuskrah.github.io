# encoding: utf-8

require "html/proofer"

task :default => [:test]

task :test do
    HTML::Proofer.new("./_site", {
        :href_ignore => [
	    "#",
	    # The additional anchor link is picked up from the Geomap JSON, but shouldn't be flagged
	    "\\\"#\\\""
	],
	:disable_external => true,
	:empty_alt_ignore => true,
	:ext => ".html"
    }).run
end
