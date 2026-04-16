# Setup Guide — Terraform Validator Agent

## 1. Colocar o agent no repositorio

Copie o arquivo `terraform-validator.agent.md` para o seu repositorio:

```
.github/
  agents/
    terraform-validator.agent.md
```

## 2. Configurar o Terraform MCP Server no VS Code

No seu `.vscode/settings.json` (ou no settings global), adicione:

```json
{
  "github.copilot.chat.agent.mcp.servers": {
    "terraform-mcp": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "terraform-mcp-server"],
      "env": {
        "TERRAFORM_CLOUD_TOKEN": "${env:TERRAFORM_CLOUD_TOKEN}"
      }
    }
  }
}
```

> Se voce nao usa HCP Terraform / Terraform Cloud, pode remover a linha
> do `TERRAFORM_CLOUD_TOKEN`. O MCP server ainda funciona para consultas
> ao registry publico.

## 3. Instalar dependencias

```bash
# O terraform-mcp-server e baixado automaticamente via npx,
# mas voce precisa ter o Node.js 18+ instalado.
node --version  # deve ser >= 18

# Opcional: instale o Terraform CLI para validacao local
terraform --version
```

## 4. Usar o agent

### No VS Code (Copilot Chat)

Abra o Copilot Chat e invoque o agent:

```
@terraform-validator valide os arquivos em infra/dev/
```

```
@terraform-validator verifique se os nomes de recursos em infra/prod/ estao corretos
```

### No GitHub.com (Copilot Coding Agent)

Se voce usa o Copilot Coding Agent (cloud), o agent sera detectado
automaticamente quando estiver em `.github/agents/`.

### No GitHub Copilot CLI

```bash
gh copilot suggest --agent terraform-validator "validate infra/hom/"
```

## 5. Customizacao

### Adicionar novos ambientes

Edite a secao "Environment Detection" no agent para incluir novos
ambientes (ex: `staging`, `sandbox`).

### Adicionar regras de naming

Edite a secao "Resource Naming Convention Rules" para adicionar
prefixos de projeto, regiao, etc.

### Adicionar tags obrigatorias

Edite a secao "Required Tags" para incluir tags especificas da
sua organizacao (ex: `CostCenter`, `Team`, `Owner`).
