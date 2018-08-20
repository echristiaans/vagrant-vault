# -*- mode: ruby -*-
# vi: set ft=ruby :

$consul_srv = <<CONFIG
cat <<EOF > /etc/consul/server/config.json
{
  "bind_addr": "$1",
  "acl_datacenter": "vagrant",
  "ui": true,
  "domain": "test",
  "datacenter": "vagrant",
  "data_dir": "/var/lib/consul",
  "encrypt": "Owpx3FUSQPGswEAeIhcrFQ==",
  "log_level": "INFO",
  "enable_syslog": true,
  "enable_debug": true,
  "server": true,
  "bootstrap_expect": 3,
  "leave_on_terminate": false,
  "skip_leave_on_interrupt": true,
  "rejoin_after_leave": true,
  "retry_join": [
    "192.168.33.10",
    "192.168.33.20",
    "192.168.33.30"
  ]
}
EOF
CONFIG

$consul_bootstrap = <<CONFIG
cat <<EOF > /etc/consul/bootstrap/config.json
{
  "acl_datacenter": "vagrant",
  "bind_addr": "$1",
  "bootstrap_expect": 3,
  "server": true,
  "datacenter": "vagrant",
  "data_dir": "/var/lib/consul",
  "disable_anonymous_signature": true,
  "disable_remote_exec": true,
  "encrypt": "Owpx3FUSQPGswEAeIhcrFQ==",
  "log_level": "DEBUG",
  "enable_syslog": true
}
EOF
CONFIG

$ca_script = <<SCRIPT
update-ca-trust enable
cp /etc/pki/tls/private/vault.crt /etc/pki/ca-trust/source/anchors/
update-ca-trust extract

systemctl daemon-reload
systemctl enable consul
systemctl enable vault
systemctl restart consul
systemctl restart vault

SCRIPT

