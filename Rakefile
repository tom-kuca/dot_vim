# Rakefile
#
# Tasks for managing dot_vim.

require 'open-uri'
require 'openssl'
require 'rubygems'
require 'json'


PLUGIN_LIST_TAG = '## Plugin List'
PLUGIN_LIST_NOTE = 'Generated by `rake update_readme`'
README_FILE = 'README.md'
VUNDLE_PLUGINS_FOLDER = 'vundle_plugins'
LINES_WITHOUT_CONFIG = 4
PLUGINS_HEADER = <<-HEADER.chomp
| Stars___ | **Plugin** | **Description** |
| -------: | :--------- | :-------------- |
HEADER

FILES_TO_LINK = %w{vimrc gvimrc}

task :default => ['vim:link']

namespace :vim do
  desc 'Create symlinks'
  task :link do
    begin
      FILES_TO_LINK.each do |file|
        dot_file = File.expand_path("~/.#{file}")
        if File.exists? dot_file
          puts "#{dot_file} already exists, skipping link."
        else
          File.symlink(".vim/#{file}", dot_file)
          puts "Created link for #{file} in your home folder."
        end
      end
    rescue NotImplementedError
      puts "File.symlink not supported, you must do it manually."
      if RUBY_PLATFORM.downcase =~ /(mingw|win)(32|64)/
        puts 'Windows 7 use mklink, e.g.'
        puts '  mklink _vimrc .vim\vimrc'
      end
    end

  end
end

desc 'Update the list of plugins in README.md'
task :update_readme do
  delete_old_plugins_from_readme
  add_plugins_to_readme(plugins)
end


# ----------------------------------------
# Helper Methods
# ----------------------------------------


# Just takes an array of strings that resolve to plugins from Vundle
def add_plugins_to_readme(plugins = [])
  lines = File.readlines(README_FILE).map{|l| l.chomp}
  index = lines.index(PLUGIN_LIST_TAG)
  unless index.nil?
    lines.insert(index+1, "\n#{PLUGINS_HEADER}")
    plugin_rows = plugins
      .sort {|x,y| 
        x[:stars] <=> y[:stars] or x[:name].downcase <=> y[:name].downcase
      }
      .reverse
      .map{|p| table_row(p) }
    lines.insert(index+2, plugin_rows)
    lines << "\n_That's #{plugins.length} plugins, holy crap._"
    lines << "\n_#{PLUGIN_LIST_NOTE} on #{Time.now.strftime('%Y/%m/%d')}._\n\n"
    write_lines_to_readme(lines)
  else
    puts "Error: Plugin List Tag (#{PLUGIN_LIST_TAG}) not found"
  end
end

def table_row(plugin)
  p = plugin
  [
    "| #{p[:stars_text]} |",
    "[#{p[:name]}](#{p[:uri]})",
    p[:config?] ? " [:page_facing_up:](#{p[:config_file]})" : nil,
    '|',
    "#{p[:description]} |"
  ].compact.join
end

def delete_old_plugins_from_readme
  lines = []
  File.readlines(README_FILE).map do |line|
    line.chomp!
    lines << line
    if line == PLUGIN_LIST_TAG
      break
    end
  end

  write_lines_to_readme(lines)
end

def write_lines_to_readme(lines)
  readme_file = File.open(README_FILE, 'w')
  readme_file << lines.join("\n")
  readme_file.close
end

# Returns an array of plugin files.
def plugin_files
  Dir.glob("./#{VUNDLE_PLUGINS_FOLDER}/**.vim")
end

# Returns an array of hashes containing info for each plugin.
def plugins
  plugins = plugin_files.map do |filename|
    lines = File.new(filename).readlines
    sub_plugins = lines.reduce([]) do |memo, line|
      if line =~ /^\s*Plugin\s+["'](.+)["']/
        plugin_info = fetch_plugin_info($1)
        plugin_info[:config?] = lines.length > LINES_WITHOUT_CONFIG
        memo << plugin_info
      end
      memo
    end
    print '.'
    sub_plugins
  end
  plugins.flatten
end


# Returns a hash of info for a plugin based on it's Vundle link.
def fetch_plugin_info(vundle_link)
  info = {}
  github_user = ''
  github_repo = ''

  if /(?<user>[a-zA-Z0-9\-]*)\/(?<name>[a-zA-Z0-9\-\._]*)/ =~ vundle_link
    github_user = user
    github_repo = name
  else
    github_user = 'vim-scripts'
    github_repo = vundle_link
  end

  info[:user] = github_user
  info[:name] = github_repo
  info[:uri] = "https://github.com/#{github_user}/#{github_repo}"
  info[:config_file_name] =
    /\.vim$/ =~ github_repo ? github_repo : "#{github_repo}.vim"
  info[:config_file] = "#{VUNDLE_PLUGINS_FOLDER}/#{info[:config_file_name]}"

  plugin_info = repo_info(github_user, github_repo)
  info[:description] = plugin_info['description'].strip
  info[:stars] = plugin_info['stargazers_count']
  info[:stars_text] = "#{comma_number(plugin_info['stargazers_count'])} ★"

  info
end

def comma_number(number)
  number.to_s.gsub(/(\d)(?=(\d\d\d)+(?!\d))/, "\\1,")
end

def repo_info(user, name)
  response = ''
  api_url = "https://api.github.com/repos/#{user}/#{name}"

  # Without a GitHub Client / Secret token you will only be able to make 60
  # requests per hour, meaning you can only update the readme once.
  # Read more here http://developer.github.com/v3/#rate-limiting.
  if ENV['GITHUB_CLIENT_ID'] and ENV['GITHUB_CLIENT_SECRET']
    api_url += "?client_id=#{ENV['GITHUB_CLIENT_ID']}&client_secret=#{ENV['GITHUB_CLIENT_SECRET']}"
  end

  begin
    if RUBY_VERSION < '1.9'
      response = open(api_url).read
    else
      response = open(api_url, :ssl_verify_mode => OpenSSL::SSL::VERIFY_NONE).read
    end
  rescue Exception => e
    message = "Problem fetching #{user}/#{name}."
    puts message + e.message
    return {'description' => message}
  end

  JSON.parse(response)
end
