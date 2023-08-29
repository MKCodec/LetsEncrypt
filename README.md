# Let's Encrypt MkAuth + RouterOS 7 / Mikrotik

[![Mikrotik](https://mikrotik.com/img/mtv2/newlogo.svg)](https://mikrotik.com/)


### Funcionalidade:
* Gerar Certificado SSL para o site do MkAuth
* Gerar Certificado SSL para a Routerboard com Mikrotik v7
* Atualizar Certificados Automaticamente

### Configuração no Mikrotik v7

 1 - Crie o usuario LetsEncrypt com privilegio de admin via terminal
* em Password Insira uma senha de sua escolha
* em Address insira o IP do seu MkAuth para Limitar a Comunicação SSH
```sh
/user add name=LetsEncrypt password=1234567890 group=full address=172.31.255.2
```

 2 - Habilite o serviço SSH
```sh
/ip service enable ssh
```
  
### Instalação no MkAuth 23.06

3 - Instale o git no seu sistema MkAuth
```sh
sudo -s
apt update
apt install git
```

4 - Instale o repositorio no seu sistema MkAuth
```sh
cd /opt
git clone https://github.com/MKCodec/LetsEncrypt.git
```
5 - Edite o arquivo de Configuração 

* Pode ser feito editando o arquivo pelo WinSCP ou via Editor Linux
```sh
nano /opt/LetsEncrypt/LetsEncrypt.settings
```
| Variavel | Valor | Descrição |
| ------ | ------ | ------ |
| ROUTEROS_HOST | 192.168.88.1 | IP local de acesso ao Mikrotik |
| DOMAIN | meudominio.com | Dominio que será Certificado |
| HOTSPOT_PROFILLE | default | Hotspot que receberá o Certificado |
| HOTSPOT_AUTH | mac,http-chap,http-pap,trial | Tipos de autenticação do Hotspot |


6 - Edite as Permissões do Arquivo:
```sh
chmod +x /opt/LetsEncrypt/LetsEncrypt.sh
```
7 - Gere uma Chave RSA para o acesso via SSH
```sh
ssh-keygen -t rsa -f /opt/LetsEncrypt/LetsEncrypt -N ""
```

8 - Envie a Chave Gerada para o Mikrotik

* Sera solicitado a senha criada no passo 1
```sh
source /opt/LetsEncrypt/LetsEncrypt.settings
scp -P $ROUTEROS_SSH_PORT /opt/LetsEncrypt/LetsEncrypt.pub "$ROUTEROS_USER"@"$ROUTEROS_HOST":"LetsEncrypt.pub"
```

### Configuração no Mikrotik v7
9 - *Importe a chave SSH gerada no MkAuth para o Mikrotik*

```sh
/user ssh-keys import user=LetsEncrypt public-key-file=LetsEncrypt.pub
```
### Instalação no MkAuth 23.06

10 - Instale o certbot no seu sistema MkAuth
```sh
apt update
apt install software-properties-common -y
add-apt-repository ppa:certbot/certbot
apt update
apt install certbot -y
```

11 - Gere um Certificado SSL com o certbot

* O certificado gerado é automaticamente instalado para o mikrotik
* no codigo abaixo será gerado um certificado coringa para o dominio especificado no passo 5
```sh
source /opt/LetsEncrypt/LetsEncrypt.settings
certbot certonly --preferred-challenges=dns --manual -d *.$DOMAIN --manual-public-ip-logging-ok --post-hook /opt/LetsEncrypt/LetsEncrypt.sh --server https://acme-v02.api.letsencrypt.org/directory
```
* caso queira gerar um certificado para um subdominio não coringa utilize o codigo abaixo
```sh
source /opt/LetsEncrypt/LetsEncrypt.settings
certbot certonly --preferred-challenges=dns --manual -d $DOMAIN --manual-public-ip-logging-ok --post-hook /opt/LetsEncrypt/LetsEncrypt.sh --server https://acme-v02.api.letsencrypt.org/directory
```
* Obs: Caso queira gerar um certificado a parte sem importação ao mikrotik utilize o codigo abaixo
```sh
certbot certonly --preferred-challenges=dns --manual -d meudominio.com --manual-public-ip-logging-ok --server https://acme-v02.api.letsencrypt.org/directory
```

12 - Habilite o certificado no MkAuth
* Pode ser feito editando o arquivo pelo WinSCP ou via Editor Linux
```sh
nano /etc/apache2/sites-enabled/000-default.conf

```
* insira o seguinte codigo dentro das tags <VirtualHost *:443> e <VirtualHost *:445>
```sh
    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/meudominio.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/meudominio.com/privkey.pem
    Include /etc/letsencrypt/options-ssl-apache.conf
```

13 - Habilite a renovação do certificado de forma automatica a cada 25 dias

```sh
crontab -e
```
* insira o seguinte codigo no final do arquivo
```sh
# RENOVAR CERTIFICADO SSL
* * 25 * * /usr/bin/certbot -q renew >/dev/null 2>&1
```

14 - Reinicie o apache do MkAuth para Validar o Certificado

```sh
service apache2 restart
```

15 - acesse seu dominio mkauth ou mikrotik ( webadmin ou hotspot ) e verifique a conclusão da instalação

```sh
https://meudominio.com
```

