# Azure Pipelines




```powershell
# publicar_painel.ps1
param (
    # URL do seu servidor. Ex: http://meuservidor/reports
    [string]$ServerUrl,
    
    # A pasta "raiz" no seu repositório Git que contém os paineis.
    # Ex: "Paineis"
    [string]$BaseRepoFolder,
    
    # A pasta "raiz" no servidor onde a estrutura será espelhada.
    # Ex: "/Paineis_Producao"
    [string]$BaseServerFolder 
)

# 1. Instalar o módulo (só na primeira vez ou se não existir no agente)
Write-Host "Verificando módulo ReportingServicesTools..."
if (-not (Get-Module -ListAvailable -Name "ReportingServicesTools")) {
    Write-Host "Instalando módulo ReportingServicesTools..."
    try {
        Install-Module -Name ReportingServicesTools -Scope CurrentUser -Force -AllowClobber -Confirm:$false -ErrorAction Stop
    } catch {
        Write-Error "Falha ao instalar módulo. Tente instalar manualmente no agente com 'Install-Module ReportingServicesTools'."
        exit 1
    }
}

# 2. Definir os caminhos de origem e destino
# $env:BUILD_SOURCESDIRECTORY é a variável do Azure DevOps para a raiz do repo
$SourcePath = Join-Path $env:BUILD_SOURCESDIRECTORY $BaseRepoFolder
Write-Host "Pasta de origem (Repositório): $SourcePath"
Write-Host "Pasta de destino (Servidor): $BaseServerFolder"

# 3. Encontrar TODOS os arquivos .pbix na pasta de origem
$allPbixFiles = Get-ChildItem -Path $SourcePath -Filter "*.pbix" -Recurse

if ($allPbixFiles.Count -eq 0) {
    Write-Warning "Nenhum arquivo .pbix encontrado em '$SourcePath'."
    exit 0 # Sai com sucesso, pois não há nada a fazer
}

Write-Host "Encontrados $($allPbixFiles.Count) arquivos .pbix. Iniciando publicação..."

# 4. Loop para publicar cada arquivo
$failedFiles = @()

foreach ($pbix in $allPbixFiles) {
    
    $pbixName = $pbix.Name
    
    # 5. Calcular o caminho de destino no servidor
    
    # Pega o caminho do diretório do arquivo (ex: D:\agent\_work\1\s\Paineis\Vendas)
    $fullSourceFolderPath = $pbix.Directory.FullName 
    
    # Remove a parte "base" do caminho (ex: \Vendas)
    $relativeFolderPath = $fullSourceFolderPath.Substring($SourcePath.Length).Replace("\", "/")
    
    # Monta o caminho final no servidor (ex: /Paineis_Producao/Vendas)
    $targetServerFolder = ($BaseServerFolder + $relativeFolderPath) -replace '//', '/' # Limpa barras duplas
    
    Write-Host "--------------------------------"
    Write-Host "Processando: $pbixName"
    Write-Host "  -> Pasta de Destino: $targetServerFolder"

    try {
        # 6. Garantir que a pasta de destino exista no servidor
        # Este comando cria a estrutura de pastas recursivamente (ex: /Paineis_Producao/Vendas)
        Write-RsFolder -RsFolder $targetServerFolder -ReportPortalUri $ServerUrl -ErrorAction Stop
        Write-Host "  -> Pasta '$targetServerFolder' verificada/criada."

        # 7. Publicar o arquivo
        Write-RsCatalogItem -Path $pbix.FullName `
                            -RsFolder $targetServerFolder `
                            -ReportPortalUri $ServerUrl `
                            -OverWrite `
                            -ErrorAction Stop
                            
        Write-Host "  -> SUCESSO: '$pbixName' publicado."

    } catch {
        Write-Error "  -> FALHA ao publicar '$pbixName': $_"
        $failedFiles += $pbixName
    }
}

# 8. Verificar se houve falhas
if ($failedFiles.Count -gt 0) {
    Write-Error "Publicação finalizada com $($failedFiles.Count) falha(s):"
    $failedFiles | ForEach-Object { Write-Error "  - $_" }
    exit 1 # Falha o pipeline
}

Write-Host "--------------------------------"
Write-Host "Publicação de todos os paineis concluída com sucesso."
exit 0
```



```yaml
# azure-pipelines.yml
# Define o pool para usar sua máquina Windows on-premise
trigger:
- main

pool:
 name: 'Meu-Pool-OnPremise' # <-- Verifique o nome do seu pool
 demands:
  - powershell # Garante que o agente tem PowerShell
# Mapeia seu grupo de variáveis secretas
variables:
- group: 'Credenciais-PowerBI-RS' # Nome do seu Variable Group

steps:
- checkout: self # Pega o código (o .pbix e o script)

- task: PowerShell@2
  displayName: 'Executar Script PowerShell de Deploy'
  inputs:
    targetType: 'filePath' # Executa um arquivo
    filePath: 'caminho/do/script/publicar_painel.ps1' # Caminho no seu repo
    arguments: > # Argumentos para o script
      -PbixFile "$(Build.SourcesDirectory)/Paineis/PainelDeVendas.pbix"
      -ServerUrl "http://meuservidor/reports"
      -TargetFolder "Vendas/Producao"
  env:
    # Mapeia os segredos do Variable Group para o script
    PBI_USER: $(PBI_USER)
    PBI_PASS: $(PBI_PASS)
```


```bash
.
├── azure-pipelines.yml
├── Scripts/
│   └── publicar-paineis-espelho.ps1
└── Paineis/                <-- BaseRepoFolder: "Paineis"
    ├── RelatorioRoot.pbix
    ├── Vendas/
    │   ├── Vendas_Diario.pbix
    │   └── Vendas_Mensal.pbix
    └── Marketing/
        └── Custo_Aquisicao.pbix
```


Dependencias:

https://github.com/microsoft/ReportingServicesTools