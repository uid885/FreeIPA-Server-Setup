#Author: Christo Deale <br>
#Date: 2024-02-22

# Setup FreeIPA Server on RHEL9.2

## Step 1: Install FreeIPA Packages
```
sudo dnf install ipa-server ipa-server-dns
```

## Step 2: Configure FreeIPA Server
```
sudo ipa-server-install --setup-dns --no-forwarders
```
```
Hostname: srv-01.example.co.za      #FQDN here
Directory Manager password:         #Your password here
IPA admin password:                 #Your IPA admin password here
DNS forwarder: yes
DNS reverse zone: yes
NetBIOS domain name: srv-01         #Domain Name here
```

## Step 3: Configure Chrony NTP Servers
**3.1** Install & enable Chrony
```
sudo dnf install chrony
sudo systemctl enable --now chronyd
sudo sed -i 's/pool 2\.fedora\.pool\.ntp\.org iburst/\
               server 0.za.pool.ntp.org iburst\n\
               server 1.za.pool.ntp.org iburst\n\
               server 2.za.pool.ntp.org iburst\n\
               server 3.za.pool.ntp.org iburst/' /etc/chrony.conf
sudo systemctl restart chronyd
```
**3.2** Verify Chrony Configuration
```
chronyc sources
ntpstat
```

## Step 4: Get Kerberos Ticet
```
kinit admin
echo "Kerberos Ticket:" > /klist.txt
klist >> /klist.txt
```

## Step 5: Add User Account
```
sudo ipa user-add novadmin --first=Nov --last=Admin --password
```

## Step 6: Allow Firewall Services
```
sudo firewall-cmd --permanent --add-service={freeipa-ldap,freeipa-ldaps,dns,ntp}
sudo firewall-cmd --reload
```

## Step 7: FreeIPA Replication Services
```
sudo ipa-replica-prepare srv-02.novnet.co.za
sudo scp /var/lib/ipa/replica-info-srv-02.example.co.za.gpg \
   root@192.168.0.21:/var/lib/ipa/
sudo ipa-replica-install --setup-ca --setup-dns --forwarder=192.168.0.20 \
   /var/lib/ipa/replica-info-srv-02.novnet.co.za.gpg
```

## Step 8: Client Installation
```
sudo dnf install ipa-client
sudo ipa-client-install --server=srv-01.example.co.za --domain=example.co.za
   Client IP: 192.168.0.50
```
## Step 9: Add DNS Entry for Client (on FreeIPA Server)
```
sudo ipa dnsrecord-add novnet.co.za client --a-rec=192.168.0.50
```

# Client configuration

## Step 1: Set DNS to FreeIPA Server
```
sudo sed -i 's/^DNS=.*$/DNS=192.168.0.20/' /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
```

## Step 2: Configure client with FreeIPA Server
```
sudo dnf install ipa-client
sudo ipa-client-install --server=srv-01.novnet.co.za --domain=example.co.za
```

## Step 3: Create Home Directory at initial login
```
sudo dnf install oddjob-mkhomedir
```

## Step 4: Verify Client Configuration
**4.1** Check to see if Client can communicate with FreeIPA server
```
kinit admin
echo "Kerberos Ticket:" > /klist_client.txt
klist >> /klist_client.txt
```
**4.2** Ensure that you can retrieve user information from FreeIPA server
```
id novadmin
```
**4.3** Test ssh login with FreeIPA Server
```
ssh novadmin@192.168.0.50
```
**4.4** Verify that the client has received DNS information from FreeIPA
```
ipa-client-install --unattended --force
```
**4.5** Ensure time synchronization between client and FreeIPA server is accurate
```
chronyc sources
ntpstat
```

# Configure the System Security Services Daemon (sssd)

## Step 1: Backup existing configuration
```
sudo cp /etc/sssd/sssd.conf /etc/sssd/sssd.conf.bak
```

## Step 2: Edit sssd.conf
```
sudo vim /etc/sssd/sssd.conf
```


## Step 3: Modify sssd.conf
**3.1** Increase verbosity for troubleshooting
```
[sssd]
debug_level = 5
```
**3.2** Adjust cache settings (if needed)
```
[nss]
entry_cache_timeout = 300
```
**3.3** Configure the services sssd will handle
```
[sssd]
services = nss, pam, sudo
```
**3.4** Tune the LDAP settings (if using LDAP for user information)
```
[domain/example.co.za]
ldap_id_use_start_tls = False
ldap_id_mapping = True
```
**3.5** Fine-tune authentication settings
```
[pam]
pam_verbosity = 3
```

## Step 4: Restart sssd service
```
sudo systemctl restart sssd
```

**NB** Monitor logs for issues /var/log/sssd/*
