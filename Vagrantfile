GO_VERSION = "1.24.5"
BOX_VERSION = "202510.26.0"

Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-24.04"
  config.vm.box_version = BOX_VERSION
  config.vm.hostname = "quicknotes-lab5"
  config.vm.box_check_update = false

  config.vm.network "forwarded_port",
    guest: 8080,
    host: 18080,
    host_ip: "127.0.0.1",
    auto_correct: false

  config.vm.synced_folder "./app", "/srv/quicknotes/app", type: "virtualbox"

  config.vm.provider "virtualbox" do |vb|
    vb.name = "quicknotes-lab5"
    vb.memory = 1024
    vb.cpus = 2
    vb.gui = false
  end

  config.vm.provision "shell", privileged: true, env: { "GO_VERSION" => GO_VERSION }, inline: <<-'SHELL'
    set -euxo pipefail

    export DEBIAN_FRONTEND=noninteractive
    apt-get update
    apt-get install -y curl ca-certificates

    guest_arch="$(dpkg --print-architecture)"
    case "$guest_arch" in
      amd64) go_arch="amd64" ;;
      arm64) go_arch="arm64" ;;
      *)
        echo "Unsupported guest architecture: $guest_arch" >&2
        exit 1
        ;;
    esac

    wanted_go="go${GO_VERSION}"
    current_go=""
    if [ -x /usr/local/go/bin/go ]; then
      current_go="$(/usr/local/go/bin/go version | awk '{print $3}')"
    fi

    if [ "$current_go" != "$wanted_go" ]; then
      tarball="${wanted_go}.linux-${go_arch}.tar.gz"
      curl -fsSL "https://go.dev/dl/${tarball}" -o "/tmp/${tarball}"
      rm -rf /usr/local/go
      tar -C /usr/local -xzf "/tmp/${tarball}"
      rm -f "/tmp/${tarball}"
    fi

    cat >/etc/profile.d/go.sh <<'EOF'
export PATH=/usr/local/go/bin:$PATH
EOF
    chmod 0644 /etc/profile.d/go.sh
    ln -sf /usr/local/go/bin/go /usr/local/bin/go

    install -d -o vagrant -g vagrant /home/vagrant/quicknotes-data
    install -d -o vagrant -g vagrant /srv/quicknotes/app

    sudo -u vagrant env PATH="/usr/local/go/bin:$PATH" sh -lc \
      'cd /srv/quicknotes/app && /usr/local/go/bin/go build -o /tmp/quicknotes .'
    install -m 0755 /tmp/quicknotes /usr/local/bin/quicknotes
    rm -f /tmp/quicknotes

    cat >/etc/systemd/system/quicknotes.service <<'EOF'
[Unit]
Description=QuickNotes
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=vagrant
Group=vagrant
WorkingDirectory=/srv/quicknotes/app
Environment=ADDR=:8080
Environment=DATA_PATH=/home/vagrant/quicknotes-data/notes.json
Environment=SEED_PATH=/srv/quicknotes/app/seed.json
ExecStart=/usr/local/bin/quicknotes
Restart=always
RestartSec=2

[Install]
WantedBy=multi-user.target
EOF

    systemctl daemon-reload
    systemctl enable --now quicknotes
    systemctl restart quicknotes
  SHELL
end
