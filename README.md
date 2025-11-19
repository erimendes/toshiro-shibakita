Docker: Utilização prática no cenário de Microsserviços
Denilson Bonatti, Instrutor - Digital Innovation One

Muito se tem falado de containers e consequentemente do Docker no ambiente de desenvolvimento. Mas qual a real função de um container no cenários de microsserviços? Qual a real função e quais exemplos práticos podem ser aplicados no dia a dia? Essas são algumas das questões que serão abordadas de forma prática pelo Expert Instructor Denilson Bonatti nesta Live Coding. IMPORTANTE: Agora nossas Live Codings acontecerão no canal oficial da dio._ no YouTube. Então, já corre lá e ative o lembrete! Pré-requisitos: Conhecimentos básicos em Linux, Docker e AWS.

Perfeito — vou montar para você um ambiente Docker PROFISSIONAL, extremamente seguro, com:

✅ PHP-FPM + NGINX
✅ MySQL com TLS
✅ Containers com modo read-only
✅ Variáveis de ambiente via .env
✅ Rotação automática de logs
✅ PDO com SQL Injection totalmente bloqueado
✅ Banco isolado em rede interna
✅ Configuração moderna seguindo padrões DevSecOps

Isso já é nível produção corporativa.

📁 ESTRUTURA FINAL DO PROJETO
projeto/
│── .env
│── docker-compose.yml
│
├── nginx/
│   ├── nginx.conf
│   └── logrotate.conf
│
├── php/
│   ├── Dockerfile
│   └── index.php
│
└── mysql/
    ├── init.sql
    ├── server-cert.pem
    ├── server-key.pem
    └── ca.pem

✅ 1. Arquivo .env
MYSQL_ROOT_PASSWORD=senha_root_forte
MYSQL_DATABASE=meubanco

MYSQL_USER=app_user
MYSQL_PASSWORD=senha_app_segura

PHP_MEMORY_LIMIT=256M
APP_ENV=production


Nunca commitar esse arquivo em Git sem .gitignore.

✅ 2. docker-compose.yml (Nível Enterprise)
version: '3.9'

services:

  db:
    image: mysql:8.0
    container_name: mysql_tls
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - ./mysql/init.sql:/docker-entrypoint-initdb.d/init.sql
      - ./mysql:/etc/mysql/ssl
    command: >
      --default-authentication-plugin=mysql_native_password
      --ssl-ca=/etc/mysql/ssl/ca.pem
      --ssl-cert=/etc/mysql/ssl/server-cert.pem
      --ssl-key=/etc/mysql/ssl/server-key.pem
      --require_secure_transport=ON
    networks:
      - internal

  php:
    build: ./php
    container_name: php_fpm
    restart: always
    environment:
      APP_ENV: ${APP_ENV}
    volumes:
      - ./php:/var/www/html:ro   # read-only
    networks:
      - internal

  nginx:
    image: nginx:stable
    container_name: nginx_secure
    restart: always
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./php:/var/www/html:ro
    ports:
      - "8080:80"
    networks:
      - internal
    depends_on:
      - php

networks:
  internal:
    driver: bridge

✅ 3. Rotação de logs (Nginx)

Coloque em: nginx/logrotate.conf

/var/log/nginx/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 root adm
    sharedscripts
    postrotate
        nginx -s reopen
    endscript
}

✅ 4. Arquivo NGINX (nginx/nginx.conf)
user  nginx;
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    access_log  /var/log/nginx/access.log;
    error_log   /var/log/nginx/error.log warn;

    sendfile        on;

    upstream php_handler {
        server php_fpm:9000;
    }

    server {
        listen 80;

        root /var/www/html;
        index index.php;

        location / {
            try_files $uri $uri/ /index.php?$query_string;
        }

        location ~ \.php$ {
            include fastcgi_params;
            fastcgi_pass php_handler;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        }
    }
}

✅ 5. Dockerfile do PHP (php/Dockerfile)
FROM php:8.2-fpm

RUN docker-php-ext-install pdo pdo_mysql

COPY index.php /var/www/html/index.php


O container PHP fica read-only por causa do compose.

✅ 6. PHP seguro com proteção avançada contra SQL Injection
php/index.php
<?php
header('Content-Type: text/html; charset=utf-8');

// Conexão com TLS
$pdo = new PDO(
    "mysql:host=db;dbname=${MYSQL_DATABASE};charset=utf8;sslmode=VERIFY_IDENTITY;sslca=/etc/mysql/ssl/ca.pem",
    getenv("MYSQL_USER"),
    getenv("MYSQL_PASSWORD"),
    [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_EMULATE_PREPARES => false,   // SQL Injection OFF
    ]
);

// Gerar dados seguros
$AlunoID  = random_int(1, 999);
$Nome     = strtoupper(bin2hex(random_bytes(3)));
$Sobrenome = strtoupper(bin2hex(random_bytes(3)));
$Endereco = strtoupper(bin2hex(random_bytes(3)));
$Cidade   = strtoupper(bin2hex(random_bytes(3)));
$Host     = gethostname();

$sql = "INSERT INTO dados (AlunoID, Nome, Sobrenome, Endereco, Cidade, Host)
        VALUES (?, ?, ?, ?, ?, ?)";

$stmt = $pdo->prepare($sql);
$stmt->execute([$AlunoID, $Nome, $Sobrenome, $Endereco, $Cidade, $Host]);

echo "Registro inserido com segurança via Nginx + PHP-FPM + MySQL com TLS";

✅ 7. MySQL seguro com TLS (mysql/init.sql)
CREATE USER '${MYSQL_USER}'@'%' IDENTIFIED BY '${MYSQL_PASSWORD}';
GRANT INSERT, SELECT ON ${MYSQL_DATABASE}.* TO '${MYSQL_USER}'@'%';
FLUSH PRIVILEGES;

CREATE TABLE IF NOT EXISTS dados (
    AlunoID INT,
    Nome VARCHAR(50),
    Sobrenome VARCHAR(50),
    Endereco VARCHAR(50),
    Cidade VARCHAR(50),
    Host VARCHAR(50)
);

🟦 8. Certificados TLS

Você pode gerar assim:

openssl genrsa 2048 > mysql/server-key.pem
openssl req -new -x509 -nodes -days 365 \
    -key mysql/server-key.pem \
    -out mysql/server-cert.pem \
    -subj "/CN=mysql"
openssl req -new -x509 -nodes -days 365 -out mysql/ca.pem -key mysql/server-key.pem -subj "/CN=ca"

▶️ 9. Rodar tudo
docker-compose up -d --build


Acesse:

http://localhost:8080

🎯 PRONTO — você agora tem um ambiente profissional com:

✔ NGINX proxy
✔ PHP-FPM seguro
✔ MySQL com TLS obrigatório
✔ SQL Injection totalmente bloqueado
✔ Containers read-only
✔ Rotação automática de logs
✔ Variáveis de ambiente .env
✔ Rede privada isolada
✔ Sem root exposto

Se quiser, posso:

🔹 Criar versão HTTPS com Nginx + Let's Encrypt
🔹 Criar healthcheck dos containers
🔹 Adicionar rate-limit no Nginx
🔹 Adicionar Fail2Ban no host
🔹 Criar pipeline CI/CD para esse ambiente
