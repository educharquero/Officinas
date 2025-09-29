# 📁 DC01 instalação por pacotes binários


## Atualizando o Sistema operacional:

```
sudo apt update && sudo apt full-upgrade
```

## Setando as Configurações e rede:

```
vim /etc/network/interfaces
```

```
allow-hotplug en7s0
iface enp7s0 inet static
address 192.168.70.250
netmask 255.255.255.0
gateway 192.168.70.254
```

## Setando as Configurações de DNS:

```
vim /etc/resolv.conf
```

```
nameserver 192.168.70.254
```

## Setando as Configurações de hosts:

```
hostnamectl set-hostname dc01
```

```
vim /etc/hosts
```

```
127.0.0.1 localhost 
127.0.1.1 dc01.esharknet.edu dc01
192.168.70.250 dc01.esharknet.edu dc01
```

## Instalação dos pacotes necessários:

```
apt install samba smb-client krb5-user krb5-config # VALIDAR: winbind acl attr
```

Durante a configuração do krb5-user, você será solicitado a inserir as informações:

    - Default Kerberos version 5 realm: ESHARKNET.EDU
    - KDCs for your realm: dc01
    - Administrative server for your Kerberos realm: dc01

SE vc errar nessa parte, pode rodar o reconfigure para o Kerberos, depois:

```
dpkg-reconfigure krb5.conf
```

Siga as instruções interativas para configurar o domínio ESHARKNET.edu. SE precisar siga o modelo abaixo:

```bash
[libdefaults]
    default_realm = ESHARKNET.EDU
    dns_lookup_realm = false
    dns_lookup_kdc = true

# The following krb5.conf variables are only for MIT Kerberos.
    kdc_timesync = 1
    ccache_type = 4
    forwardable = true
    proxiable = true
    rdns = false

# The following libdefaults parameters are only for Heimdal Kerberos.
    fcc-mit-ticketflags = true

[realms]
    esharknet.edu = {
        kdc = dc01.esharknet.edu
        admin_server = dc01.esharknet.edu
        default_domain = esharknet.edu
    }

[domain_realm]
    .esharknet.edu = ESHARKNET.EDU
    esharknet.edu = ESHARKNET.EDU
```

## Valide o arquivo do nsswitch.conf:

```bash
vim /etc/nsswitch.conf
```
```
passwd:       files systemd winbind
group:        files systemd winbind
```

## Parando os serviços `smbd`, `nmbd`, `winbind`, ANTES de provisionar o domínio:

```
systemctl stop smbd nmbd winbind

systemctl disable smbd nmbd winbind
```

## Efetivando o _provisionamento_ do domínio:

```
mv /etc/samba/smb.conf{,.orig}
```

```
samba-tool domain provision --use-rfc2307 --interactive
```
```bash
Realm [ESHARKNET.EDU]: (nome de seu domínio completo).
dc01 [esharknet]: (nome de seu domínio abreviado).
Server Role (dc, member, standalone) [dc]: (tipo de servidor controlador de domínio).
DNS backend (SAMBA_INTERNAL, BIND9_FLATFILE, BIND9_DLZ, NONE) [SAMBA_INTERNAL]: (tipo de serviço DNS).
DNS forwarder IP address (write ‘none’ to disable forwarding) [192.168.0.1]: (Encaminhador DNS)
```

## Validando as portas em escuta:

```
apt install net-tools
```

```
netstat -pultan
```

## Ativando os serviços do `samba-ad-dc`
```
systemctl unmask samba-ad-dc.service
```
```
systemctl enable samba-ad-dc.service
```
```
systemctl start samba-ad-dc.service
```

## Revalidando as portas em escuta:

```
apt install net-tools
```

```
netstat -pultan
```

## Reapontando o Servidor de DNS para o próprio DC01:

```
vim /etc/resolv.conf
```

```
nameserver 127.0.0.1
```

## Travando o arquivo contra alterações automáticas:

```
chattr +i /etc/resolv.conf
```

