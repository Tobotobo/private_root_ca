# private_root_ca

プライベート認証局によるサーバー証明書の発行  
https://pvision.jp/tech/2023/05/web-server-https-support-by-private-ca/

各項の詳細な手順は↑見て

## 概要
* (前提)２つ証明書の作成が必要
* プライベート認証局の証明書を作成する（以下、ルート証明書）
* Web サーバー用の証明書を作成する（以下、サーバー証明書）
* ルート証明書でサーバー証明書に署名する
* ルート証明書（root-ca.crt）をクライアント（接続元のPCとか）にインストールする
* サーバー証明書（web-server.crt、web-server.key）をサーバー（Apache や Nginx）に設定する

以上により  
ルート証明書がインストールされたクライアントが、サーバー証明書が設定された Web サーバーに https で接続した際、
インストールされたルート証明書によって、そのサーバー証明書がルート証明書によって署名されたものか判断できるようになり、
https 接続が可能となる。

## 環境構築

```
cd ~
mkdir root-ca
cd root-ca
mkdir certs db private
chmod 700 private
touch db/index
openssl rand -hex 16 > db/serial
```

## 認証局の証明書の作成

```
cd ~/root-ca
nano root-ca.conf
```

```
name_opt = utf8,esc_ctrl,multiline,lname,align

[req]
default_bits = 4096
encrypt_key = no
default_md = sha256
utf8 = yes
string_mask = utf8only
prompt = no
distinguished_name = req_dn
req_extensions = req_ext

[req_dn]
countryName = "JP"
organizationName ="Hoge CA"
commonName = "Root CA"

[req_ext]
basicConstraints = critical,CA:true
keyUsage = critical,keyCertSign
subjectKeyIdentifier = hash

[ca]
default_ca = CA_default

[CA_default]
name = root-ca
home = .
database = $home/db/index
serial = $home/db/serial
certificate = $home/$name.crt
private_key = $home/private/$name.key
RANDFILE = $home/private/random
new_certs_dir = $home/certs
unique_subject = no
copy_extensions = none
default_days = 3650
default_md = sha256
policy = policy_match

[policy_match]
countryName = supplied
stateOrProvinceName = optional
organizationName = supplied
organizationalUnitName = optional
commonName = supplied
emailAddress = optional
```

```
cd ~/root-ca
openssl req -new -config root-ca.conf -out root-ca.csr -keyout private/root-ca.key
openssl ca -selfsign -config root-ca.conf -in root-ca.csr -out root-ca.crt -extensions req_ext --batch
```

## 署名用構成ファイルの作成

```
cd ~/root-ca
nano sign-server.conf
```

```
name_opt = utf8,esc_ctrl,multiline,lname,align

[ca]
default_ca = CA_default

[CA_default]
name = root-ca
home = .
database = $home/db/index
serial = $home/db/serial
certificate = $home/$name.crt
private_key = $home/private/$name.key
RANDFILE = $home/private/random
new_certs_dir = $home/certs
unique_subject = no
copy_extensions = copy
default_days = 365
default_md = sha256
policy = policy_match

[policy_match]
countryName = supplied
stateOrProvinceName = optional
organizationName = supplied
organizationalUnitName = optional
commonName = supplied
emailAddress = optional

[server_ext]
authorityKeyIdentifier = keyid:always
basicConstraints = critical,CA:false
extendedKeyUsage = serverAuth
keyUsage = critical,digitalSignature,keyEncipherment
subjectKeyIdentifier = hash
```

ここまで実行した後の状態
```
vagrant@vagrant:~/root-ca$ ls
certs  db  private  root-ca.conf  root-ca.crt  root-ca.csr  sign-server.conf
```

## ルート証明書の受け取り (@クライアントPC)

`~/root-ca/root-ca.crt` をどうにかして取り出してクライアントPCに持っていく  

```
cp ~/root-ca/root-ca.crt /vagrant
```

## ルート証明書のWebブラウザへの読み込み (@クライアントPC)

以下、参照

[プライベート認証局によるサーバー証明書の発行](https://pvision.jp/tech/2023/05/web-server-https-support-by-private-ca/#i-3:~:text=root%2Dca.crt-,%E3%83%AB%E3%83%BC%E3%83%88%E8%A8%BC%E6%98%8E%E6%9B%B8%E3%81%AEWeb%E3%83%96%E3%83%A9%E3%82%A6%E3%82%B6%E3%81%B8%E3%81%AE%E8%AA%AD%E3%81%BF%E8%BE%BC%E3%81%BF%20(%40%E3%82%AF%E3%83%A9%E3%82%A4%E3%82%A2%E3%83%B3%E3%83%88PC),-Web%E3%83%96%E3%83%A9%E3%82%A6%E3%82%B6%E3%81%B8)

## サーバー証明書の作成 (@認証局)

```
cd ~
mkdir web-server
chmod 700 web-server
```

```
cd ~/web-server
openssl genpkey -out web-server.key -algorithm RSA -pkeyopt rsa_keygen_bits:2048
```

```
# openssl req -new -key web-server.key -out web-server.csr -addext "subjectAltName=DNS:example.com"
openssl req -new -key web-server.key -out web-server.csr -addext "subjectAltName=IP:XXX.XXX.XXX.XXX" -subj "/C=JP/O=XXXXXXXX/CN=XXX.XXX.XXX.XXX"
```
※`"subjectAltName=DNS:example.com"` は適宜変更する。IP で指定する場合は `"subjectAltName=IP:XXX.XXX.XXX.XXX"`

```
cd ~/root-ca
openssl ca -config sign-server.conf \
    -in ../web-server/web-server.csr \
    -out ../web-server/web-server.crt \
    -extensions server_ext \
    --batch
```

ここまで実行した後の状態
```
vagrant@vagrant:~$ tree
.
├── root-ca
│   ├── certs
│   │   ├── XXXXXXXXXXXXXXXXXXXXXX.pem
│   │   └── XXXXXXXXXXXXXXXXXXXXXX.pem
│   ├── db
│   │   ├── index
│   │   ├── index.attr
│   │   ├── index.attr.old
│   │   ├── index.old
│   │   ├── serial
│   │   └── serial.old
│   ├── private
│   │   └── root-ca.key
│   ├── root-ca.conf
│   ├── root-ca.crt
│   ├── root-ca.csr
│   └── sign-server.conf
└── web-server
    ├── web-server.crt
    ├── web-server.csr
    └── web-server.key

5 directories, 16 files
```

## Webサーバーへの証明書の組み込み

`~/web-server/web-server.crt` と `~/web-server/web-server.key` をどうにかして取り出してサーバーに持っていく  

```
cp ~/web-server/web-server.crt /vagrant
cp ~/web-server/web-server.key /vagrant
```

設定方法は以下参照
* [Apacheをプライベート認証局やオレオレ証明書でHTTPS化](https://pvision.jp/tech/2023/05/apache-how-to-enable-https/)
* [Nginxをプライベート認証局やオレオレ証明書でHTTPS化](https://pvision.jp/tech/2024/03/install-nginx-with-https/)
