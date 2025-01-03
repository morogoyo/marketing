Vagrant.configure("2") do |config|
  # Specify the base box
  config.vm.box = "ubuntu/focal64"

  # Configure VM resources
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "10240"  # 10GB RAM
    vb.cpus = 8          # 8 CPUs
  end

  # Ensure the "playbooks" directory exists on the host
  if !File.directory?("./playbooks")
    Dir.mkdir("./playbooks") # Create the directory if it doesn't exist
  end

  # Sync the "playbooks" directory to the Vagrant box
  config.vm.synced_folder "./playbooks", "/vagrant/playbooks"

  # Provision the Vagrant box
  config.vm.provision "shell", inline: <<-SHELL
    # Update system packages
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

    # Create a Docker volume for Ansible data
    docker volume create ansible_data

    # Pull the Ansible Docker image
    docker pull cytopia/ansible

    # Check if the Ansible container is already running
    if [ ! "$(docker ps -q -f name=ansible)" ]; then
      # If the container exists but is stopped, remove it
      if [ "$(docker ps -aq -f name=ansible)" ]; then
        docker rm ansible
      fi
      # Create and run the Ansible Docker container
      docker run -d \
        --name ansible \
        -v ansible_data:/vagrant/ansible_data \
        -v /vagrant/playbooks:/playbooks \
        -v /var/run/docker.sock:/var/run/docker.sock \
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
