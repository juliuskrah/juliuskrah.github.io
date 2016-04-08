#!/usr/bin/env ruby

require 'html/proofer'
HTML::Proofer.new("./_site", {
    :href_ignore => [
	    "#",
		# The additional anchor link is picked up from the Geomap JSON, but shouldn't be flagged
		"\\\"#\\\""
    ],
	:disable_external => true,
	:ext => ".html"
}).run
