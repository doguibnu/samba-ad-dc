# SAMBA AD-DC

## **Para OpenSUSE 15.3**

Passos para que o OpenSUSE seja um **Controlador de Domínio** em sua Rede. A instalação do **SAMBA-AD-DC** será em uma instalação limpa do sistema ou seja, sem qualquer ferramenta do samba ter sido instalado anteriormente, tanto como um servidor de arquivos simples,  como um serviço de Controlador de domínio anterior ou ainda, Membro de um domínio.  

**OBS: Todos os comandos estarão em modo root no terminal (#)**



### Editar o arquivo hostname

```
nano /etc/hostname 
```

**dclinux**
por exemplo

salvar o arquivo
O editor nano usa as teclas **control+o** para salvar e **control+x** para sair. 

### Atualizar o sistema

```
zypper refresh
zypper up
```

## Instalar o samba-ad-dc e krb5-client

```
zypper in samba-ad-dc krb5-client
```

## Editar o arquivo hosts

```
nano /etc/hosts
```

> 127.0.0.1 localhost
> 
> **10.1.1.9 DCLINUX.SEU.DOMINIO   DCLINUX**

Onde:
**10.1.1.9** = seu IP do Controlador de Domínio (DC)
**Salvar o arquivo**



## Alterar o arquivo config em:

```
nano /etc/sysconfig/network/config
```

Procurar pelas linhas e alterar como mostra

> **NETCONFIG_DNS_STATIC_SEARCHLIST="seu.dominio"  
> NETCONFIG_DNS_STATIC_SERVERS="10.1.1.9 8.8.8.8"**

salvar o arquivo



## Remover o arquivo krb5.conf:

```
rm /etc/krb5.conf
```

## Remover o arquivo smb.conf:

```
rm /etc/samba/smb.con
```

## Reiniciar o sistema:

```
reboot
```

### Ao iniciar novamente, **Habilitar** o serviço samba-ad-dc e **iniciar** o mesmo:

```
systemctl enable samba-ad-dc  
systemctl restart samba-ad-dc
```

## Provisionando o samba em modo **interativo**:

```
samba-tool domain provision --use-rfc2307 --interactive
```

> Realm: **SEU.Dominio.BR**  
> Domain [AD]: **Domínio**  
> Server Role (dc, member, standalone) [dc]: dc 
> DNS backend (SAMBA_INTERNAL, BIND9_FLATFILE, BIND9_DLZ, NONE) SAMBA_INTERNAL]: SAMBA_INTERNAL 
> DNS forwarder IP address (write 'none' to disable forwarding) [SEU_IP_DNS]:IP DNS  
> Administrator password: **escolher uma senha**  
> Retype password: **repetir a mesma senha**



## Copiar o arquivo krb5.conf para o diretório /etc

```
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```

## Editar o arquivo krb5.conf:

```
nano /etc/krb5.conf
```

```
[libdefaults]  

default_realm = DOMINIO.BR  
dns_lookup_realm = false  
dns_lookup_kdc = true
# adicionar a linha
default_ccache_name = FILE:/tmp/krb5cc_%{uid}

[realms]  
DOMINIO.BR = {  
default_domain = dominio.br  
}

[domain_realm]  
localhost = DOMINIO.BR
```

## smb.conf um exemplo do arquivo:

```
nano /etc/samba/smb.conf
```

```
# Global parameters

[global]  
dns forwarder = Seu_DNS  
netbios name = SRVAD  
realm = SEU.DOMINIO.BR  
server role = active directory domain controller  
workgroup = DOMINIO  
idmap_ldb:use rfc2307 = yes

[sysvol]  
path = /var/lib/samba/sysvol  
read only = No

[netlogon]  
path = /var/lib/samba/sysvol/prude.pr/scripts  
read only = No
```

## Reiniciar o serviço do samba-ad-dc

```
systemctl restart samba-ad-dc
```

## Criando zona reversa:

```
samba-tool dns zonecreate IP (seu IP ou hostname ad-dc server) 0.99.10.in-addr.arpa -UAdministrator
```

## Verificando o servidor de arquivo com o comando:

```
smbclient -L localhost -U%
```

## Verificar a autenticação, conectar o compartilhamento netlogon usando a conta de administrador de domínio com o comando:

```
smbclient //localhost/netlogon -UAdministrator -c 'ls'
```

## Verificando DNS:

obs: sem usuário root

```
sudo su - usuário_server
```

```
dig -t SRV _ldap._tcp.seudominio.com  
dig -t SRV _kerberos._udp.seudominio.com  
dig -t A dclinux.seudominio.com
```

## Verificando Kerberos:

obs: sem usuário root

```
sudo su - usuário_server
```

## Fazer o comando:

```
kinit administrator
```

e  então:

```
klist
```

## Abrir as portas necessárias no firewalld para o samba-ad-dc TCP:

```
firewall-cmd --permanent --zone=public --add-port={135/tcp,88/tcp,139/tcp,445/tcp,464/tcp,636/tcp,3268/tcp,3269/tcp,53/tcp,389/tcp,49152-65535/tcp}
```

## E para UDP:

```
firewall-cmd --permanent --zone=public --add-port={53/udp,88/udp,389/udp,123/udp,137/udp,138/udp,464/udp}
```

## Recarregar o Firewalld:

```
firewall-cmd --reload
```

## Listar as portas usadas:

```
firewall-cmd --list-ports
```

Se tudo deu certo até  este ponto, seu controloador de domínio deve estar funcional