Vagrant.configure(2) do |config|
  config.vm.box = "bento/centos-7.5"
  config.vm.box_check_update = true

  # config.vm.network "forwarded_port", guest: 80, host: 8080
  # config.vm.network "public_network"
  # config.vm.synced_folder "../data", "/vagrant_data"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "1024"
  end

  config.vm.provision "shell", inline: <<-SHELL
    cd /root/

    yum update -y #&& yum upgrade -y
    yum install -y curl unzip epel-release yum-utils nginx vim ca-certificates jq

    mkdir -p /etc/vault /etc/consul/server /etc/consul/bootstrap /etc/consul-template
    mkdir -p /var/lib/consul
    #mkdir -p /opt/consul-ui

    if [[ ! -f /usr/local/bin/consul ]]; then
      curl -L -o consul.zip https://releases.hashicorp.com/consul/1.2.2/consul_1.2.2_linux_amd64.zip
      unzip consul.zip && mv consul /usr/local/bin && rm consul.zip

      #curl -L -o consul-ui.zip https://dl.bintray.com/mitchellh/consul/0.5.2_web_ui.zip
      #unzip consul-ui.zip && mv dist/ /opt/consul-ui && rm consul-ui.zip
    fi

    if [[ ! -f /usr/local/bin/vault ]]; then
      curl -L -o vault.zip https://releases.hashicorp.com/vault/0.10.4/vault_0.10.4_linux_amd64.zip
      unzip vault.zip && mv vault /usr/local/bin && rm vault.zip
    fi

    if [[ ! -f /usr/local/bin/consul-template ]]; then
      curl -L -o consul-template.zip https://releases.hashicorp.com/consul-template/0.19.5/consul-template_0.19.5_linux_amd64.zip
      unzip consul-template.zip && mv consul-template /usr/local/bin && rm -rf consul-template.zip
    fi

    chmod +x /usr/local/bin/*
    chgrp -R root /usr/local/bin
    chown root /usr/local/bin/*

    if [[ ! $(grep '/usr/local/bin' /root/.bash_profile) ]]; then
      echo 'export PATH=$PATH:/usr/local/bin' >> /root/.bash_profile
      echo 'export PATH=$PATH:/usr/local/bin' >> /vagrant/.bash_profile
      echo 'export VAULT_ADDR="https://$(hostname):8200"' >> /vagrant/.bash_profile
      echo 'export VAULT_ADDR="https://$(hostname):8200"' >> /root/.bash_profile
      echo 'alias l="ls -lah"' >> /root/.bash_profile
      echo 'alias l="ls -lah"' >> /vagrant/.bash_profile
    fi

    cp '/vagrant/vault.hcl' '/etc/vault/vault.hcl'
    #cp '/vagrant/consul-bootstrap.json' '/etc/consul/bootstrap/config.json'
    #cp '/vagrant/consul-server.json' '/etc/consul/server/config.json'
    cp '/vagrant/consul.service' '/etc/systemd/system/consul.service'
    cp '/vagrant/vault.service' '/etc/systemd/system/vault.service'
  SHELL

  config.vm.define "vault-01", primary: true do |n1|
    ip = "192.168.33.10"
    n1.vm.hostname = 'vault-01'
    n1.vm.network "private_network", ip: "#{ip}"
    n1.vm.network 'forwarded_port',  guest: 8500, host: 8500
    # kill this manually, after the other two have run, then start consul through systemd:
    n1.vm.provision "shell", inline: <<-SHELL
      [[ -f /etc/consul/server/bind.json ]] || echo '{ "bind_addr": "192.168.33.10" }' >> /etc/consul/server/bind.json
      openssl req -x509 -nodes -days 1825 -newkey rsa:4096 -sha256 -keyout /etc/pki/tls/private/vault.key -out /etc/pki/tls/private/vault.crt -reqexts SAN -extensions SAN -subj "/C=NL/ST=Noord-Holland/L=Driehuis/O=DirtyBit/OU=Engineering/CN=vault-01" -config <(cat /etc/pki/tls/openssl.cnf; printf "[SAN]\nsubjectAltName=DNS:#{ip},DNS:127.0.0.1")
    SHELL
    n1.vm.provision "shell", inline: $consul_srv, :args => [ "#{ip}", 1 ]
    n1.vm.provision "shell", inline: $consul_bootstrap, :args => [ "#{ip}", 1 ]
    n1.vm.provision "shell", inline: $ca_script
  end

  config.vm.define "vault-02" do |n2|
    ip = "192.168.33.20"
    n2.vm.hostname = 'vault-02'
    n2.vm.network "private_network", ip: "#{ip}"
    n2.vm.network 'forwarded_port',  guest: 8500, host: 8501
    n2.vm.provision "shell", inline: <<-SHELL
      [[ -f /etc/consul/server/bind.json ]] || echo '{ "bind_addr": "192.168.33.20" }' >> /etc/consul/server/bind.json
      openssl req -x509 -nodes -days 1825 -newkey rsa:4096 -sha256 -keyout /etc/pki/tls/private/vault.key -out /etc/pki/tls/private/vault.crt -reqexts SAN -extensions SAN -subj "/C=NL/ST=Noord-Holland/L=Driehuis/O=DirtyBit/OU=Engineering/CN=vault-02" -config <(cat /etc/pki/tls/openssl.cnf; printf "[SAN]\nsubjectAltName=DNS:#{ip},DNS:127.0.0.1")
    SHELL
    n2.vm.provision "shell", inline: $consul_srv, :args => [ "#{ip}", 1 ]
    n2.vm.provision "shell", inline: $consul_bootstrap, :args => [ "#{ip}", 1 ]
    n2.vm.provision "shell", inline: $ca_script
  end

  config.vm.define "vault-03" do |n3|
    ip = "192.168.33.30"
    n3.vm.hostname = 'vault-03'
    n3.vm.network "private_network", ip: "#{ip}"
    n3.vm.network 'forwarded_port',  guest: 8500, host: 8502
    # consul agent -config-dir /etc/consul/server &
    n3.vm.provision "shell", inline: <<-SHELL
      [[ -f /etc/consul/server/bind.json ]] || echo '{ "bind_addr": "192.168.33.30" }' >> /etc/consul/server/bind.json
      openssl req -x509 -nodes -days 1825 -newkey rsa:4096 -sha256 -keyout /etc/pki/tls/private/vault.key -out /etc/pki/tls/private/vault.crt -reqexts SAN -extensions SAN -subj "/C=NL/ST=Noord-Holland/L=Driehuis/O=DirtyBit/OU=Engineering/CN=vault-03" -config <(cat /etc/pki/tls/openssl.cnf; printf "[SAN]\nsubjectAltName=DNS:#{ip},DNS:127.0.0.1")
    SHELL
    n3.vm.provision "shell", inline: $consul_srv, :args => [ "#{ip}", 1 ]
    n3.vm.provision "shell", inline: $consul_bootstrap, :args => [ "#{ip}", 1 ]
    n3.vm.provision "shell", inline: $ca_script
  end

  # now you can test it!
  # on vault-01, other terminal:
  #   vault operator init -tls-skip-verify
  #   vault operator unseal -tls-skip-verify
  #   vault operator unseal -tls-skip-verify
  #   vault operator unseal -tls-skip-verify
  #   vault login -tls-skip-verify <root token>
  #   vault token create -tls-skip-verify
  #   curl -k https://127.0.0.1:8200/v1/sys/init should return {"initialized":true}
end
