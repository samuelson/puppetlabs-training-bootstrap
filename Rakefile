require 'open-uri'
require 'yaml'
require 'net/http'
require 'r10k/puppetfile'

PRE_RELEASE = ENV['PRE_RELEASE'] == 'true'
PTB_VERSION = YAML.load_file('./build_files/version.yaml')
STABLE = ENV['STABLE'] || 'false'
DEBUG = ENV['DEBUG'] || false

FILESHARE_SERVER = '//guest@int-resources.ops.puppetlabs.net/Resources'
STDOUT.sync = true

##############################################################
#                                                            #
# Methods to abstract the environment variable CL interface. #
#                                                            #
##############################################################

def pe_family
  if !ENV['PE_VERSION'] || ENV['PE_VERSION'] == 'latest'
    if ENV['PE_FAMILY']
      return ENV['PE_FAMILY']
    elsif !PRE_RELEASE
      return nil
    else
      puts "You must set the PE_VERSION or PE_FAMILY environment variable to build a pre-release version"
      exit 1
    end
  else
    ENV["PE_VERSION"].split('.')[0 .. 1].join('.')
  end
end

def pe_version
  if !ENV['PE_VERSION'] || ENV['PE_VERSION'] == 'latest'
    PRE_RELEASE ? latest_pre_version(pe_family) : latest_release_version
  else
    ENV['PE_VERSION']
  end
end

########################################
#                                      #
# Helper methods for installer caching #
#                                      #             
########################################

# At some point we might modify this method to specify dist, release, and arch
def installer_filename
  if PRE_RELEASE
    "puppet-enterprise-#{pe_version}-el-7-x86_64.tar"
  else
    "puppet-enterprise-#{pe_version}-el-7-x86_64.tar.gz"
  end
end

# Note that because we gzip the pre-release installer, this is always .tar.gz
def cached_installer_path
  "./file_cache/installers/puppet-enterprise-#{pe_version}-el-7-x86_64.tar.gz"
end

# Get the latest pre-release version string for a given PE Version family.
def latest_pre_version(pe_family)
  open("http://getpe.delivery.puppetlabs.net/latest/#{pe_family}").read
end

def latest_release_version
  redirect_response = Net::HTTP.get_response(URI.parse("https://pm.puppetlabs.com/cgi-bin/download.cgi\?dist\=el\&rel\=7\&arch\=x86_64\&ver\=latest"))
    .header['location']
  return redirect_response.match(/(?<version>\d{4}\.\d+\.\d+)/).to_s
end

# Public releases are .tar.gz, while internal releases are just tar.
# When cacheing a pre-release, gzip it.
def pe_installer_url
  if PRE_RELEASE
    "http://enterprise.delivery.puppetlabs.net/#{pe_family}/ci-ready/#{installer_filename}"
  else
    "https://s3.amazonaws.com/pe-builds/released/#{pe_version}/#{installer_filename}"
  end
end

# Check if the installer exists and the file isn't empty.
def already_cached?
  File.exist?(cached_installer_path) and File.zero?(cached_installer_path) == false
end

def gzip_installer
  `gzip ./file_cache/installers/#{installer_filename}`
end

# Download the installer. Curl is nicer than re-inventing the wheel with ruby.
def download_installer
  puts pe_installer_url
  unless `curl #{pe_installer_url} -o ./file_cache/installers/#{installer_filename}`
    fail "Error downloading the PE installer"
  end
  gzip_installer if PRE_RELEASE
end

def cache_directories
  ["./file_cache/gems", "./file_cache/installers", "./output", "./packer_cache"]
end

def cache_directories_exist?
  cache_directories.each{ |dir| return false unless File.exist?(dir) }
  true
end

def create_cache_directories
  FileUtils.mkdir_p(cache_directories)
end

