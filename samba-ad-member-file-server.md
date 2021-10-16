# Samba AD-Member e File Server

## Para OpenSUSE 15.3

Membro de AD e servidor de arquivos

Máquina 02

## Editar o hostname da máquina:

```
nano /etc/hostname
fileserver
```

**por exemplo**

**Salvar o arquivo**

## Editar arquivo hosts:

```
nano /etc/hosts
127.0.0.1       localhost  
10.1.1.13       FILESERVER.dominio.br  FILESERVER
```

**Salvar o arquivo**

Onde:

```
10.1.1.13 - É o IP de seu próprio servidor de arquivo
FILESERVER.dominio.br - Hostname com o domínio
FILESERVER - Nome identificando a máquina
```

## Editar o arquivo nsswitch.conf:

```
nano /etc/nsswitch.conf
```

Localizar as linhas e substituir por:

```
passwd:       files winbind  
group:        files winbind
```

Salvar o arquivo

## Atualizar o sistema:

```
zypper refresh  
zypper up
```

## Instalar o samba, samba-ad-dc e o samba-winbind:

```
zypper in samba samba-ad-dc samba-winbind
```

## Iniciar os serviço smb, nmb e winbind:

```
systemctl start smb nmb winbind
```

## Habilitar o serviço smb, nmb e winbind no boot do sistema:

```
systemctl enable smb nmb winbind
```

## Abrir a porta do samba no firewall para o samba:

```
firewall-cmd --add-service=samba --permanent
```

## Recarregar o firewall:

```
firewall-cmd --reload
```

## Verificar se o samba tem suporte a ACL com o comando:

```
smbd -b | grep HAVE_LIBACL   
HAVE_LIBACL
```

## Fazer o arquivo user.map com o seguinte conteúdo

```
nano /etc/samba/user.map
```

Inserir o seu domínio com o usuário teste (por exemplo) como abaixo:

```
!root = DOMINIO\teste
```

**Salvar o arquivo**

## Alterar o arquivo config em:

```
nano /etc/sysconfig/network/config
```

Procurar pelas linhas:

```
NETCONFIG_DNS_STATIC_SEARCHLIST="seu.domínio"

Insira um dos DNS como sendo o IP de seu AD:

NETCONFIG_DNS_STATIC_SERVERS="10.1.1.9 8.8.8.8"
```

**salvar o arquivo**

## Reiniciar a network

```
systemctl restart network
```

Configurar o arquivo **smb.conf **para um servidor de arquivo **(fileserver)** em um Domínio. Um exemplo de como pode ser o arquivo seguindo esse modelo **ad**.  **SUBSTITUA** por suas informações:

```
nano /etc/samba/smb.conf
```

```
[global]  
 workgroup = DOMINIO  
 security = ADS  
 realm = DOMINIO.BR  
 bind interfaces only = yes  
 interfaces = 127.0.0.1 eth0

#logs  
 log file = /var/log/samba/%m.log  
 log level = 1

#winbind nss info  
 winbind nss info = rfc2307  

#winbind  
 winbind refresh tickets = Yes  
 winbind enum users = yes  
 winbind enum groups = yes

#objects  
 vfs objects = acl_xattr  
 map acl inherit = Yes  
 store dos attributes = Yes

#keytabs  
 dedicated keytab file = /etc/krb5.keytab  
 kerberos method = secrets and keytab

#winbind dominio default  
 winbind use default domain = yes

#idmaps  
 idmap config * : backend = tdb  
 idmap config * : range = 3000-7999

#usermap  
 username map = /etc/samba/user.map

#idmaps2  
 idmap config DOMINIO.BR:backend = ad  
 idmap config DOMINIO.BR:schema_mode = rfc2307  
 idmap config DOMINIO.BR:range = 10000-999999  
 idmap config DOMINIO.BR:unix_nss_info = yes

#template para login shell e diretorio home  
 template shell = /bin/bash  
 template homedir = /home/%U

#unix primary group  
 idmap config DOMINIO.BR:unix_primary_group = yes
```

**Salvar o arquivo**

## Verificar se existe algum erro de sintaxe com o comando

