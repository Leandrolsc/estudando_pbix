Projeto de Estudo: CI/CD para Power BI On-Premises (PBIRS)

Objetivo: Implementar um pipeline de CI/CD (Integração Contínua e Entrega Contínua) para relatórios Power BI (.pbix) em um ambiente On-Premises (Power BI Report Server).

Este projeto utiliza pbi-tools para permitir a visualização de diferenças (diff) em Pull Requests e a API do PBIRS (através de PowerShell) para publicação automática.

1. Conceitos e Ferramentas

a. O Problema do .pbix

O arquivo .pbix é um binário compilado (essencialmente um arquivo zip). Isso o torna péssimo para controle de versão, pois o Git não consegue identificar o que mudou dentro dele (ex: uma medida DAX, um objeto visual).

b. Solução: pbi-tools

É uma ferramenta de linha de comando (open-source) que "quebra" (descompila) o .pbix em uma estrutura de pastas legível por humanos:

Model.bim (JSON com o modelo de dados)

*.dax (arquivos de texto para cada medida)

*.m (arquivos de texto para cada consulta do Power Query)

Report.json (layout do relatório)

Ao versionar essa estrutura de pastas junto com o .pbix, podemos ver as mudanças exatas em um Pull Request.

c. Solução: API do PBIRS (via PowerShell)

O Power BI Report Server expõe uma API REST (v2.0) que permite o upload e gerenciamento de relatórios. A maneira mais fácil de interagir com ela é usando o módulo ReportingServicesTools do PowerShell, que abstrai a complexidade da API.

2. Arquitetura do Pipeline

O fluxo de trabalho será o seguinte:

Desenvolvedor (Local):

Edita o MeuRelatorio.pbix.

Executa pbi-tools extract MeuRelatorio.pbix -out "src/MeuRelatorio".

Commita ambos o .pbix e a pasta src/MeuRelatorio.

Pull Request (CI):

O desenvolvedor abre um PR de feature/login para main.

O revisor (Tech Lead) abre o PR e vê as mudanças exatas nos arquivos de texto na pasta src/MeuRelatorio (ex: "Medida 'TotalVendas' foi alterada").

O pipeline de CI pode opcionalmente re-executar o pbi-tools extract para garantir que os arquivos de texto estão sincronizados com o .pbix (fail-fast).

Merge (CD):

O PR é aprovado e mesclado na branch main.

Isso dispara o pipeline de CD.

O runner do pipeline (ex: Azure DevOps Agent, Jenkins) executa um script PowerShell.

O script PowerShell usa o módulo ReportingServicesTools para se autenticar na API v2.0 do PBIRS e publicar o arquivo .pbix da branch main no servidor.

[Imagem de um diagrama de fluxo de CI/CD do Git para o Power BI Report Server]

3. Pré-requisitos

Git: Um repositório Git (Azure Repos, GitHub, GitLab, etc.).

CI/CD Tool: Um orquestrador de pipeline (Azure DevOps, Jenkins, GitLab CI).

Runner/Agente: Um agente de build que possa executar PowerShell e tenha acesso de rede ao PBIRS.

Power BI Report Server: Instalado e configurado.

Permissões:

O Agente de Build precisa de uma conta de serviço (ou credenciais) com permissão para publicar no PBIRS (normalmente "Publicador" ou "Administrador de Conteúdo").

Acesso para instalar o .NET SDK (necessário para pbi-tools).

Ferramentas (no Agente de Build):

.NET SDK (6.0 ou superior): Necessário para pbi-tools.

pbi-tools: Instalado como ferramenta global: dotnet tool install -g pbi-tools.

ReportingServicesTools: Módulo PowerShell: Install-Module -Name ReportingServicesTools -Scope AllUsers.

4. Estrutura do Repositório

Seu repositório deve ter esta aparência:


```bash
/meu-repo-bi/
|
|-- .gitignore
|
|-- /src/                   <-- Relatórios descompilados (para diffs)
|   |-- /RelatorioA/
|   |   |-- Model.bim
|   |   |-- Report.json
|   |   |-- ...
|   |-- /RelatorioB/
|       |-- ...
|
|-- /pbix/                  <-- Os arquivos .pbix originais (para deploy)
|   |-- RelatorioA.pbix
|   |-- RelatorioB.pbix
|
|-- /deploy/                <-- Scripts de deploy
|   |-- deploy-pbirs.ps1
|
|-- azure-pipelines.yml     <-- Definição do pipeline (ou Jenkinsfile, etc.)
```

``.gitignore``

```bash
# Cache do pbi-tools
.pbi-tools.cache

# Arquivos PBIT (geralmente não são versionados se o PBIX é a fonte)
*.pbit
```

5. Fase 1: O Pipeline de CI (Pull Request)

O objetivo aqui é a validação e visualização da mudança.

O pipeline (ex: azure-pipelines.yml) no evento de Pull Request deve:

Instalar Ferramentas:

Garantir que .NET SDK esteja presente.

dotnet tool install -g pbi-tools