def get_base_vm(image_type)
	if image_type == "student" then
		image_box="centos-6.6-i386-virtualbox-nocm-1.0.3.box"
	else
		image_box="centos-7.2-x86_64-virtualbox-nocm-1.0.1.box"
	end
	vagrant_base_url="http://int-resources.ops.puppetlabs.net/Vagrant%20images"
  #Public download URL: https://atlas.hashicorp.com/puppetlabs/boxes/centos-7.2-64-nocm/versions/1.0.1/providers/virtualbox_desktop.box
  #Public download URL: https://atlas.hashicorp.com/puppetlabs/boxes/centos-6.6-32-nocm/versions/1.0.3/providers/virtualbox_desktop.box

  `rm -rf output/#{image_type}-base-virtualbox`
  `mkdir output/#{image_type}-base-virtualbox`

  puts "Downloading #{image_type} base image"
  `curl #{vagrant_base_url}/#{image_box} -o output/#{image_type}-base-virtualbox/#{image_box}`
  `cd output/#{image_type}-base-virtualbox/; tar xzvf ../#{image_box}`
  `mv output/#{image_type}-base-virtualbox/*.ovf output/#{image_type}-base-virtualbox/#{image_type}-base.ovf`
end

###################################
#                                 #
# Helper methods for building VMs #
#                                 #             
###################################

def subprocess_trap(io)
  trap('INT') do
    Process.kill('INT', io.pid)
    while (line = io.gets) do
      puts line
    end
    exit
  end
end

def call_packer(template, args={}, var_file=nil)
  arg_string = DEBUG ? "-debug " : ""
  args.each do |k, v|
    arg_string << " -var '#{k}=#{v}' "
  end
  arg_string << " -var-file=#{var_file} " if var_file
  # Call packer with a -f flag to remove any existing builds.
  # Pass everything through to STDOUT live and pass SIGINT through
  packer_io = IO.popen("packer build -force #{arg_string} #{template}") do |io|
    subprocess_trap(io)
    while (line = io.gets) do
      puts line
    end
  end
end

def template_dir
  './templates'
end

def template_file(build_type)
  case build_type
  when 'base'
    File.join(template_dir, 'educationbase.json')
  when 'build'
    File.join(template_dir, 'educationbuild.json')
  when 'student'
    File.join(template_dir, 'student.json')
  when 'ami'
    File.join(template_dir, 'awsbuild.json')
  else
    fail "ERROR: Invalid build type: #{build_type}"
  end
end

def var_file(vm_name)
  if vm_name == "student"
    nil
  else
    File.join(template_dir, "#{vm_name}.json")
  end
end

# This method isn't currently used.
def output_dir(vm_name, build_type)
  unless build_type == 'base'
    File.join('./output/', "#{vm_name}-virtualbox")
  else
    File.join('./output/', "#{vm_name}-base-virtualbox")
  end
end

def packer_args
  {
    'pe_version'  => pe_version,
    'pe_family'   => pe_family,
    'pre_release' => PRE_RELEASE,
    'ptb_version' => "#{PTB_VERSION[:major]}.#{PTB_VERSION[:minor]}",
    'stable'      => ENV['STABLE']
  }
end

