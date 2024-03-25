# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'getoptlong'

# 명령어 라인 옵션을 처리하지 않고, 직접 토큰 값을 설정합니다.
REGISTRATION_TOKEN = '8Ysn7HMSJTB-hp19krgz'

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  # GitLab VM 설정
  config.vm.define "gitlab" do |gitlab|
    gitlab.vm.box = "ubuntu/jammy64"
    gitlab.vm.hostname = "gitlab.local"

    if Vagrant.has_plugin?("vagrant-vbguest")
      gitlab.vbguest.auto_update = false
    end

    gitlab.vm.network "forwarded_port", guest: 80, host: 8080
    gitlab.vm.network "forwarded_port", guest: 22, host: 8022
    gitlab.vm.network "private_network", ip: "192.168.33.44"
    gitlab.vm.synced_folder ".", "/vagrant", type: "rsync", rsync__exclude: [".git/"]

    gitlab.vm.provider "virtualbox" do |vb|
      vb.name = "gitlab.local"
      vb.memory = "4096"
    end

    gitlab.vm.provision "shell", inline: <<-SHELL
      sudo apt-get update
      sudo apt-get install -y curl openssh-server ca-certificates

      debconf-set-selections <<< "postfix postfix/mailname string $HOSTNAME"
      debconf-set-selections <<< "postfix postfix/main_mailer_type string 'Internet Site'"
      DEBIAN_FRONTEND=noninteractive sudo apt-get install -y postfix

      if [ ! -e /vagrant/gitlab-ce.deb ]; then
          wget --content-disposition -O /vagrant/gitlab-ce.deb https://packages.gitlab.com/gitlab/gitlab-ce/packages/ubuntu/jammy/gitlab-ce_16.8.0-ce.0_amd64.deb/download.deb
          sudo dpkg -i /vagrant/gitlab-ce.deb
      fi

      sudo gitlab-ctl reconfigure
    SHELL
  end

  # GitLab Runner VM 설정
  config.vm.define "runner" do |runner|
    runner.vm.box = "ubuntu/jammy64"
    runner.vm.hostname = "ci-runner"

    runner.vm.network "private_network", ip: "192.168.33.45"

    runner.vm.provision "shell", inline: <<-SHELL
      # CPU 아키텍처에 따라 다른 GitLab Runner 바이너리를 다운로드
      ARCH=$(uname -m)
      if [ "$ARCH" = "x86_64" ]; then
        sudo curl -L --output /usr/local/bin/gitlab-runner "https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64"
      elif [ "$ARCH" = "aarch64" ]; then
        sudo curl -L --output /usr/local/bin/gitlab-runner "https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-arm64"
      else
        echo "Unsupported architecture: $ARCH"
        exit 1
      fi

      sudo chmod +x /usr/local/bin/gitlab-runner 
      sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash
      sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
      sudo gitlab-runner start
      
      # GitLab Runner를 GitLab 서버에 등록
      sudo gitlab-runner register \
        --non-interactive \
        --url "http://192.168.33.44/" \
        --registration-token "8Ysn7HMSJTB-hp19krgz" \
        --executor "shell" \
        --description "shell-runner" \
        --maintenance-note "Free-form maintainer notes about this runner" \
        --tag-list "shell,aws" \
        --run-untagged="true" \
        --locked="false" \
        --access-level="not_protected"
    SHELL
  end
end
