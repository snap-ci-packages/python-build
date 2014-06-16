require 'rubygems'
require 'bundler/setup'

require 'rake/clean'

distro = nil
fpm_opts = ""

if File.exist?('/etc/system-release') && File.read('/etc/redhat-release') =~ /centos|redhat|fedora|amazon/i
  distro = 'rpm'
  fpm_opts << " --rpm-user root --rpm-group root "
elsif File.exist?('/etc/os-release') && File.read('/etc/os-release') =~ /ubuntu|debian/i
  distro = 'deb'
  fpm_opts << " --deb-user root --deb-group root "
end

unless distro
  $stderr.puts "Don't know what distro I'm running on -- not sure if I can build!"
end

# we do not build 3.1 since it's not going anywhere
%w(2.6.9 2.7.6 3.2.5 3.3.5 3.4.0).each do |version|
  namespace version do
    release = Time.now.utc.strftime('%Y%m%d%H%M%S')

    description_string = %Q{Python is an interpreted, interactive, object-oriented programming language often compared to Tcl, Perl, Scheme or Java. Python includes modules, classes, exceptions, very high level dynamic data types and dynamic typing. Python supports interfaces to many system calls and libraries, as well as to various windowing systems (X11, Motif, Tk, Mac and MFC).}

    jailed_root = File.expand_path('../jailed-root', __FILE__)
    prefix = File.join('/opt/local/python', version)

    CLEAN.include("downloads")
    CLEAN.include("jailed-root")
    CLEAN.include("log")
    CLEAN.include("pkg")

    task :init do
      mkdir_p "log"
      mkdir_p "pkg"
      mkdir_p "src"
      mkdir_p "downloads"
      mkdir_p "jailed-root"
      cmd = 'gpg2 --recv-keys 6A45C816 36580288 7D9DC8D2 18ADD4FF A4135B38 A74B06BF EA5BBD71 ED9D77D5 E6DF025C 6F5E1540 F73C700D'
      sh(cmd) do |ok, res|
        if !ok
          sh(cmd)
        end
      end
    end

    task :download do
      cd 'downloads' do
        sh("curl --fail --location http://www.python.org/ftp/python/#{version}/Python-#{version}.tgz     > Python-#{version}.tgz     2>/dev/null")
        sh("curl --fail --location http://www.python.org/ftp/python/#{version}/Python-#{version}.tgz.asc > Python-#{version}.tgz.asc 2>/dev/null")
        sh("gpg2 --verify Python-#{version}.tgz.asc")
      end
    end

    task :configure do
      cd "src" do
        sh "tar -zxf ../downloads/Python-#{version}.tgz"
        cd "Python-#{version}" do
          configure_flags = ' '
          if File.exist?('/usr/lib/x86_64-linux-gnu')
            configure_flags << "LDFLAGS='-L/usr/lib/x86_64-linux-gnu -L/lib/x86_64-linux-gnu' CFLAGS='-I/usr/include/x86_64-linux-gnu' CPPFLAGS='-I/usr/include/x86_64-linux-gnu' "
          end
          sh "#{configure_flags}./configure --prefix=#{prefix} --with-fpectl --disable-shared --enable-unicode=ucs4 --with-system-ffi > #{File.dirname(__FILE__)}/log/configure.#{version}.log 2>&1"
        end
      end
    end

    task :make do
      num_processors = %x[nproc].chomp.to_i
      num_jobs       = num_processors + 1

      cd "src/Python-#{version}" do
        sh("make -j#{num_jobs} > #{File.dirname(__FILE__)}/log/make.#{version}.log 2>&1")
      end
    end

    task :make_install do
      rm_rf jailed_root
      mkdir_p jailed_root
      cd "src/Python-#{version}" do
        sh("make install DESTDIR=#{jailed_root} > #{File.dirname(__FILE__)}/log/make-install.#{version}.log 2>&1")
      end

      # we have to do this fudging around because some versions of python prefix their binaries with the binprefix :(
      cd("#{jailed_root}/#{prefix}/bin") do

        binprefixes = [
          version.split('.').first(2).join('.'),
          version.split('.').first(1)
        ].flatten

        binfiles = %w(idle@@ python@@ pydoc@@ python@@-config).each do |file_template|
          symlink = file_template.gsub('@@', '')

          if File.exist?(symlink)
            $stderr.puts "Skip creation of #{symlink} because there's already something there"
            next
          end

          potential_binaryprefix = binprefixes.find do |prefix|
            potential_binary_name = file_template.gsub('@@', prefix)
            File.exist?(potential_binary_name)
          end

          unless potential_binaryprefix
            $stderr.puts("Could not find binaries matching #{file_template} skipping ahead")
            next
          end

          filename = file_template.gsub('@@', potential_binaryprefix)

          ln_sf(filename, symlink)
        end
      end

    end

    task :fpm do
      cd "pkg" do
        sh(%Q{
             bundle exec fpm -s dir -t #{distro} --name python-#{version} -a x86_64 --version "#{version}" -C #{jailed_root} --directories #{prefix} --verbose #{fpm_opts} --maintainer snap-ci@thoughtworks.com --vendor snap-ci@thoughtworks.com --url http://snap-ci.com --description "#{description_string}" --iteration #{release} --license 'Python' .
        })
      end
    end
    desc "build and package python-#{version}"
    task :all => [:clean, :init, :download, :configure, :make, :make_install, :fpm]
  end

  task :default => "#{version}:all"
end

desc "build all python"
task :default