Validar Sincronia (Opcional, mas recomendado):

Para cada .pbix no commit, execute pbi-tools extract.

Use git diff --exit-code para verificar se a extração gerou alguma mudança nos arquivos de texto.

Se houver mudanças, significa que o desenvolvedor esqueceu de rodar o extract localmente. O pipeline deve falhar, forçando o desenvolvedor a commitar os arquivos extraídos corretos.

Revisão:

O revisor agora pode ver o "diff" legível no PR.

6. Fase 2: O Pipeline de CD (Merge)

O objetivo aqui é a publicação automática.

O pipeline no evento de Merge na branch main deve:

Instalar Ferramentas:

Garantir que o módulo PowerShell ReportingServicesTools esteja disponível.

Executar Script de Deploy:

Chamar o script deploy/deploy-pbirs.ps1 passando os parâmetros necessários.

Script Chave: deploy/deploy-pbirs.ps1

Este é o script que usa o módulo ReportingServicesTools para interagir com a API v2.0 do PBIRS.

```bash
<#
.SYNOPSIS
    Publica um arquivo .pbix no Power BI Report Server.
    Este script usa o módulo ReportingServicesTools que encapsula a API v2.0.
#>
param(
    [Parameter(Mandatory=$true)]
    [string]$PbixFilePath,      # Caminho para o .pbix (ex: "$(Build.SourcesDirectory)/pbix/RelatorioA.pbix")

    [Parameter(Mandatory=$true)]
    [string]$ReportServerUri,   # URI do servidor (ex: "http://meu-pbirs/reports")

    [Parameter(Mandatory=$true)]
    [string]$TargetFolder,      # Pasta de destino no PBIRS (ex: "/Vendas/Relatórios Principais")

    [Parameter(Mandatory=$false)]
    [string]$ReportName         # Nome do relatório no servidor (se omitido, usa o nome do arquivo)
)

# Verifica se o módulo está instalado
if (-not (Get-Module -ListAvailable -Name ReportingServicesTools)) {
    Write-Error "Módulo ReportingServicesTools não encontrado. Instale com: Install-Module ReportingServicesTools"
    exit 1
}

# Se o nome do relatório não for fornecido, extrai do nome do arquivo
if ([string]::IsNullOrEmpty($ReportName)) {
    $ReportName = [System.IO.Path]::GetFileNameWithoutExtension($PbixFilePath)
}

Write-Host "Iniciando publicação..."
Write-Host "  Arquivo: $PbixFilePath"
Write-Host "  Servidor: $ReportServerUri"
Write-Host "  Destino: $TargetFolder"
Write-Host "  Nome: $ReportName"

Try {
    # 1. Conectar ao servidor (usará as credenciais do Windows do agente/usuário)
    # Para automação, o Agente de Build deve rodar com uma conta de serviço que tenha permissão.
    $session = New-RsRestSession -ReportServerUri $ReportServerUri
    
    # 2. Verificar se a pasta de destino existe
    if (-not (Test-RsRestCatalogItem -RsSession $session -Path $TargetFolder -ItemType 'Folder')) {
        Write-Host "Pasta '$TargetFolder' não encontrada. Criando..."
        New-RsRestFolder -RsSession $session -Path (Split-Path $TargetFolder) -Name (Split-Path $TargetFolder -Leaf)
    }

    # 3. Publicar o relatório (O COMANDO CHAVE)
    # Este comando usa a API v2.0 (/PowerBIReports) por baixo dos panos.
    Write-RsRestCatalogItem -RsSession $session `
        -Path $PbixFilePath `
        -Destination $TargetFolder `
        -Name $ReportName `
        -Overwrite
        
    Write-Host "SUCESSO: Relatório '$ReportName' publicado em '$TargetFolder'."

} Catch {
    Write-Error "FALHA: Não foi possível publicar o relatório. Erro: $_"
    # Lança o erro para que o pipeline de CI/CD falhe
    exit 1
}
```

7. Considerações de Segurança (Importante)

Credenciais: O script acima usa autenticação integrada do Windows (UseDefaultCredentials). Isso significa que a conta de serviço que executa o Agente de Build do seu pipeline (Azure DevOps Agent, Jenkins, etc.) deve ter permissão de "Publicador" (ou superior) no PBIRS.

Alternativa (Credenciais Explícitas): Se o agente não puder usar autenticação integrada, você pode usar Get-Credential e passar para New-RsRestSession. Nunca armazene senhas em texto puro no script. Use o gerenciador de segredos da sua ferramenta de CI/CD (ex: Azure Key Vault, Jenkins Credentials, GitHub Secrets).

```bash
# Exemplo com credenciais explícitas (NÃO RECOMENDADO SEM SEGREDOS)
$username = "dominio\usuario"
$password = "sua-senha-aqui" | ConvertTo-SecureString -AsPlainText -Force
$creds = New-Object System.Management.Automation.PSCredential($username, $password)

$session = New-RsRestSession -ReportServerUri $ReportServerUri -Credential $creds
```