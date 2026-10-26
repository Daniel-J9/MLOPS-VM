Vagrant.configure("2") do |config|
  # Wybieramy gotowy, oficjalny obraz systemu Ubuntu Linux 24.04
  config.vm.box = "bento/ubuntu-24.04"

  # INTERFEJS 2 (Sieć wewnętrzna na maszynie gospodarza / Host-Only):
  # Przypisujemy maszynie adres 192.168.56.10, dostępny TYLKO z Twojego Windowsa.
  config.vm.network "private_network", ip: "192.168.56.10"

  # KONFIGURACJA ZASOBÓW VIRTUALBOX (4 GB RAM i 2 rdzenie CPU dla modelu LLM):
  config.vm.provider "virtualbox" do |vb|
    vb.gui = false
    vb.memory = "4096"
    vb.cpus = 2
  end

  # Automatyczne włączenie logowania hasłem po SSH oraz instalacja Git i curl
  config.vm.provision "shell", inline: <<-SHELL
    export DEBIAN_FRONTEND=noninteractive
    sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
    rm -f /etc/ssh/sshd_config.d/*.conf
    systemctl restart ssh

    apt-get update -y
    apt-get install -y git curl
  SHELL
end