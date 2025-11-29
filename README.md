# IaC DevOps Blueprints

Bem-vindo ao repositório IaC DevOps Blueprints! Este repositório serve como um hub central para projetos DevOps, com foco em Infraestrutura como Código (IaC) e compatibilidade multicloud (Azure, AWS, GCP).

## Objetivos e Escopo
- Fornecer práticas recomendadas para implementação de Infraestrutura como Código.
- Oferecer templates e tutoriais para ajudar os usuários a começar com IaC em diferentes plataformas de nuvem.
- Garantir conformidade com padrões como ISO 27001, ISO 27701, ISO 22301, NIST, ITIL e PCI DSS.
- Compartilhar implementações de referência reutilizáveis para a comunidade aproveitar.

Nosso objetivo é capacitar desenvolvedores e organizações a adotar práticas DevOps de forma eficaz, mantendo padrões de conformidade e segurança.

---

## Estrutura do repositório (proposta)
Sugestão de organização de pastas para facilitar navegação e contribuição:

- /azure/
  - quickstart/ (ex.: Bicep, ARM templates, Terraform examples)
  - modules/ (módulos reutilizáveis específicos Azure)
  - docs/ (detalhes específicos Azure)
- /aws/
  - quickstart/ (CloudFormation, CDK, Terraform)
  - modules/ (módulos reutilizáveis AWS)
  - docs/
- /gcp/
  - quickstart/ (Deployment Manager, Terraform)
  - modules/
  - docs/
- /common/
  - modules/ (módulos multi-cloud ou agnósticos)
  - templates/ (ex.: políticas, etiquetas, rotinas CI)
- /examples/
  - impl-exemplo-1/ (projeto completo demonstrativo com README)
  - impl-exemplo-2/
- /docs/
  - arquitetura.md
  - compliance-mapping.md
  - security-guidelines.md
  - how-to-contribute.md
- /ci/
  - pipelines/ (ex.: GitHub Actions, Azure Pipelines, GitLab CI)
  - workflows/ (templates)
- /scripts/
  - utils/ (scripts de validação, helpers)
- /tools/
  - linters-configs/ (pre-commit, tfsec, tflint)
  - scanner-configs/
- /assets/
  - diagramas/
  - imagens/
- README.md
- CONTRIBUTING.md (opcional — sugerido)
- CODE_OF_CONDUCT.md (opcional)
- roadmap.md

Cada pasta deve conter um README.md local explicando propósito, como executar e dependências específicas.

---

## Quickstarts (rápido)
Instruções resumidas para começar com exemplos práticos.

### Azure (Terraform)
- Pré-requisitos: Azure CLI, Terraform
- Passos:
  1. az login
  2. cd azure/quickstart/terraform/simple-vnet
  3. terraform init
  4. terraform plan -out plan.tfplan
  5. terraform apply plan.tfplan

### AWS (Terraform)
- Pré-requisitos: AWS CLI com credenciais configuradas, Terraform
- Passos:
  1. aws configure
  2. cd aws/quickstart/terraform/simple-vpc
  3. terraform init
  4. terraform apply

### GCP (Terraform)
- Pré-requisitos: gcloud, Terraform
- Passos:
  1. gcloud auth login
  2. gcloud config set project <PROJECT_ID>
  3. cd gcp/quickstart/terraform/simple-network
  4. terraform init
  5. terraform apply