## Configuração do arquivo `smb.conf`:

```
vim /etc/samba/smb.conf
```
# Global parameters
    [global]
	    dns forwarder = 192.168.122.1
	    netbios name = dc01
	    realm = ESHARKNET.EDU
	    server role = active directory domain controller
	    workgroup = ESHARKNET
	    idmap_ldb:use rfc2307 = yes

    [sysvol]
	    path = /var/lib/samba/sysvol
	    read only = No

    [netlogon]
	    path = /var/lib/samba/sysvol/esharknet.edu/scripts
	    read only = No

# SE for usar o Controlador de Domínio como FileServer (Não recomendado!), adiciona os campos ao smb.conf:

    [ARQUIVOS]
        path = /srv/samba/arquivos
        comment = Compartilhamentos da Rede
        browsable = yes
        writable = yes
        read only = no


## Criando o diretório compartilhado:

```
mkdir -p /srv/samba/arquivos
```
## Setando as permissões de acesso:

```
wbinfo -g
```
```
chmod -R 0770 /srv/samba/arquivos
chown -R root:"domain users" /srv/samba/arquivos
```

## Revisando usuários e grupos:

```
cat /etc/passwd # ( usuários locais do Linux gerencia do LDAP )
```
```
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
dhcpcd:x:100:65534:DHCP Client Daemon,,,:/usr/lib/dhcpcd:/bin/false
systemd-timesync:x:991:991:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:990:990:System Message Bus:/nonexistent:/usr/sbin/nologin
sshd:x:989:65534:sshd user:/run/sshd:/usr/sbin/nologin
suporte:x:1000:1000:Suporte:/home/suporte:/bin/bash
```

```
samba-tool user list # ( usuários de rede gerenciados pelo SAMBA4 )
```
```
Administrator
Guest
krbtgt
```

## Outras validações no domínio:

```
wbinfo -u
```
```
ESHARKNET\administrator
ESHARKNET\guest
ESHARKNET\krbtgt
```

```
wbinfo -g
```
```
ESHARKNET\cert publishers
ESHARKNET\ras and ias servers
ESHARKNET\allowed rodc password replication group
ESHARKNET\denied rodc password replication group
ESHARKNET\dnsadmins
ESHARKNET\enterprise read-only domain controllers
ESHARKNET\domain admins
ESHARKNET\domain users
ESHARKNET\domain guests
ESHARKNET\domain computers
ESHARKNET\domain controllers
ESHARKNET\schema admins
ESHARKNET\enterprise admins
ESHARKNET\group policy creator owners
ESHARKNET\read-only domain controllers
ESHARKNET\protected users
ESHARKNET\dnsupdateproxy
```

```
wbinfo --ping-dc
```
```
checking the NETLOGON for domain[ESHARNKET dc connection to "dc01.esharknet.edu" succeeded
```

```
wbinfo --all-domains
```
```
BUILTIN
ESHARKNET
```

## Verificando serviço ativo:

```
ps aux | grep samba
```
```
ps ax | egrep "samba|smbd|nmbd|winbindd"
```
```
ps axf
```

## Consultando o Servidor:

```
testparm
```
```
Load smb config files from /etc/samba/smb.conf
Loaded services file OK.
Weak crypto is allowed by GnuTLS (e.g. NTLM as a compatibility fallback)

Server role: ROLE_ACTIVE_DIRECTORY_DC

Press enter to see a dump of your service definitions

# Global parameters
[global]
	dns forwarder = 192.168.70.1
	passdb backend = samba_dsdb
	realm = ESHARNKET.INFO
	server role = active directory domain controller
	workgroup = ESHARNKET
	rpc_server:tcpip = no
	rpc_daemon:spoolssd = embedded
	rpc_server:spoolss = embedded
	rpc_server:winreg = embedded
	rpc_server:ntsvcs = embedded
	rpc_server:eventlog = embedded
	rpc_server:srvsvc = embedded
	rpc_server:svcctl = embedded
	rpc_server:default = external
	winbindd:use external pipes = true
	idmap_ldb:use rfc2307 = yes
	idmap config * : backend = tdb
	map archive = No
	vfs objects = dfs_samba4 acl_xattr


[sysvol]
	path = /var/lib/samba/sysvol
	read only = No


[netlogon]
	path = /var/lib/samba/sysvol/esharknet.edu/scripts
	read only = No


[ARQUIVOS]
	comment = Compartilhamentos da Rede
	path = /srv/samba/Arquivos
	read only = No
```
```
smbclient --version
```
```
Version 4.22.4-Debian-4.22.4+dfsg-1~deb13u1

```

