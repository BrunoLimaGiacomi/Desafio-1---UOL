# Desafio 1 - Avaliação e Redesenho de Arquitetura (Security Design Review)

# Inventário

### Load Balancers
- Quantidade: 1

### Instâncias EC2
- Quantidade: 2

### Bancos de Dados (RDS)
- Quantidade: 1

### Security Groups
- Quantidade: 1  

### Usuários IAM
- Quantidade: Não especificado, mas são locais

### Integrações
- Quantidade: 2  
- Detalhamento:  
  - On-Prem  
  - Terceiro (Backup)  

### Protocolos Utilizados

- HTTP/80
- SSH/22  
- MySQL/3306  
- FTP/21 

---

# Classificação de riscos

##  Riscos Urgentes

### RDS publicamente acessível e sem criptografia em repouso > vazamento de dados pessoais > interceptação/violação de dados e consequentemente da LGPD
- Probabilidade: Alta  
- Impacto: Crítico  
- Classificação de Risco: Urgente  

### Backup via FTP (sem criptografia/plaintext) contendo dados pessoais > interceptação/violação de dados e consequentemente da LGPD
- Probabilidade: Alta  
- Impacto: Crítico  
- Classificação de Risco: Urgente  

### Security Groups excessivamente permissivos
- Probabilidade: Alta  
- Impacto: Crítico  
- Classificação de Risco: Urgente  


### Tráfego web sem TLS > Man in the Middle, credenciais expostas
- Probabilidade: Alta  
- Impacto: Crítico  
- Classificação de Risco: Urgente  

---

##  Riscos Altos

### SSH direto pela Internet (credenciais) > acesso não autorizado/movimento lateral
- Probabilidade: Alta  
- Impacto: Alto  
- Classificação de Risco: Alto  

### Basic Auth API exposta > credenciais fracas, reutilizáveis e sem rotação/expiração
- Probabilidade: Alta  
- Impacto: Alto  
- Classificação de Risco: Alto  

### Ausência de WAF > Maior probabilidade de exploração de OWASP Top 10
- Probabilidade: Alta  
- Impacto: Alto  
- Classificação de Risco: Alto  

### Uso de usuários IAM locais sem federação, MFA ou uma renovação de credenciais explícita
- Probabilidade: Alta  
- Impacto: Alto  
- Classificação de Risco: Alto  

### API exposta na Internet sem controle de origem
- Probabilidade: Alta  
- Impacto: Alto  
- Classificação de Risco: Alto  

---

##  Riscos Médios

### Ausência de CloudTrail/registro de LOGs
- Probabilidade: Mediano  
- Impacto: Alto  
- Classificação de Risco: Mediano  

### Ausência de GuardDuty/IDS
- Probabilidade: Mediano  
- Impacto: Alto  
- Classificação de Risco: Mediano

# Decisões de Arquitetura

## Web Tier para Subnets Privadas

**O que muda:**  
- Mover as instâncias EC2 para uma subnet privada
- ALB permanece em uma subnet pública

**Por quê:**  
- Reduz a superfície de ataque  
- Impede acesso direto via Internet às cargas de aplicação  

---

## ALB Somente HTTPS + ACM + WAF

**O que muda:**  
- Listener na porta 443 com certificado gerenciado pelo ACM  
- Redirecionamento automático 80 > 443  
- AWS WAF em frente ao ALB com regras gerenciadas (OWASP) e rate limiting  

**Por quê:**  
- Garante confidencialidade e integridade do tráfego cliente > aplicação  
- Proteção contra ataques de camada de aplicação  

---

## RDS Privado, Multi-AZ e Criptografado (KMS)

**O que muda:**  
- Deixa de ser público  
- Habilitar Multi-AZ  
- Ativar criptografia com KMS para banco e snapshots  
- Exigir TLS nas conexões  

**Por quê:**  
- Elimina exposição direta do banco  
- Protege dados em repouso  
- Melhora disponibilidade  

---

## Remoção de SSH Público/Administração via SSM

**O que muda:**  
- Bloquear porta 22 em Security Groups públicos  
- Utilizar AWS Systems Manager Session Manager para acesso operacional  
- Bastion host apenas se estritamente necessário via VPN + MFA  

