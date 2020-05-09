require 'tmpdir'

task :default do
  STDERR.puts <<~EOI
    use:
      rake kernel/build/VERSION
      rake image/build/NAME
  EOI
end

desc 'clone linux kernel sources'
file 'kernel/source' do
  sh 'git clone https://github.com/torvalds/linux.git kernel/source'
end

rule %r{^kernel/build/v.*$} => ->(o) { "kernel/config/#{File.basename(o)}" } do |task|

  # invoked explicitly here to have only one dependency
  # on the `rule` in which case it behaves like a `file`
  # task for the task name
  Task['kernel/source'].invoke

  version = File.basename(task.name)

  Dir.chdir('kernel/source') do
    sh <<~EOSH
      git clean -f
      make clean
      make mrproper
    EOSH

    cp "../config/#{version}", '.config'

    sh <<~EOFSH
      git checkout #{version}
      make -j 8 vmlinux
    EOFSH

    cp 'vmlinux', "../build/#{version}"
  end
end

rule %r{^image/build/.*$} => ->(o) { "image/docker/#{File.basename(o)}" } do |task|
  name       = File.basename(task.name)
  image_name = "#{File.basename(__dir__)}-#{name}"

  Dir.mktmpdir do |tmp|
    sh "docker build -f image/docker/#{name} -t #{image_name} #{tmp}"
  end

  Dir.mktmpdir do |tmp|
    sh <<~EOSH
      dd if=/dev/zero of=#{task.name} bs=1M count=256
      mkfs.ext4 #{task.name}
      sudo mount #{task.name} #{tmp}

      docker container create --name=#{image_name} #{image_name}
      docker container export #{image_name} | sudo tar -xf - -C #{tmp}
      docker container rm #{image_name}

      sudo umount #{tmp}
    EOSH
  end
end
