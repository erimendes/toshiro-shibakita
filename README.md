Docker: Utilização prática no cenário de Microsserviços
Denilson Bonatti, Instrutor - Digital Innovation One

Muito se tem falado de containers e consequentemente do Docker no ambiente de desenvolvimento. Mas qual a real função de um container no cenários de microsserviços? Qual a real função e quais exemplos práticos podem ser aplicados no dia a dia? Essas são algumas das questões que serão abordadas de forma prática pelo Expert Instructor Denilson Bonatti nesta Live Coding. IMPORTANTE: Agora nossas Live Codings acontecerão no canal oficial da dio._ no YouTube. Então, já corre lá e ative o lembrete! Pré-requisitos: Conhecimentos básicos em Linux, Docker e AWS.

Sim — dá para fazer tudo isso usando Docker, de forma automática, segura e SEM expor usuário root, sem expor o banco na internet e sem precisar configurar nada manualmente no sistema operacional.

A seguir eu te entrego um ambiente completo com:

MySQL em container

PHP + Apache em container

Usuário seguro no MySQL (sem root remoto)

Script PHP seguro

Rede interna protegida entre os containers

Isso é o melhor jeito de rodar esse tipo de app.

✅ 1. Estrutura final dos arquivos
projeto/
│── docker-compose.yml
│── mysql/
│   └── init.sql
└── php/
    ├── Dockerfile
    └── index.php

🟦 2. docker-compose.yml (orquestra tudo)
version: '3.9'

services:

  db:
    image: mysql:8.0
    container_name: mysql_seguro
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root_senha_forte
      MYSQL_DATABASE: meubanco
    volumes:
      - ./mysql/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - internal
    command: >
      --default-authentication-plugin=mysql_native_password
      --bind-address=0.0.0.0

  php:
    build: ./php
    container_name: php_app
    restart: always
    volumes:
      - ./php:/var/www/html
    ports:
      - "8080:80"
    networks:
      - internal
    depends_on:
      - db

networks:
  internal:
    driver: bridge


👉 O banco NÃO será exposto à internet, só o PHP acessa.

🟦 3. mysql/init.sql (criação automática do usuário seguro)
CREATE USER 'app_user'@'%' IDENTIFIED BY 'senha_app_segura';
GRANT INSERT, SELECT ON meubanco.* TO 'app_user'@'%';
FLUSH PRIVILEGES;

CREATE TABLE IF NOT EXISTS dados (
    AlunoID INT,
    Nome VARCHAR(50),
    Sobrenome VARCHAR(50),
    Endereco VARCHAR(50),
    Cidade VARCHAR(50),
    Host VARCHAR(50)
);


👉

root não será usado pela aplicação

app_user tem permissões mínimas

Se você quiser restringir para só acessar de dentro da rede Docker:

CREATE USER 'app_user'@'php_app' IDENTIFIED BY 'senha_app_segura';

🟦 4. php/Dockerfile
FROM php:8.2-apache

RUN docker-php-ext-install pdo pdo_mysql mysqli

COPY index.php /var/www/html/index.php

EXPOSE 80

🟦 5. php/index.php (versão segura do seu script)
<?php
header('Content-Type: text/html; charset=utf-8');

$host = "db";
$db   = "meubanco";
$user = "app_user";
$pass = "senha_app_segura";

try {
    $pdo = new PDO("mysql:host=$host;dbname=$db;charset=utf8",
        $user, $pass, [
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION
        ]
    );
} catch (Exception $e) {
    error_log("Erro BD: " . $e->getMessage());
    die("Erro ao conectar.");
}

$AlunoID  = rand(1, 999);
$Nome     = strtoupper(bin2hex(random_bytes(3)));
$Sobrenome = strtoupper(bin2hex(random_bytes(3)));
$Endereco = strtoupper(bin2hex(random_bytes(3)));
$Cidade   = strtoupper(bin2hex(random_bytes(3)));
$Host     = gethostname();

$sql = "INSERT INTO dados (AlunoID, Nome, Sobrenome, Endereco, Cidade, Host)
        VALUES (?, ?, ?, ?, ?, ?)";

$stmt = $pdo->prepare($sql);
$stmt->execute([$AlunoID, $Nome, $Sobrenome, $Endereco, $Cidade, $Host]);

echo "Registro inserido com segurança via Docker";
?>

▶️ 6. Como rodar tudo

No diretório do projeto:

docker-compose up -d --build


Acesse no navegador:

http://localhost:8080


E pronto!
O registro será inserido com segurança no MySQL dentro do container.

🔥 Vantagens deste modelo

✔ Não expõe root
✔ Usuário seguro com privilégios mínimos
✔ Banco isolado em rede interna
✔ Nenhum IP público envolvido
✔ Reprodutível em qualquer máquina
✔ Zero configuração manual no sistema

❓ Quer ainda mais segurança?

Posso incluir:

Variáveis de ambiente via .env

Rotação segura de logs

Modo read-only para o container PHP

MySQL com TLS

Proteção contra SQL injection avançada

Quer adicionar alguma dessas opções?