**Por quê:**  
- Reduz risco de acesso remoto não autorizado  
- Fornece auditoria de sessões  

---

## Segurança de Rede com SG-to-SG e VPC Endpoints

**O que muda:**  
- Security Groups restritos (ALB > App, App > DB)  
- Criar VPC Endpoints para S3, Secrets Manager e KMS

**Por quê:**  
- Aplicação do princípio do menor privilégio  
- Redução de tráfego de saída para Internet pública  

---

## Substituição de FTP por S3

**O que muda:**  
- Descontinuar uso de FTP plaintext  
- Usar S3 com SSE-KMS e lambda para sftp push.

**Por quê:**  
- Protege dados em trânsito  
- Permite auditoria e controles de acesso mais robustos  

---

## Gerenciamento de Segredos com Secrets Manager e IAM Roles

**O que muda:**  
- Migrar credenciais para AWS Secrets Manager  
- Utilizar IAM Roles para aplicações (eliminando chaves estáticas)  

**Por quê:**  
- Permite rotação automática de segredos  
- Reduz exposição de credenciais  

---

## Identidade Federada: Entra ID > AWS IAM Identity Center

**O que muda:**  
- Integrar Azure AD (Entra) com AWS IAM Identity Center (SSO/SAML/OIDC)  
- Policies e Roles baseadas em função  
- MFA obrigatório  

**Por quê:**  
- Centraliza identidade  
- Reduz gestão manual de usuários IAM  
- Melhora aderência a requisitos de compliance  


---

## Revisão do Modelo de Exposição e Autenticação da API

### O que muda

- A API deixa de utilizar autenticação básica (Basic Auth)
- Implementação de autenticação baseada em token (ex.: OAuth2 / OIDC)
- Restrição de origem por meio de rede privada, VPN ou túnel criptografado

### Por quê:

- Elimina o uso de credenciais estáticas reutilizáveis
- Permite revogação granular e expiração controlada de credenciais
- Reduz o risco de ataques de força bruta e reuso da credencial

---

## Observabilidade e Detecção

### O que muda

- Implementação de trilha de auditoria centralizada
- Habilitação de monitoramento comportamental
- Registro estruturado e retenção controlada de logs de rede e aplicação
- Integração com mecanismos de alerta e correlação de eventos

### Por quê:

- Reduz o MTTD de incidentes
- Permite investigação forense com evidências confiáveis, Melhorando a capacidade de resposta e contenção de ameaças

---

# Conectividade AWS ↔ On-Premises e Mecanismos de Autenticação/Autorização da API

## Conectividade

- Implementar Site-to-Site VPN (IPSec) entre on-premises e VPC.
- Roteamento:
  - Sub-redes privadas para serviços internos.
  - API acessível via endpoint privado (API Gateway Private) ou ALB interno através da VPN.

## Modelo de Exposição da API

- Não expor API sensível diretamente na Internet.
- Consumo via VPN ou private endpoint.
- Caso exposição pública da API seja necessária:
  - Proteger com CloudFront + WAF.
  - Aplicar rate limiting rigoroso.
  - Implementar IP allowlists.

---

## Autenticação/Autorização da API

### Protocolos

- OAuth 2.0/OIDC no Entra ID.

### Fluxo

1. Cliente on-prem obtém token no Entra.
2. Envia JWT no header `Authorization`.
3. API valida token (API Gateway + Lambda Authorizer).
4. Autorização baseada em scopes.

---

# Proteção de Dados e Gestão de Segredos

## Dados em Trânsito

- TLS 1.2+ obrigatório no ALB.
- VPN.
- Forçar TLS entre aplicação > RDS (verificação de certificado).

---

## Dados em Repouso

- RDS:
  - Criptografia habilitada com AWS KMS.
- Snapshots RDS:
  - Criptografados automaticamente (herda do KMS).
- Logs e backups:
  - Bucket criptografado.
  - Lifecycle para retenção.

---

## Gestão de Segredos e Chaves

- AWS Secrets Manager:
  - Armazenar credenciais de banco, APIs e tokens externos.
  - Habilitar rotação automática.
