# Introdução à Infraestrutura como Código (IaC)

## Visão Geral

A Infraestrutura como Código (IaC) é uma prática fundamental no DevOps moderno que permite gerenciar e provisionar recursos de infraestrutura através de arquivos de configuração legíveis por máquina, em vez de processos manuais ou ferramentas interativas.

## Alinhamento com TOGAF

Este documento segue os princípios do TOGAF (The Open Group Architecture Framework), organizando o conteúdo nas seguintes camadas arquiteturais:

```
┌─────────────────────────────────────────────────────────┐
│                  Arquitetura de Negócios                │
│    (Objetivos estratégicos, requisitos de negócio)      │
├─────────────────────────────────────────────────────────┤
│                  Arquitetura de Dados                   │
│        (Modelos de dados, fluxos de informação)         │
├─────────────────────────────────────────────────────────┤
│                Arquitetura de Aplicações                │
│      (Sistemas, serviços e integrações)                 │
├─────────────────────────────────────────────────────────┤
│                Arquitetura de Tecnologia                │
│   (Infraestrutura, redes, plataformas cloud)            │
└─────────────────────────────────────────────────────────┘
```

## O que é IaC?

IaC é a prática de definir infraestrutura usando código declarativo ou imperativo, permitindo:

- **Versionamento**: Toda a infraestrutura é rastreável via controle de versão
- **Repetibilidade**: Ambientes idênticos podem ser criados consistentemente
- **Automatização**: Redução de erros humanos e aumento de velocidade
- **Documentação**: O código serve como documentação viva da infraestrutura

## Benefícios Principais

| Benefício | Descrição |
|-----------|-----------|
| **Consistência** | Elimina variações entre ambientes (dev, staging, prod) |
| **Velocidade** | Provisionamento em minutos ao invés de dias |
| **Escalabilidade** | Facilita escalar recursos conforme demanda |
| **Recuperação** | Permite reconstruir infraestrutura rapidamente em caso de desastres |
| **Auditoria** | Histórico completo de mudanças via Git |
| **Colaboração** | Equipes podem revisar e aprovar mudanças via PRs |

## Paradigmas de IaC

### Declarativo vs Imperativo

```
┌────────────────────────────┬────────────────────────────┐
│        Declarativo         │        Imperativo          │
├────────────────────────────┼────────────────────────────┤
│ Define o estado desejado   │ Define os passos para      │
│                            │ alcançar o estado          │
├────────────────────────────┼────────────────────────────┤
│ Terraform, CloudFormation  │ Ansible, Bash scripts      │
├────────────────────────────┼────────────────────────────┤
│ "O que" você quer          │ "Como" fazer               │
└────────────────────────────┴────────────────────────────┘
```

### Mutable vs Immutable

- **Infraestrutura Mutável**: Servidores são atualizados in-place
- **Infraestrutura Imutável**: Servidores são substituídos a cada mudança (recomendado)

## Ciclo de Vida IaC

```
     ┌─────────┐
     │  Plan   │ ──── Planejamento e design
     └────┬────┘
          │
          ▼
     ┌─────────┐
     │  Code   │ ──── Desenvolvimento do código
     └────┬────┘
          │
          ▼
     ┌─────────┐
     │  Test   │ ──── Validação e testes
     └────┬────┘
          │
          ▼
     ┌─────────┐
     │ Deploy  │ ──── Aplicação das mudanças
     └────┬────┘
          │
          ▼
     ┌─────────┐
     │ Monitor │ ──── Monitoramento contínuo
     └────┬────┘
          │
          └──────────────┐
                         ▼
                    ┌─────────┐
                    │ Iterate │ ──── Melhoria contínua
                    └─────────┘
```

## Primeiros Passos

1. **Escolha uma ferramenta**: Avalie suas necessidades (multi-cloud, vendor-specific)
2. **Configure o controle de versão**: Use Git para versionar seu código
3. **Estabeleça pipelines CI/CD**: Automatize validação e deployment
4. **Defina padrões**: Crie módulos reutilizáveis e convenções de nomenclatura
5. **Implemente gradualmente**: Comece com recursos simples e evolua

## Próximos Passos

- Explore as [Ferramentas de IaC](./tools-overview.md) disponíveis
- Consulte as [Melhores Práticas](./best-practices.md) recomendadas
- Veja [exemplos práticos](./examples/) de implementação