```
smbclient -L localhost -U%
```
```
Version 4.22.4-Debian-4.22.4+dfsg-1~deb13u1
root@dc01:~# smbclient -L localhost -U%

	Sharename       Type      Comment
	---------       ----      -------
	sysvol          Disk      
	netlogon        Disk      
	ARQUIVOS        Disk      Compartilhamentos da Rede
	IPC$            IPC       IPC Service (Samba 4.22.4-Debian-4.22.4+dfsg-1~deb13u1)
SMB1 disabled -- no workgroup available
```

```
samba-tool domain level show
```
```
dc01 and forest function level for dc01 'DC=officinas,DC=edu'

Forest function level: (Windows) 2008 R2
dc01 function level: (Windows) 2008 R2
Lowest function level of a DC: (Windows) 2008 R2
```

## Desabilitando a complexidade de senhas (INSEGURO!):

```
samba-tool domain passwordsettings show
```
```
samba-tool domain passwordsettings set --complexity=off
```
```
samba-tool domain passwordsettings set --history-length=0
```
```
samba-tool domain passwordsettings set --min-pwd-length=0
```
```
samba-tool domain passwordsettings set --min-pwd-age=0
```
```
samba-tool user setexpiry Administrator --noexpiry
```

## Relendo as configurações do Samba4:

```
smbcontrol all reload-config
```

## Validando a troca de tickets do Kerberos:

```
kinit Administrator@ESHARKNET.EDU
```
```
klist
```
```
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: Administrator@ESHARKNET.EDU

Valid starting       Expires              Service principal
23/09/2025 16:35:24  24/09/2025 02:35:24  krbtgt/ESHARKNET.EDU@ESHARKNET.EDU
	renew until 24/09/2025 16:35:19
```

## Consultando serviços do `kerberos` e do `ldap`:

```
host -t A esharknet.edu
```
```
esharknet.edu has address 192.168.70.250
```

```
Using dc01 server:
Name: esharknet.edu
Address: 192.168.70.1#53
Aliases: 

A has no SRV record
```
```
host -t srv _kerberos._tcp.ESHARKNET.edu
```
```
_ldap._tcp.esharknet.edu has SRV record 0 100 389 dc01.esharknet.edu.
```
```
host -t srv _ldap._tcp.ESHARKNET.edu
```
```
_kerberos._udp.esharknet.edu has SRV record 0 100 88 dc01.esharknet.edu.
```
```
dig ESHARKNET.EDU
```
```
; <<>> DiG 9.20.11-4-Debian <<>> ESHARKNET.EDU
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 52366
;; flags: qr aa rd ra ad; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 0

;; QUESTION SECTION:
;ESHARKNET.EDU.			IN	A

;; ANSWER SECTION:
ESHARKNET.EDU.		900	IN	A	192.168.70.250

;; AUTHORITY SECTION:
esharknet.edu.		3600	IN	SOA	dc01.esharknet.edu. hostmaster.esharknet.edu. 19 900 600 86400 3600

;; Query time: 4 msec
;; SERVER: 127.0.0.1#53(127.0.0.1) (UDP)
;; WHEN: Tue Sep 23 16:40:41 -03 2025
;; MSG SIZE  rcvd: 120
```


## Agora é só instalar as ferramentas do `rsat` em uma máquina Windows e gerenciar o Servidor!


THAT’S ALL FOLKS!!
