# Pipeline de CI/CD para Build Aplicação Maven

Este pipeline está estruturado com múltiplos estágios que incluem a compilação, empacotamento e implantação da aplicação. Ele utiliza o Maven para a compilação e empacotamento e configura o proxy e o SonarQube para análise de qualidade.

## Estrutura do Pipeline

### Estágios

1. **app_build**: Compilação e empacotamento da aplicação.
2. **deploy**: Implantação da aplicação (se aplicável).
3. **test**: Execução de testes (se aplicável).

### Variáveis de Ambiente

| Variável                | Descrição                                                |
|-------------------------|----------------------------------------------------------|
| `PROXY_IP_DEV`          | IP do servidor de proxy para o ambiente de desenvolvimento. |
| `PROXY_USER_DEV`        | Usuário para autenticação no proxy.                      |
| `PROXY_PASSWORD_DEV`    | Senha do usuário para autenticação no proxy.             |
| `SONAR_HOST_URL`        | URL do servidor SonarQube.                               |
| `PROXY_DOCKER`          | Endereço do proxy Docker para exportação de variáveis.   |
| `JAVA_VERSION`          | Versão do Java para o ambiente de compilação.

## Configuração dos Jobs

### 1. `app_build`

Este job (`app_build`) é responsável pela compilação e empacotamento da aplicação utilizando Maven e Docker. Ele configura o proxy, compila o código, empacota o projeto, e gera os artefatos para o próximo estágio do pipeline.

```yaml
app_build:                                          # Job de compilação e empacotamento da aplicação.
  stage: app_build                                  # Define que este job pertence ao estágio 'build'.
  image: maven:3.9.6-eclipse-temurin-$JAVA_VERSION  # Define a imagem base Docker que será usada para executar os trabalhos no pipeline.
  script:
    - |
      echo '<settings
      xmlns="http://maven.apache.org/SETTINGS/1.0.0"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
      https://maven.apache.org/xsd/settings-1.0.0.xsd">
        <localRepository>/cache/.m2</localRepository>
        <proxies>
          <proxy>
            <active>true</active>
            <protocol>https</protocol>
            <host>${env.PROXY_IP_DEV}</host>
            <port>8080</port>
            <username>${env.PROXY_USER_DEV}</username>
            <password>${env.PROXY_PASSWORD_DEV}</password>
          </proxy>
        </proxies>
        <pluginGroups>
          <pluginGroup>org.sonarsource.scanner.maven</pluginGroup>
        </pluginGroups>
        <profiles>
          <profile>
            <id>sonar</id>
            <activation>
              <activeByDefault>true</activeByDefault>
            </activation>
            <properties>
              <sonar.host.url>${env.SONAR_HOST_URL}</sonar.host.url>
            </properties>
          </profile>
        </profiles>
      </settings>' > ~/.m2/settings.xml 2>/dev/null || true
    - export http_proxy=$PROXY_DOCKER  # Faz a exportação dos dados do proxy para o container.
    - export https_proxy=$PROXY_DOCKER # Faz a exportação dos dados do proxy para o container.
    - echo "Compilação iniciada"
    - mvn compile                      # Compila o código fonte do projeto.
    - echo "Compilação finalizada"
    - mvn package -DskipTests          # Empacota o projeto em um arquivo JAR, ignorando os testes.
    - ls -laR target/                  # Lista os arquivos no diretório 'target/'.
    - echo "CRIAR PASTA DEPLOY"
    - mkdir -p deploy                  # Cria a pasta 'deploy', se não existir.
    - echo "MOVER DO TARGET PARA PASTA DEPLOY"
    - mv target/*.jar deploy/          # Move os arquivos '.jar' da pasta 'deploy'.
    - echo "MOVIDO FIM"
  artifacts:
    paths:
      - deploy/*.jar                   # Define que os arquivos '.jar' na pasta 'deploy' são artefatos.
    expire_in: 1 week
  tags:
    - docker                           # Define que o job deve ser executado em runners etiquetados com 'docker'.
  allow_failure: false                 # Esta tarefa deve ser bem-sucedida.
