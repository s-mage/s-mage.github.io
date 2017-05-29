require 'rake'
require 'image_optim'

POSTS_DIR = File.join('.', '_posts')

# Based on: https://github.com/plusjade/jekyll-bootstrap/blob/master/Rakefile
# Usage: rake post title="A Title" [date="2012-02-09"] [tags=[tag1,tag2]] [category="category"]
desc "Begin a new post in #{POSTS_DIR}"
task :post do
  title = ENV["title"] || "new-post"
  description = ENV["description"] || ""
  slug = title.downcase.strip.gsub(' ', '-').gsub(/[^\w-]/, '')
  begin
    date = (ENV['date'] ? Time.parse(ENV['date']) : Time.now).strftime('%Y-%m-%d')
  rescue
    puts "Error - date format must be YYYY-MM-DD, please check you typed it correctly!"
    exit(-1)
  end
  filename = File.join(POSTS_DIR, "#{date}-#{slug}.md")
  if File.exist?(filename) && ask("#{filename} already exists. Overwrite?", ['y', 'n']) == 'n'
    abort("rake aborted!")
  end

  puts "Creating new post: #{filename}"
  open(filename, 'w') do |post|
    post.puts "---"
    post.puts "layout: post"
    post.puts "title: \"#{title.gsub(/-/,' ')}\""
    post.puts "description: \"#{description}\""
    post.puts "date: #{date}"
    post.puts "---"
  end
end

desc 'Run jekyll serve -w --drafts'
task :preview do
  system 'jekyll serve -w --drafts'
end

desc 'Optimize images (specify a directory with dir=./dir)'
task :image_optim do
  image_optim = ImageOptim.new(pngout: false, svgo: false, verbose: true)
  path = File.join('images', (ENV['dir'] ? ENV['dir'] : '**'), '*.*')
  image_optim.optimize_images!(Dir[path])
  puts 'Done'
end
