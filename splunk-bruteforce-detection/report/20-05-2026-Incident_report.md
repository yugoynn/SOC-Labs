# Incident Report — SSH Brute Force

> Baseado no framework **NIST SP 800-61** — Ciclo de Resposta a Incidentes

---

## 1. Dados Gerais

| Campo                      | Informação                                                                 |
|----------------------------|----------------------------------------------------------------------------|
| **Data do Incidente**      | 22/05/2026                                                                 |
| **Hora**                   | 10:54 às 11:05                                                             |
| **Local**                  | Host: `cancer` — IP: `192.168.3.121` — Serviço: SSH (Porta 22)             |
| **Tipo de Incidente**      | Ataque de Força Bruta via SSH (Brute Force)                                |
| **Responsável**            | Danilo Pires de Oliveira                                                   |
| **Cargo / Função**         | Analista de Segurança                                                      |
| **Pessoas Envolvidas**     | Danilo Pires de Oliveira — Analista / Pesquisador responsável pelo teste   |
| **Gravidade**              | Média                                                                      |

---

## 2. Identificação 

**Descrição do Incidente:**
Em 22/05/2026 às 10:54h, foi detectado ataque de força bruta SSH originado do IP `192.168.3.153` contra o host `cancer` (`192.168.3.121:22`). Foram registradas 23 tentativas de autenticação falhas e 3 logins aceitos. Os usuários-alvo foram `yugo` e `root`.

**Como foi Detectado:**
Detectado visualmente no dashboard "SOC Linux Monitoring" no Splunk Enterprise e analisado via logs do `/var/log/auth.log`. Query utilizada: 
`index=* penalty: "failed authentication"`.

**Sistemas / Dados Afetados:**
Possível comprometimento das contas `yugo` e `root` (3 logins aceitos durante o ataque).

---

## 3. Contenção 

**Ações Imediatas (Curto Prazo):**
O daemon `sshd[98795]` bloqueou automaticamente as conexões via mecanismo `PerSourcePenalties`, derrubando múltiplas conexões simultâneas do IP atacante.

**Ações Temporárias:**
Monitoramento contínuo via Splunk — identificação de novos picos de tentativas. Isolamento do IP `192.168.3.153` e suspensão temporária dos usuários `yugo` e `root`.

**Hora de Contenção:**
22/05/2026 às 11:05 — último evento registrado antes do timeout de autenticação.

---

## 4. Erradicação 

**Causa Raiz Identificada:**
Serviço SSH exposto sem restrição de IP, sem limite de tentativas configurado no UFW, senha fraca ou previsível nas contas-alvo.

**Ações de Eliminação:**
Revisão das contas comprometidas (`yugo` e `root`), troca de senhas e verificação de sessões ativas.

**Vulnerabilidades Identificadas:**
- SSH sem autenticação por chave
- UFW sem regras de limitação de tentativas — não bloqueou o ataque
- Root com acesso SSH habilitado

---

## 5. Recuperação

**Ações de Recuperação:**
Verificação de integridade do sistema, revisão de logs completos, reset de credenciais das contas afetadas e isolamento do IP `192.168.3.153`.

**Status Final:** Totalmente recuperado

---

## 6. Ações Corretivas

| Ação                                  | Descrição                                                              | Prazo     |
|---------------------------------------|------------------------------------------------------------------------|-----------|
| Desabilitar login SSH do root         | Configurar `PermitRootLogin no` em `/etc/ssh/sshd_config`             | Imediato  |
| Autenticação por chave SSH            | Gerar par de chaves e desabilitar autenticação por senha               | 7 dias    |
| Instalar e configurar Fail2Ban        | Banir IPs automaticamente após X tentativas falhas na porta 22         | 7 dias    |
| Restringir SSH por IP no UFW          | Permitir SSH apenas de IPs confiáveis via regras de firewall           | 15 dias   |
| Trocar senhas das contas afetadas     | Redefinir senhas de `yugo` e `root` com credenciais fortes             | Imediato  |
| Alerta automático no Splunk           | Disparar alerta para +5 tentativas falhas em menos de 1 minuto         | 30 dias   |

---

## 7. Lições Aprendidas

- O `sshd` atuou como última linha de defesa via `PerSourcePenalties` — o UFW deveria ter bloqueado antes
- A ausência do Fail2Ban permitiu que o ataque chegasse ao nível do daemon SSH
- O acesso root via SSH representa um risco crítico e deve ser desabilitado em qualquer ambiente
- O monitoramento via Splunk foi eficaz na detecção e correlação dos eventos