```
testparm
```

Se não houver erros, executar o comando para recarregar o samba novamente

```
smbcontrol all reload-config
```

## Reiniciar o servidor:

```
reboot
```

## Comando para se juntar ao Domínio Criado:

```
net ads join -U administrator  
Enter administrator's password: senha do provisionamento
```

```
Using short domain name -- DOMINIO  
Joined 'FILESERVER' to dns domain 'dominio.seu'
```

## Verificar conexão NETLOGON com o AD:

```
wbinfo --ping-dc
```

## Verificar se o serverad volta resposta com o comando: nslookup nome__do__domínio:

```
nslookup SRVAD.DOMINIO.BR
```

Por exemplo

E pelo IP do seu Servidor AD:

```
nslookup IP-serverad
```

Se receber as respostas corretas, tudo deve estar funcionando. Caso não consiga obter as repostas, verifique as configurações novamente.

Neste momento você já deve ter uma máquina com Windows, à partir da versão 7 que possibilite se juntar em um domínio. Escolha entre windows 7 e até o 8.1 de 64 bits que **não** sejam versões **HOME EDITION**. O windows 10 infelizmente retirou a aba no RSAT onde trata dos **atributos UNIX** (**GID - Groups ID e ID - Users ID**) e nesse caso você terá que inserir manualmente por ordem. Também na hora de escolher as opções que serão instaladas para o gerenciamento junto ao RSAT, não esquecer de **marcar** as opções como são mostradas ![aqui](/home/douglas/Documentos/rsat-win/ferramenta-nis-instalacao.png)

É necessário deixar estas opões marcadas para que você tenha acesso a aba **Unix Attributes** como ![aqui](/home/douglas/Documentos/rsat-win/unix-attributes1.png)

Então se o usuário** Nil**, for criado através do RSAT com windows 10, você terá que inserir manualmente o valor de usuário, como: **10001** e assim por diante. Do mesmo modo com Grupos: Grupo **TI** terá que inserir manualemente o valor: **3001**, assim como, os caminhos do diretório **home** e o **shell**.

Assumindo que você já criou o **usuário** e o **Grupo** através do **RSAT** no windows e inseriu seus atributos unix (**aba UNIX ATTRIBUTES**) assim sendo, cada um deles possuindo seu GID (número do grupo) e ID (número do usuário), passe para póximo passo estando no servidor de arquivos (file server):

## Verificar se retorna o GID do Grupo:

```
getent group "DOMINIO\\Domain Unix"  
domain unix:x:3006:
```

## Garantir privilégio de acesso do Grupo Criado junto com o usuário criado no RSAT:

Comando:

```
net rpc rights grant "DOMINIO\Domain Unix" SeDiskOperatorPrivilege -U "DOMINIO\usuário_criado"  
Enter DOMINIO\usuário_criado's password: senha do usuário_criado
# Caso tenha criado corretamente o sistema irá dar sucesso do comando

Successfully granted rights.
```

Chegando nesse ponto, está apto a iniciar a configuração da montagem de disco, o diretório escolhido para abrigar seus arquivos bem como, ajustar as permissões de acesso para que seja possível fazer as alterações via RSAT com as devidas permissões. 

Exemplo:

Se criou um disco e disponibilizou o diretório **/srv/ad** e então, indicou esse caminho em seu **smb.conf** ou seja, após a sessão **global** indicando o nome de seu compartilhamento do servidor de arquivos, caminho, e ajustar as permissões de acesso como por exemplo:

```
nano /etc/samba/smb.conf
```

```
[Rede_Arquivos]
   path = /srv/ad/
   read only = no
   browseable = yes
```

Salve o arquivo

Ajustando as permissões para o Grupo no diretório criado:

```
chown root:"Domain Unix" /srv/ad
chmod 755 /srv/ad
```

Então recarregue novamente:

```
smbcontrol all reload-config
```

Através de seu Windows, logue-se com a conta que Criou e Garantiu Direitos e através do RSAT inicie como pretende que funcione seu AD. 

Espero poder ter ajudado. Muito Obrigado






