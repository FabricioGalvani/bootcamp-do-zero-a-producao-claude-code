# Guia de Configuração — Agente Validador Terraform

## 1. Adicionar o agent ao repositório

Copie o arquivo `terraform-validator-pt-br.agent.md` (ou a versão em inglês
`terraform-validator.agent.md`) para o seu repositório:

```
.github/
  agents/
    terraform-validator.agent.md
```

> **Dica:** O Copilot entende instruções em português, então você pode usar
> a versão PT-BR sem problemas. Se preferir manter o agent em inglês
> (mais comum em times mistos), use a versão original.

## 2. Configurar o Terraform MCP Server no VS Code

No seu `.vscode/settings.json` (ou nas configurações globais), adicione:

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

> Se você não usa HCP Terraform / Terraform Cloud, pode remover a linha
> do `TERRAFORM_CLOUD_TOKEN`. O MCP server ainda funciona para consultas
> ao registry público.

## 3. Instalar dependências

```bash
# O terraform-mcp-server é baixado automaticamente via npx,
# mas você precisa ter o Node.js 18+ instalado.
node --version  # deve ser >= 18

# Opcional: instale o Terraform CLI para validação local
terraform --version
```

## 4. Como usar o agent

### No VS Code (Copilot Chat)

Abra o Copilot Chat e invoque o agent:

```
@terraform-validator valide os arquivos em infra/dev/
```

```
@terraform-validator verifique se os nomes de recursos em infra/prod/ estão corretos
```

```
@terraform-validator analise as regras de segurança dos arquivos de produção
```

### No GitHub.com (Copilot Coding Agent)

Se você usa o Copilot Coding Agent (cloud), o agent será detectado
automaticamente quando estiver em `.github/agents/`.

### No GitHub Copilot CLI

```bash
gh copilot suggest --agent terraform-validator "validar infra/hom/"
```

## 5. Personalização

### Adicionar novos ambientes

Edite a seção "Detecção de Ambiente" no agent para incluir novos
ambientes (ex: `staging`, `sandbox`, `qa`).

### Adicionar regras de naming

Edite a seção "Regras de Convenção de Nomenclatura de Recursos" para
adicionar prefixos de projeto, região, time, etc. Por exemplo:

```
O nome de todo recurso deve seguir o padrão:
<projeto>-<componente>-<ambiente>

Exemplo: myapp-api-gateway-dev
```

### Adicionar tags obrigatórias

Edite a seção "Tags Obrigatórias" para incluir tags específicas da
sua organização. Exemplo:

```hcl
tags = {
  Environment = "<dev|hom|prod>"
  Project     = "<nome-do-projeto>"
  ManagedBy   = "terraform"
  CostCenter  = "<centro-de-custo>"
  Team        = "<nome-do-time>"
  Owner       = "<email-do-responsavel>"
}
```

### Adicionar validações customizadas

Você pode adicionar qualquer regra nova ao arquivo `.agent.md`.
Basta escrever em linguagem natural o que o agent deve verificar.
Por exemplo:

```markdown
## 8. Regras de Rede

Sinalize como ERRO se:
- Uma VPC em dev tiver o CIDR `10.0.0.0/16` (reservado para prod).
- Uma subnet pública existir em prod sem NAT Gateway associado.
```

## 6. Solução de Problemas

| Problema | Solução |
|----------|---------|
| Agent não aparece no Copilot Chat | Verifique se o arquivo está em `.github/agents/` e tem extensão `.agent.md` |
| MCP server não conecta | Verifique se `npx` está disponível e Node.js >= 18 |
| Agent não valida regras customizadas | Reinicie o VS Code para recarregar o agent |
| Token do Terraform Cloud inválido | Configure a variável de ambiente `TERRAFORM_CLOUD_TOKEN` ou remova do settings |
