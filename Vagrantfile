MACHINES = {
  :"pam" => {
    :box_name => "ubuntu-26.04",
    :cpus => 2,
    :memory => 2048,
    :ip => "192.168.57.10",
  }
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    config.vm.synced_folder ".", "/vagrant", disabled: true
    config.vm.network "private_network", ip: boxconfig[:ip]

    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxname.to_s

      box.vm.provider "virtualbox" do |v|
        v.memory = boxconfig[:memory]
        v.cpus = boxconfig[:cpus]
        v.customize ["modifyvm", :id, "--audio", "none"]
        v.customize ["modifyvm", :id, "--graphicscontroller", "vmsvga"]
        v.customize ["modifyvm", :id, "--accelerate3d", "off"]
        v.customize ["modifyvm", :id, "--vram", "32"]
        v.customize ["modifyvm", :id, "--nested-hw-virt", "off"]
      end

      box.vm.provision "shell", inline: <<-SHELL
        # Аутентификация по паролю в SSH
        sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
        # В Ubuntu также нужно проверить cloud-init конфиги, если они переопределяют SSH
        if [ -d /etc/ssh/sshd_config.d ]; then
          sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config.d/*.conf 2>/dev/null || true
        fi
        systemctl restart ssh

        # Создаём пользователей и группу admin
        groupadd -f admin
        useradd -m -s /bin/bash otusadm
        useradd -m -s /bin/bash otus

        echo "otusadm:Otus2022!" | chpasswd
        echo "otus:Otus2022!" | chpasswd

        usermod -aG admin otusadm
        usermod -aG admin root
        usermod -aG admin vagrant

        # Создаём файл праздничных дней для их исключения
        cat << 'EOF' > /etc/holidays.txt
01-01
01-02
01-03
01-04
01-05
01-06
01-07
01-08
02-23
03-08
05-01
05-09
06-12
11-04
EOF

        # Создаём PAM-скрипт проверки доступа
        cat << 'EOF' > /usr/local/bin/login.sh
#!/bin/bash

HOLIDAYS_FILE="/etc/holidays.txt"
TODAY_MD=$(date +%m-%d)
DAY_OF_WEEK=$(date +%u)

if [ -f "$HOLIDAYS_FILE" ] && grep -qx "$TODAY_MD" "$HOLIDAYS_FILE"; then
    exit 0
fi

if [ "$DAY_OF_WEEK" -eq 6 ] || [ "$DAY_OF_WEEK" -eq 7 ]; then
    if id -nG "$PAM_USER" | grep -qw "admin"; then
        exit 0
    else
        exit 1
    fi
fi

exit 0
EOF

        chmod +x /usr/local/bin/login.sh

        # 5. Добавляем правило в PAM для SSH (в секцию account)
        if ! grep -q "/usr/local/bin/login.sh" /etc/pam.d/sshd; then
          sed -i '/@include common-account/a account required pam_exec.so /usr/local/bin/login.sh' /etc/pam.d/sshd
        fi
      SHELL
    end
  end
end
