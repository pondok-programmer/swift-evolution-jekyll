task :bootstrap do
  exec 'git submodule update --init --recursive'
end

task :update do
  require 'date'
  proposals = Dir.glob('Submodules/swift-evolution/proposals/*.md').sort
  nav_order = 1
  proposals.each do | p |
    file_content = []
    statuses = [
      'Implemented',
      'Rejected',
      'Scheduled',
      'Active Review',
      'Accepted',
      'Deferred'
    ]
      
    title = nil
    proposal = nil
    author = []
    review_manager = []
    status = nil
    scheduled = nil
    implementation = nil
    bug = nil
    
    File.open(p).each_line do | line |      
      if matches = line.match(/^# (.*)$/)
        title = title || matches[1]
      elsif matches = line.match(/Status: .*().*/)
        status = status || matches[1]
      elsif matches = line.match(/Proposal: .*(SE-[0-9]+)/)
        proposal = proposal || matches[1]
      end
      file_content.push(line)
    end
    content = <<-CONTENT
---
layout: default
nav_order: #{nav_order}
status: #{status}
title: "#{proposal} #{title.gsub('"', '\"')}"
author: #{author}
proposal: #{proposal}
author: #{author}
review_manager: #{review_manager}
scheduled: #{scheduled}
implementation: #{implementation}
bug: #{bug}
categories: jekyll update
---

CONTENT

    filename = Date.today.strftime('%F') + '-' + p.split('/').last
    File.open('docs/' + filename, 'w') do |f|
      f.puts content + file_content.join()
    end
    nav_order += 1
  end
end

task :build do
  exec 'bundle exec jekyll build --incremental'
end

task :clean do
  exec 'bundle exec jekyll clean'
end

task :init_search_data do
  exec 'bundle exec just-the-docs rake search:init'
end

task :serve do
  exec 'bundle exec jekyll serve --incremental'
end
