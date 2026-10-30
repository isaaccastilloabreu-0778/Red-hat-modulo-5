# Red-hat-modulo-5
comandos del modulo 5
##########################################################
# PRACTICA 1: SINCRONIZACION DE CARPETAS CON RSYNC
##########################################################

# ----- AMBOS SERVIDORES -----
sudo dnf install rsync cronie
sudo systemctl enable --now crond

# ----- SERVIDOR SECUNDARIO -----
mkdir ~/sync_destino

# ----- SERVIDOR PRIMARIO -----
mkdir ~/sync_origen
cd ~/sync_origen
touch archivo{1..100}.txt
ls | wc -l
cd ~

# Llave SSH sin contraseña (para que cron pueda sincronizar)
ssh-keygen -t rsa -b 4096
ssh-copy-id usuario@IP_SECUNDARIO
ssh usuario@IP_SECUNDARIO

# Primera sincronizacion manual
rsync -avz ~/sync_origen/ usuario@IP_SECUNDARIO:~/sync_destino/

# Script de sincronizacion
nano ~/sync.sh
#!/bin/bash
rsync -avz --delete ~/sync_origen/ usuario@IP_SECUNDARIO:~/sync_destino/ >> ~/sync.log 2>&1

chmod +x ~/sync.sh
~/sync.sh

# Crontab cada 1 minuto
crontab -e
* * * * * /home/usuario/sync.sh
crontab -l

# Prueba: crear archivo/carpeta nuevo en el primario
touch ~/sync_origen/prueba_cron.txt
mkdir ~/sync_origen/carpeta_nueva
cat ~/sync.log

# ----- SERVIDOR SECUNDARIO (validar tras 1 minuto) -----
ls ~/sync_destino
ls ~/sync_destino | wc -l


##########################################################
# PRACTICA 2: CLUSTER PACEMAKER + COROSYNC (2 NODOS, IP FLOTANTE)
##########################################################

# ----- AMBOS NODOS -----
sudo hostnamectl set-hostname nodo1        # en el nodo 2: nodo2
sudo nano /etc/hosts
IP_NODO1   nodo1
IP_NODO2   nodo2

sudo subscription-manager repos --enable rhel-10-for-$(arch)-highavailability-rpms
sudo dnf install pcs pacemaker corosync fence-agents-all
sudo systemctl enable --now pcsd
sudo firewall-cmd --permanent --add-service=high-availability
sudo firewall-cmd --reload
sudo passwd hacluster

# ----- SOLO NODO 1 -----
sudo pcs host auth nodo1 nodo2 -u hacluster
sudo pcs cluster setup micluster nodo1 nodo2
sudo pcs cluster start --all
sudo pcs cluster enable --all
sudo pcs property set stonith-enabled=false
sudo pcs property set no-quorum-policy=ignore
sudo pcs resource create IP_Flotante ocf:heartbeat:IPaddr2 ip=IP_FLOTANTE cidr_netmask=24 op monitor interval=10s
sudo pcs status
sudo pcs cluster status
sudo corosync-cfgtool -s

# ----- PRUEBAS (desde un cliente / tercera maquina) -----
ping IP_FLOTANTE

# ----- EN EL NODO QUE TIENE LA IP -----
sudo pcs status
ip a
sudo reboot

# ----- EN EL OTRO NODO (validar el cambio) -----
sudo pcs status
ip a

# Alternar: reiniciar el otro nodo y repetir
sudo reboot

# Alternativa sin reiniciar
sudo pcs node standby nodo1
sudo pcs status
sudo pcs node unstandby nodo1


##########################################################
# PRACTICA 3: ALTA DISPONIBILIDAD HTTP CON KEEPALIVED
##########################################################

# ----- AMBOS SERVIDORES -----
sudo dnf install httpd keepalived
sudo systemctl enable --now httpd
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-protocol=vrrp
sudo firewall-cmd --reload
ip a

# ----- SERVIDOR 1 -----
echo "<h1>SERVER1</h1>" | sudo tee /var/www/html/index.html

sudo cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
sudo nano /etc/keepalived/keepalived.conf
global_defs {
    router_id SERVER1
}
vrrp_script chk_httpd {
    script "/usr/bin/systemctl is-active httpd"
    interval 2
    weight -20
}
vrrp_instance VI_1 {
    state MASTER
    interface ens160
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass clave123
    }
    virtual_ipaddress {
        IP_VIRTUAL/24
    }
    track_script {
        chk_httpd
    }
}

# ----- SERVIDOR 2 -----
echo "<h1>SERVER2</h1>" | sudo tee /var/www/html/index.html

sudo cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
sudo nano /etc/keepalived/keepalived.conf
global_defs {
    router_id SERVER2
}
vrrp_script chk_httpd {
    script "/usr/bin/systemctl is-active httpd"
    interval 2
    weight -20
}
vrrp_instance VI_1 {
    state BACKUP
    interface ens160
    virtual_router_id 51
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass clave123
    }
    virtual_ipaddress {
        IP_VIRTUAL/24
    }
    track_script {
        chk_httpd
    }
}

# ----- AMBOS SERVIDORES -----
sudo keepalived -t
sudo systemctl enable --now keepalived
sudo systemctl status keepalived

# ----- SERVIDOR 1 (verificar IP virtual) -----
ip a | grep IP_VIRTUAL

# ----- CLIENTE (navegador o terminal) -----
curl http://IP_VIRTUAL
# navegador: http://IP_VIRTUAL  (debe decir SERVER1)

# ----- PRUEBA DE FALLO: apagar el servidor 1 -----
sudo poweroff
# alternativa sin apagar:
sudo systemctl stop keepalived

# ----- SERVIDOR 2 -----
ip a | grep IP_VIRTUAL
sudo journalctl -u keepalived | tail -20

# ----- CLIENTE -----
curl http://IP_VIRTUAL
# navegador: http://IP_VIRTUAL  (ahora debe decir SERVER2)

# ----- RESTAURAR -----
sudo systemctl start keepalived        # (o encender el servidor 1)
curl http://IP_VIRTUAL                 # vuelve a SERVER1
