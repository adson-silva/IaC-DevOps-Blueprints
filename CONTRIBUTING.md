# CONTRIBUTING

Obrigado por contribuir com o IaC DevOps Blueprints! Este documento descreve como abrir issues, enviar pull requests e as práticas recomendadas para colaborar neste repositório.

Sumário rápido
- Abra uma Issue para discutir grandes mudanças antes de trabalhar nelas.
- Crie branches com nomes claros: feature/, fix/, docs/, chore/.
- Siga o padrão de commits (Conventional Commits).
- Inclua testes/validadores e documentação local do módulo quando aplicável.
- Use linters e scanners em alterações que afetem IaC.

1. Como começar
- Forke o repositório e clone para sua máquina:
  - git clone https://github.com/adson-silva/IaC-DevOps-Blueprints.git
  - git remote add upstream https://github.com/adson-silva/IaC-DevOps-Blueprints.git
- Crie uma branch descritiva:
  - git checkout -b feature/minha-nova-funcionalidade
  - git checkout -b fix/bug-no-modulo-vnet
  - git checkout -b docs/atualiza-readme-azure

2. Abrir uma Issue
- Use Issues para:
  - Reportar bugs (incluir passos para reproduzir).
  - Propor novas funcionalidades.
  - Pedir orientação ou roadmap.
- Inclua título claro, descrição, passos para reproduzir, logs relevantes e impacto esperado.
- Marque a área (Azure/AWS/GCP/common/docs/ci) no corpo da issue para facilitar triagem.

3. Trabalhar em um Pull Request
- Antes de implementar, é recomendado abrir uma Issue para alinhar o escopo em mudanças grandes.
- Mantenha PRs pequenos e focados.
- Inclua no PR:
  - Resumo do que foi alterado.
  - Racional técnico (por que a mudança é necessária).
  - Instruções para testar localmente (comandos).
  - Checklist preenchido (veja seção abaixo).
- Etiquetas sugeridas: enhancement, bug, docs, ci, security.

Template de checklist para PR (copiar para o corpo do PR)
- [ ] Descrevi a mudança e o motivo.
- [ ] Atualizei/adicionei documentação local no módulo afetado (README do módulo).
- [ ] Adicionei/atualizei testes e os testes passam.
- [ ] Rodei linters e scanners (tflint, tfsec, checkov, etc).
- [ ] Segui o padrão de commits (Conventional Commits).
- [ ] Confirmei que não há segredos ou credenciais commited.
- [ ] Pipeline CI (se configurado) passa.

4. Padrão de commits
Adotamos Conventional Commits para clareza nos históricos e automações:
- formato: <type>(escopo?): <descrição>
- tipos comuns:
  - feat: nova funcionalidade
  - fix: correção de bug
  - docs: documentação
  - chore: manutenção, scripts, configs
  - ci: alterações em CI
  - refactor: refatoração sem mudança de comportamento
- exemplo: feat(azure-quickstart): adicionar exemplo simple-vnet

5. Linters, scanners e validação local
- Terraform:
  - terraform init
  - terraform validate
  - terraform fmt -check
  - tflint
  - tfsec
  - checkov
- Exemplo de execução rápida (na raiz de um módulo Terraform):
  - cd path/to/module
  - terraform init -backend=false
  - terraform fmt && terraform validate
  - tflint --init && tflint
  - tfsec .
- Scripts de validação podem ser adicionados em /scripts/ e referenciados no README do módulo.

6. Testes
- Inclua testes quando aplicável (unitários, integração de módulos).
- Para módulos Terraform, documente como validar/destruir infra de teste com workspaces de teste ou projetos isolados.
- Automatize verificações essenciais (lint, security scan) no CI.

7. Revisão de código e Merge
- Pelo menos uma revisão por outro maintainer é recomendada para mudanças significativas.
- PRs aprovados por revisores e com CI verde podem ser mesclados.
- Para mudanças críticas em segurança/infra produção, solicitar revisão de um maintainer sênior e executar um runbook de validação pós-deploy.

8. Segurança e reporte responsável
- Não publique vulnerabilidades publicamente em Issues. Envie um e-mail para o mantenedor (perfil no GitHub) ou abra Issue com rótulo private/security (se o repositório oferecer).
- Nunca commite segredos. Se um segredo for acidentalmente commited, reporte imediatamente para revogá-lo e remova o histórico (seguir instruções do GitHub sobre tokens expostos).

9. Documentação de módulos
- Cada pasta de módulo deve ter README com:
  - Propósito do módulo
  - Variáveis/outputs documentados
  - Exemplo de uso
  - Requisitos/versões
  - Passos de teste
- Atualize a documentação do módulo ao alterar interfaces (variáveis/outputs).

10. Padrões de estrutura e nomes
- Mantenha consistência com a estrutura proposta no README (azure/, aws/, gcp/, common/, examples/, docs/).
- Nomeie módulos e arquivos de forma clara e em inglês para interoperabilidade (mas documentação e README principal em Português conforme decisão do projeto).

11. Templates úteis (copie para .github/ quando desejar)
- Exemplo de template de Issue (issue_template.md)
---
Título: [BUG|FEATURE] Resumo curto
Descrição:
- Área afetada: (Azure/AWS/GCP/common/docs/ci)
- Descrição detalhada:
- Passos para reproduzir:
- Resultado atual:
- Resultado esperado:
- Impacto/gravidade:
- Sugestão de solução (se souber):
---

- Exemplo de template de Pull Request (pull_request_template.md)
---
## Descrição
(Explique o que foi alterado e por que.)

## Tipo de mudança
- [ ] bugfix
- [ ] feature
- [ ] docs
- [ ] ci
- [ ] outros

## Checklist
- [ ] Documentação atualizada
- [ ] Testes adicionados/atualizados
- [ ] Linters/scanners executados
- [ ] Commit messages seguindo Conventional Commits
- [ ] Pipeline CI verde
---

12. Governança e contato
- Mantenedores: adson-silva
- Para decisões de arquitetura ou roadmap, abra uma Issue com o rótulo proposal/arquitetura.
- Para dúvidas rápidas, mencione o maintainer nas issues/PRs (use nome de usuário do GitHub).

13. Código de Conduta
- Recomendamos adicionar CODE_OF_CONDUCT.md ao repositório. Até lá, espera-se comportamento respeitoso e profissional em discussões e revisões.

14. Observações finais
- Obrigado por ajudar a tornar este projeto melhor. Trabalhos pequenos e bem documentados têm mais chance de revisão rápida.
- Se quiser, posso gerar arquivos PR/Issue templates e um CONTRIBUTING.md já pronto no repo em um PR — diga se quer que eu crie a branch e abra o PR.