def box_to_ova(vm_name)
  box_name = "puppet-#{pe_version}-#{vm_name}-#{PTB_VERSION[:major]}.#{PTB_VERSION[:minor]}"
  puts %x{mkdir -p output/temp}
  puts %x{cd output/temp; tar xvf ../#{box_name}.box}
  open('output/temp/box.ovf','r') do |f|
    open('output/temp/box.ovf.temp','w') do |f2|
      f.each_line do |line|
        # Remove mac address for vbox support
        line.sub!(/MACAddress=\"[0-9A-F]*\"\s/,'')
        # Remove VM hardware to support vbox and vmware
        f2.write(line) unless line.match(/VirtualSystemType/)
      end
    end
  end
  FileUtils.mv 'output/temp/box.ovf.temp', 'output/temp/box.ovf'
  puts %x{cd output/temp; tar cvf #{box_name}.ova *.ovf *.vmdk *.json Vagrantfile}
  puts %x{mv output/temp/#{box_name}.ova output}
  puts %x{rm -rf output/temp}
  puts %x{rm -f output/#{box_name}.box}
end

def build_vm(build_type, vm_name)
  validate_build_details
  call_packer(template_file(build_type), packer_args, var_file(vm_name))
end

def symlink_latest(vm_name)
  `cd output; ln -sf puppet-#{pe_version}-#{vm_name}-#{PTB_VERSION[:major]}.#{PTB_VERSION[:minor]} #{vm_name}_latest.ova`
end

def validate_build_details
  puts "\nPE version: #{pe_version}\n"
  name = STABLE == 'true' ? "Puppetfile.stable" : "Puppetfile"
  basedir = File.expand_path(File.join(File.dirname(__FILE__), '/build_files'))
  puppetfile_path = File.join(basedir, name)
  puppetfile = R10K::Puppetfile.new(basedir, module_dir = nil, puppetfile_path = puppetfile_path)
  puppetfile.load
  puts "\nThe following modules will be included in the Puppetfile for this build:\n\n"
  puppetfile.modules.each do | m |
    puts "#{m.name.ljust(20)}#{m.instance_variable_get(:@args)}"
  end
  unless ENV['AUTOMATED_BUILD'] == 'true'
    puts "Do you wish to begin the build with these modules? Y/n"
    raise "Cancelled" if [ 'n', 'no' ].include? STDIN.gets.strip.downcase
  end
end

############################################
#                                          #
# Helper methods for Learning VM packaging #
#                                          #             
############################################

def ova_name
  "puppet-#{pe_version}-learning-#{PTB_VERSION[:major]}.#{PTB_VERSION[:minor]}.ova"
end

def make_learning_vm_dir
  `rm -rf /tmp/learning_puppet_vm && mkdir /tmp/learning_puppet_vm`
end

def copy_ova_to_dir
  `cp ./output/#{ova_name} /tmp/learning_puppet_vm/#{ova_name}`
end

def strip_version_include(string)
  string.split("\n")[1..-1].join("\n")
end

def troubleshooting_string
  open('https://raw.githubusercontent.com/puppetlabs/puppet-quest-guide/master/troubleshooting.md')
    .read
end

def setup_string
  open('https://raw.githubusercontent.com/puppetlabs/puppet-quest-guide/master/SETUP.md')
    .read
end

def readme_markdown
  strip_version_include(setup_string) + "\n" + strip_version_include(troubleshooting_string)
end

def readme_rtf
  PandocRuby.new(readme_markdown, :standalone).to_rtf
end

def write_readme
  File.write('/tmp/learning_puppet_vm/readme.rtf', readme_rtf)
end

def vm_path(vm_type)
  if vm_type == 'learning'
    './output/learning_puppet_vm.zip'
  else
    "./output/puppet-#{pe_version}-#{vm_type}-#{PTB_VERSION[:major]}.#{PTB_VERSION[:minor]}.ova"
  end
end

def zip_learning_vm
  puts "Compressing Learning VM..."
  `zip -jrds 100  #{vm_path("learning")} /tmp/learning_puppet_vm/`
end

def create_md5(vm_type)
  if `uname` =~ /Darwin/
    `md5 #{vm_path(vm_type)} > #{vm_path(vm_type) + ".md5"}`
  else
    `md5sum #{vm_path(vm_type)} > #{vm_path(vm_type) + ".md5"}`
  end

end

def bundle_learning_vm
  make_learning_vm_dir
  copy_ova_to_dir
  write_readme
  zip_learning_vm
  create_md5("learning")
end

def mount_fileshare
  begin
    `mkdir -p /tmp/fileshare && mount_smbfs #{FILESHARE_SERVER} /tmp/fileshare`
  rescue
    puts "There was an error mounting the #{FILESHARE_SERVER} to /tmp/fileshare"
    puts "Pleause check that this server isn't already mounted"
    exit 1
  end
end

def unmount_fileshare
  `umount /tmp/fileshare`
end

def ship_to_fileshare(path, destination)
  FileUtils.cp(path, destination)
  FileUtils.chmod(0644, File.join(destination, File.basename(path)))
end

def ship_directory
  "/tmp/fileshare/EducationVMs/learning/puppet-#{pe_version}-learning-#{PTB_VERSION[:major]}.#{PTB_VERSION[:minor]}/"
end



def update_symlink
  Dir.chdir(ship_directory) do
    if PRE_RELEASE
      `ln -sf learning_puppet_vm.zip ../learning_beta.zip`
      `ln -sf learning_puppet_vm.zip.md5 ../learning_beta.zip.md5`
    else
      `ln -sf learning_puppet_vm.zip ../learning_latest.zip`
      `ln -sf learning_puppet_vm.zip.md5 ../learning_latest.zip.md5`
    end 
  end
end

def ship_vm_files(vm_type)
  mount_fileshare
  `mkdir -p #{ship_directory}`
  ship_to_fileshare(vm_path(vm_type), ship_directory)
  ship_to_fileshare(vm_path(vm_type) + ".md5", ship_directory)
  if vm_type == "learning"
    update_symlink
  end
  unmount_fileshare
end

#################################################
#                                               #
# Rake tasks for prepping the local environment #
#                                               #             
#################################################

desc "Set up default cache dirctories"
task :set_up_cache_dirs do
  create_cache_directories unless cache_directories_exist?
end

# TODO Add a task to set up symlinks for file_cache, output, and packer_cache
# then mkdir_p for file_cache/gems and file_cache/installers
# Environment variables for FILE_CACHE_SRC, OUTPUT_SRC, PACKER_CACHE_SRC

desc "Cache the PE Installer"
task :cache_pe_installer do
  cached_installer_path = "./file_cache/installers/#{installer_filename}"
  if not already_cached? 
    puts "Downloading PE installer for #{pe_version}"
    download_installer
  else
    puts "PE Installer for #{pe_version} is already cached. Moving on..."
  end
end

desc "Setup build environment"
task :setup => [:set_up_cache_dirs] do
	get_base_vm "education"
	get_base_vm "student"
end

##################
#                #
# VM Build tasks #
#                #             
##################

desc "Training VM base build"
task :training_base => [:cache_pe_installer]do
  build_vm('base', 'training')
end

desc "Training VM build"
task :training_build do
  build_vm('build', 'training')
  box_to_ova('training')
  create_md5("training")
end

desc "Training AMI build"
task :training_ami do
  build_vm('ami', 'training')
end

desc "Master VM base build"
task :master_base => [:cache_pe_installer] do
  build_vm('base', 'master')
end

desc "Master VM build"
task :master_build do
  build_vm('build', 'master')
  box_to_ova('master')
  symlink_latest('master')
  create_md5("master")
end

desc "Master AMI build"
task :master_ami do
  build_vm('ami', 'master')
end

desc "Learning VM base build"
task :learning_base => [:cache_pe_installer] do
  build_vm('base', 'learning')
end

desc "Learning VM build"
task :learning_build do
  build_vm('build', 'learning')
  box_to_ova('learning')
end

desc "Demo VM base build"
task :demo_base => [:cache_pe_installer] do
  build_vm('base', 'demo')
end

desc "Demo VM build"
task :demo_build do
  build_vm('build', 'demo')
  box_to_ova('demo')
end

desc "Student VM build"
task :student_build do
  build_vm('student', 'student')
  box_to_ova('student')
  create_md5("student")
end

desc "Package learning VM"
task :package_learning do
  begin
    require 'pandoc-ruby'
  rescue LoadError
    puts "You must have pandoc installed for the package Learning VM task!"
    exit 1
  end
  bundle_learning_vm
end

desc "Ship Learning VM"
task :ship_learning do
  ship_vm_files("learning")
end

desc "Ship Master VM"
task :ship_master do
  ship_vm_files("master")
end

desc "Ship Training VM"
task :ship_training do
  ship_vm_files("training")
end

desc "Ship Student VM"
task :ship_student do
  ship_vm_files("student")
end

desc "Create PR to release branch"
task :release do
  `hub pull-request -h puppetlabs/education-builds:master -b puppetlabs/education-builds:release`
end