- AWS KMS:
  - Gerenciar CMKs para dados críticos.
  - Restringir via IAM + key policies.
  - Logging via CloudTrail.
- Evitar credenciais em variáveis/arquivos/AMIs.
- Utilizar IAM Roles (instance profile / task role).
- Compartilhamento controlado:
  - Presigned URLs.
  - Validade curta + restrição por IP quando possível.

---

# Observabilidade e Governança

## Auditoria e Logs

- CloudTrail global ativado.
- Centralização em conta de logging (S3).
- Retenção configurada e proteção contra deleção.
- CloudWatch Logs:
  - Aplicação.
  - ALB.
  - VPC Flow Logs.
- VPC Flow Logs:
  - Habilitados em subnets críticas.
  - Envio para CloudWatch/S3.

---

## Detecção e Correlação

- GuardDuty habilitado (findings > Security Hub).
- Security Hub como centralizador da organização.

---

## Monitoramento e Alertas

- CloudWatch Alarms:
  - Disponibilidade ALB.
  - Latência.
  - Uso elevado de DB.
  - Aumento de egress.

---

## Governança

- Obrigar MFA.
- Identity Center com session policies.
- Revisões trimestrais:
  - IAM Roles.
  - Security Groups.
- Patch management via SSM Patch Manager.
- Pipeline de AMI imutável.
- Política formal de backup e retenção.
- Auditoria de acesso a backups.

---

# Roadmap, Rollback e Riscos Residuais

## Roadmap

### Fase 0 — Preparação (0–5 dias)

- Inventário final.
- Snapshots RDS e AMIs.
- Criar conta e bucket de logging.

---

### Fase 1 — Urgente (0–14 dias)

- Ajustar Security Groups (remover 0.0.0.0/0 das portas 22/3306/21).
- Tornar RDS privado e habilitar criptografia KMS (com snapshot prévia).
- Bloquear/pausar FTP.
- Habilitar HTTPS no ALB.
- Redirect 80 > 443.

---

### Fase 2 — Curto Prazo (15–40 dias)

- Provisionar SSM e Remover SSH público.
- Migrar credenciais para Secrets Manager e rotacionar as secrets.
- Provisionar WAF.
- Criar VPC Endpoints:
  - Secrets Manager.
  - KMS.
  - CloudWatch.

---

### Fase 3 — Médio Prazo (41–70 dias)

- Implementar Site-to-Site VPN (IPSec) com on-prem.
- Migrar API para private endpoint ou ALB interno.
- Implementar OAuth2/OIDC (Entra ID) e integrar com IAM Identity Center.
- Habilitar GuardDuty, Security Hub e VPC Flow Logs centralizados.

---

### Fase 4 — Maturidade (>71 dias)

- Documentar todo o processo.
- Aumentar a divulgação das politicas e arquiteturas padrões de segurança para evitar novos casos no futuro.
- Revisão do processo de CI/CD.
- Estudar possibilidade de adquirir CIAM para todas as aplicações que usuários terceiros fazem login.

---

## Rollback

- Sempre antes de mudanças críticas: snapshots RDS, AMIs e registro de estado.
- Documentar regras de SG originais para reverter rapidamente.
- Para migração de autenticação: manter endpoint legacy com failback via DNS/ALB routing até validação completa.

---

## Riscos Residuais e Mitigação

- Vulnerabilidade zero-day:
  - Patching ágil.
  - WAF.
  - Inspeção runtime.
- Credencial de terceiro comprometida:
  - Rotação periódica.
  - Escopo mínimo.
  - Cláusulas contratuais.
- Erro de configuração IAM:
  - Revisão por pares.
  - Least privilege.
  - IAM Access Analyzer.
- Dependência do Entra ID:
  - Plano de contingência temporário.
  - Controles reforçados.

---

# Assunções

- Temos autorização para alterar rede e recursos AWS.
- Temos o Entra ID disponível para federation SAML/OIDC.
- Janela de manutenção aceitável para cutovers.
- Orçamento para WAF, NAT, S3 com SSM-KMS, logs, GuardDuty aprovado ou negociável.
- Ambiente de teste/staging disponível para validações antes de ir para produção.
- O Parceiro do backup opera fora da AWS.