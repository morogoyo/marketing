Vagrant.configure("2") do |config|

  # Specify the base box
  config.vm.box = "ubuntu/focal64"

  # Configure VM resources: 8 CPUs and 10GB RAM
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "10240"  # 10GB RAM
    vb.cpus = 8          # 8 CPUs
  end

  # Ensure the host "playbooks" directory exists
  if !File.directory?("./playbooks")
    Dir.mkdir("./playbooks")
  end

  # Sync the "playbooks" directory to the VM
  config.vm.synced_folder "./playbooks", "/vagrant/playbooks"

  # Provision the VM
  config.vm.provision "shell", inline: <<-SHELL
    # Update the package list and upgrade installed packages
    sudo apt-get update
    sudo apt-get upgrade -y

    # Install Docker
    sudo apt-get install -y docker.io
    sudo systemctl start docker
    sudo systemctl enable docker
    sudo usermod -aG docker vagrant

    # Install Docker Compose
    sudo curl -L "https://github.com/docker/compose/releases/download/2.2.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    sudo chmod +x /usr/local/bin/docker-compose

    # Create a Docker volume for Ansible files
    docker volume create ansible_files

    # Pull the Ansible Docker image
    docker pull cytopia/ansible

    # Check if the Ansible container is already running
    if [ ! "$(docker ps -q -f name=ansible)" ]; then
      # If the container does not exist, create and run it
      if [ "$(docker ps -aq -f name=ansible)" ]; then
        # If a stopped container exists, remove it
        docker rm ansible
      fi
      docker run -d \
        --name ansible \
        -v ansible_files:/data \                 # Mount the Docker volume
        -v /vagrant/playbooks:/playbooks \      # Sync playbooks directory
        -v /var/run/docker.sock:/var/run/docker.sock \  # Mount Docker socket
        cytopia/ansible tail -f /dev/null
    fi
  SHELL

  # Create a systemd service to ensure the Ansible container starts on boot
  config.vm.provision "shell", inline: <<-SHELL
    echo "[Unit]
    Description=Ansible Container
    Requires=docker.service
    After=docker.service

    [Service]
    Restart=always
    ExecStart=/usr/bin/docker start -a ansible
    ExecStop=/usr/bin/docker stop ansible

    [Install]
    WantedBy=multi-user.target" | sudo tee /etc/systemd/system/ansible.service

    # Reload systemd, enable the service, and start it
    sudo systemctl daemon-reload
    sudo systemctl enable ansible.service
    sudo systemctl start ansible.service
  SHELL

end
