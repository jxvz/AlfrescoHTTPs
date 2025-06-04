# 🔐 Alfresco HTTPS  
**🔐🛠️ Passo a Passo para Configurar Nginx como Proxy e HTTPS para o Alfresco**  
Antes de iniciar é necessário ter o Nginx instalado no servidor.
```bash
sudo apt install nginx -y
```
![Nginx](https://img.shields.io/badge/-Nginx-%23009639?logo=nginx&logoColor=white) ![OpenSSL](https://img.shields.io/badge/-OpenSSL-%23721416?logo=openssl&logoColor=white)

---

## 🔧 Configuração no Servidor Proxy (Linux/Nginx)

### 1. 🛡️➕ Criar Autoridade Certificadora (CA) Local
```bash
openssl genrsa -out alfresco-CA.key 4096
openssl req -x509 -new -nodes -key alfresco-CA.key -sha256 -days 3650 -out alfresco-CA.crt \
  -subj "/C=BR/ST=Estado/L=Cidade/O=Empresa/CN=alfresco.homologa"
```

### 2. 🔑➕ Gerar Chave Privada para o Alfresco

```bash
openssl genrsa -out alfresco.key 2048
```

### 3. 📄✍️ Criar Certificate Signing Request (CSR)

```bash
openssl req -new -key alfresco.key -out alfresco.csr \
  -subj "/C=BR/ST=Estado/L=Cidade/O=Empresa/CN=alfresco.homologa"
```

### 4. 📄🛠️ Criar Arquivo de Extensão SAN (alfresco.ext)

```bash 
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = alfresco.homologa
```

### 5. 📜✔️🛡️ Assinar Certificado com a CA

```bash
openssl x509 -req -in alfresco.csr -CA alfresco-CA.crt -CAkey alfresco-CA.key -CAcreateserial \
  -out alfresco.crt -days 825 -sha256 -extfile alfresco.ext
```

### 6. 🖥️⚙️ Configurar Nginx (/etc/nginx/conf.d/alfresco.conf)

```bash

server {
    listen 443 ssl;
    server_name alfresco.homologa;

    ssl_certificate /etc/nginx/ssl/alfresco.crt;
    ssl_certificate_key /etc/nginx/ssl/alfresco.key;

    # Redireciona a raiz para /share/
    location = / {
        return 301 /share/;
    }

    # Proxy para o Alfresco Share corretamente
    location /share/ {
        proxy_pass http://ip-alfresco/share/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
    # Proxy para o backend Alfresco (WebDAV, API, etc)
    location /alfresco/ {
        proxy_pass http://ip-alfresco/alfresco/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }

}
```
### 7. 🖥️🔄 Reiniciar Nginx

```bash
sudo systemctl restart nginx
```

### 💻 Configuração no Windows (DNS/Cliente)

1. 📤 Transferir Certificado CA
Copie alfresco-CA.crt para o Windows (via rede/pendrive).

2. 🔒 Instalar Certificado CA
Clique duplo em alfresco-CA.crt

Clique em "Instalar Certificado"

Selecione "Usuário Atual" ou "Computador Local"

Escolha:
"Colocar todos os certificados no repositório a seguir" → "Procurar..." →
"Autoridades de Certificação Raiz Confiáveis"

3. ✅ Verificar Instalação
Execute certmgr.msc

Navegue até:
Autoridades de Certificação Raiz Confiáveis → Certificados

Confirme se alfresco-CA está listado.




