# Sistema Programa de Gestão de Desempenho - PGD - Fork Docker

Versão do [Sistema_Programa_de_Gestao_Susep](https://github.com/spbgovbr/Sistema_Programa_de_Gestao_Susep) que sobe em containers Docker.

## Subir aplicação em ambiente Docker

É possível subir a aplicação por meio do [Docker](https://www.docker.com/). Dentre as vantagens estão:
1. A ausência da necessidade de uma configuração do IIS;
1. A simplificação da configuração (a maior parte das configurações já foram feitas);
1. A ausência da necessidade de possuir licenças para o Windows Server para rodar a aplicação, tendo em vista que as imagens foram configuradas utilizando Microsoft suporta oficialmente;
1. A ausência da obrigatoriedade de configurar um servidor SQL Server. O docker-compose utiliza o SQL Server 2019 para Linux, oficialmente suportado para a Microsoft, no modo de avaliação. Atente-se que a utilização em produção exige uma licença válida.

**Observações:**

* Para possibilitar a execução em ambiente docker (imagens linux/amd64), realizamos uma troca do plugin de logs Eventlog para o Serilog, pois o Serilog é multiplataforma. É possível ver todas as mudanças e adições realizadas neste [pull request](https://github.com/spbgovbr/Sistema_Programa_de_Gestao_Susep/pull/17);
* Dado que até o momento o código fonte não foi completamente compartilhado, tivemos de utilizar algumas `dlls` já compiladas, disponibilizadas pela SUSEP. Elas foram copiadas da pasta [install/](install/) para a pasta [src/Susep.libs/](src/Susep.libs/);
* Deve ser possível utilizar a mesma solução em servidores Windows, caso ele esteja configurado para rodar imagens Linux. Porém, o desempenho possivelmente será ligeiramente inferior do que se configurar diretamente com IIS. Não foi testado;
  * A microsoft também disponibiliza imagens docker nativas para Windows. Porém a minha versão do SO não é a Professional, então não tive como testar. A geração de imagens com Github actions está nos planos futuros.
* A aplicação provavelmente deve funcionar em máquinas Mac, mas também não foi testado.


### Configurando a aplicação

Em uma máquina que tenha o [Docker](https://docs.docker.com/engine/install/) e o [docker-compose](https://docs.docker.com/compose/install/) instalados, baixe o código. Esse passo pode ser via git
```bash
git clone https://github.com/SrMouraSilva/Sistema_Programa_de_Gestao_Susep.git
git checkout docker-codigo-fonte
```
ou baixando diretamente o código pelo link:
* <https://github.com/SrMouraSilva/Sistema_Programa_de_Gestao_Susep/archive/docker-codigo-fonte.zip>

Após baixar, acesse a pasta do projeto pelo terminal
```bash
cd Sistema_Programa_de_Gestao_Susep
```

Por fim, execute o seguinte comando
```bash
docker-compose -f docker/docker-compose.yml up -d
```

Pronto, a aplicação está acessível no endereço http://localhost. Porém você não irá conseguir se logar se não configurar o LDAP (veja abaixo) e se não inserir as pessoas na tabela de pessoas.

#### Configurações

Após alterar uma configuração, execute
```bash
docker-compose -f docker/docker-compose.yml down
docker-compose -f docker/docker-compose.yml up -d
```

##### Verificando se deu certo

Execute o seguinte comando
```bash
docker-compose -f docker/docker-compose.yml ps -a
```
Os 5 containers devem estar ativos. Caso nenhum dos três relacionados ao dotnet subirem (`web-app`, `web-api`, `gateway`), provavelmente o usuário **dos containers** não tem permissão o suficiente para subir o processo e utilizar a porta 80. Esse erro foi detectado no CentOs, mas não ocorre no Debian e nem no Ubuntu. A forma que sabemos até o momento de "contornar" esse problema é dizer que o usuário root que irá rodar esse processo. Altere o docker-compose adicionando o seguinte:
```diff
web-api:
    image: ghcr.io/srmourasilva/sistema_programa_de_gestao_susep/sgd:latest
+    user: 0:0
...
api-gateway:
    image: ghcr.io/srmourasilva/sistema_programa_de_gestao_susep/sgd:latest
+    user: 0:0
...
web-app:
    image: ghcr.io/srmourasilva/sistema_programa_de_gestao_susep/sgd:latest
+    user: 0:0
```
Atente-se que o yml é sensível a identação e que foi utilizado espaço como identação.

##### Configurar Servidor de email

Acesse o arquivo `docker/api/Settings/appsettings.Homolog.json` e edite as seguintes linhas
```json
"emailOptions": {
  "EmailRemetente": "ENDEREÇO EMAIL REMETENTE SAIDA",
  "NomeRemetente": "NOME REMETENTE SAIDA",
  "SmtpServer": "SERVIDOR SMTP",
  "Port": "PORTA SERVIDOR SMTP"
},
```

##### Configurar Servidor ldap

Acesse o arquivo `docker/api/Settings/appsettings.Homolog.json` e edite as seguintes linhas
```json
"ldapOptions": {
  "Url": "URL SERVIDOR LDAP",
  "Port": 389,
  "BindDN": "DN do usuário de serviço que será utilizado para autenticar no LDAP", //Exemplo: CN=Fulano de tal,CN=Users,DC=orgao
  "BindPassword": "Senha do usuário de serviço que será utilizado para autenticar no LDAP",
  "SearchBaseDC": "DC que será utilizado para chegar à base de usuários no LDAP", //Exemplo: CN=Users,DC=orgao
  "SearchFilter": "Consulta a ser aplicada no LDAP para encontrar os usuários", //Exemplo: (&(objectClass=user)(objectClass=person)(sAMAccountName={0}))
  "CpfAttributeFilter": "Campo do LDAP em que será encontrado o CPF do usuário", 
  "EmailAttributeFilter": "Campo do LDAP em que será encontrado o e-mail do usuário"
}
```

###### Observações

* O login só ocorrerá adequadamente caso exista um usuário na tabela `[dbo].[Pessoa]` com o CPF e o email igual ao usuário do LDAP;
* Caso seja consultado uma pessoa que não exista na base do LDAP, o `api-gateway` retornará um erro `500` e nada será exibido para o usuário pelo `web-app`. No response você poderá ver uma mensagem como `System.Threading.Tasks.TaskCanceledException: A task was canceled.`;
* Por algum motivo desconhecido, em alguns casos é necessário pressionar o botão de `Entrar` duas vezes;
* A aplicação possui [usuários de teste](https://github.com/spbgovbr/Sistema_Programa_de_Gestao_Susep#valida%C3%A7%C3%A3o-da-instala%C3%A7%C3%A3o-3%C2%AA-etapa) para simplificar o processo de homologação. Você pode ver [a lista completa dos usuários que não necessitam de senha](https://github.com/spbgovbr/Sistema_Programa_de_Gestao_Susep/blob/97892e1/src/Susep.SISRH.Application/Auth/ResourceOwnerPasswordValidator.cs#L45-L54). Caso utilize em produção, **RETIRE ESSES USUÁRIOS DO SCRIPT** `install/3. Inserir dados de teste.sql`. Caso não sejam retirados, estes usuários poderão serem utilizados por pessoas má-intensionadas como [backdook](https://pt.wikipedia.org/wiki/Backdoor).

##### Configurar Acesso ao Banco de Dados

Caso você queira utilizar um servidor de banco de dados SQL Server com a devida licença, será necessário alterar a configuração do banco.
Acesse o arquivo `connectionstrings.Homolog.json` e edite as seguintes linhas:
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "data source=db;initial catalog=master;User ID=sa;Password=P1ssw@rd;"
  }
}
```

##### Configurar Script de inserção de dados no Banco de Dados

**Obs:** Este passo só é necessário caso você utilize o banco de dados configurado no docker-compose. Caso você decida utilizar outro banco de dados (por exemplo, um banco SQL Server com licença), naturalmente você deverá executar todos os scripts do banco.

Edite o arquivo `install/3. Inserir dados de teste.sql`, conforme desejado.

**Obs:** Quando o banco de dados aplicação sobe, são executados os três arquivos `.sql` automaticamente.

## Trabalhos futuros

1. Configurar um servidor LDAP de testes para possibilitar uma homologação do sistema sem precisar configurar manualmente esse passo (criar um docker-compose específico);
1. Banco de homologação
   1. Configurar adequadamente o volume do banco;
   1. Evitar que o scripts sqls sejam executados mais de uma vez em caso de reinício do banco de dados;
1. Criar docker compose de produção:
   1. Sem banco de dados;
   1. Ajustar configurações para uso em produção para evitar exibir stacktrace  desnecessário para os usuários:
      * Mudar arquivos de configuração de `Homolog` para `Production`;
      * Mudar variáveis de ambiente de `Homolog` para `Production`.
1. Gerar imagens nativas para Windows por Github Actions.

## Outras informações

Caso você deseje fazer o build local ao invés de utilizar a imagem preparada:
```
docker build -f docker/Dockerfile -t susep .
```
