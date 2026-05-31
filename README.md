# SSH Brute Force Detection — Splunk SOC Lab

> Simulação e detecção de ataque de força bruta via SSH em ambiente Linux monitorado pelo Splunk.

---

## Contexto

Simulação de ataque de brute force SSH contra um endpoint Linux (Debian), monitorado por um dashboard SOC no Splunk. O ataque foi executado a partir de uma VM atacante na mesma rede local (kali linux), e todo o processo de detecção, contenção e resposta se baseou no framework **NIST SP 800-61**.

---

## Objetivo

Detectar autenticações falhas repetidas via SSH, identificar o IP de origem, os usuários-alvo e verificar se houve algum comprometimento de contas durante o ataque.

---

## Ambiente

| Componente        | Detalhe                        |
|-------------------|--------------------------------|
| OS Alvo           | Debian Linux (host: `cancer`)  |
| IP Alvo           | `192.168.3.121` — porta 22     |
| VM Atacante       | IP `192.168.3.153`(Kali linux) |
| SIEM              | Splunk                         |
| Rede              | Rede local isolada (lab)       |
| Serviço atacado   | SSH (sshd)                     |

---

## Ataque Simulado

Múltiplas tentativas de autenticação SSH realizadas ultilizabdo o hydra a partir do IP `192.168.3.153` contra o host `cancer` (`192.168.3.121:22`).

- **Total de tentativas falhas:** 23
- **Logins aceitos:** 3
- **Usuários alvo:** `yugo`, `root`
- **Período:** 22/05/2026 das 10:54h às 11:05h
- **Bloqueio:** O próprio daemon `sshd` via mecanismo `PerSourcePenalties` encerrou as conexões

---

## Investigação

### Event IDs / Logs observados

Logs coletados de `/var/log/auth.log` via Splunk (sourcetype: `syslog`):

```
2026-05-22T11:04:06 cancer sshd[98795]: drop connection #1 from [192.168.3.153]:55736 on [192.168.3.121]:22 penalty: failed authentication
2026-05-22T11:04:06 cancer sshd[98795]: PerSourcePenalties logging rate-limited: additional 4 connections dropped
2026-05-22T11:05:41 cancer sshd[98795]: Timeout before authentication for connection from 192.168.3.153 to 192.168.3.121
```

### Query utilizada

```
spl

index=* penalty: "failed authentication"

```

> Retornou **16 eventos** no período de 21/05/26 11:00 a 22/05/26 11:50.

### Timeline do incidente

| Hora        | Evento                                              |
|-------------|-----------------------------------------------------|
| 10:54:14    | Primeiras tentativas de autenticação falha          |
| 11:04:06    | Pico de conexões — sshd inicia bloqueio automático  |
| 11:04:06    | PerSourcePenalties derruba conexões simultâneas     |
| 11:05:41    | Timeout de autenticação — ataque encerrado          |

### IOCs (Indicadores de Comprometimento)

| Tipo             | Valor              |
|------------------|--------------------|
| IP Atacante      | `192.168.3.153`    |
| IP Alvo          | `192.168.3.121`    |
| Porta            | `22 (SSH)`         |
| Usuários alvo    | `yugo`, `root`     |
| Processo         | `sshd[98795]`      |
| Log source       | `/var/log/auth.log`|

### Dashboard SOC — Splunk

Evidências coletadas no dashboard **"Security Operations Center (SOC) - Linux Monitoring"**:

- **Failed Passwords:** 23
- **Accepted Passwords:** 3
- **Usuários Alvos:** `yugo`, `root`
- **Pico de tentativas:** 17 eventos às 10:40h (visualizado no gráfico Login Attempts)

> Screenshots disponíveis na pasta `/screenshots`

---

## Resultado

O comportamento foi identificado como um ataque de **brute force SSH**, correlacionando 23 falhas de autenticação consecutivas originadas de um único IP em menos de 2 minutos. O bloqueio foi realizado automaticamente pelo `sshd` via `PerSourcePenalties`, sem intervenção do UFW — o que evidenciou um descuido na configuração do firewall.

Três logins foram aceitos durante o ataque, indicando **possível comprometimento das contas** `yugo` e `root`.

---

## Mitigação

| Ação                                      | Descrição                                                                 | Prazo     |
|-------------------------------------------|---------------------------------------------------------------------------|-----------|
| Desabilitar login SSH do root             | Configurar `PermitRootLogin no` em `/etc/ssh/sshd_config`                 | Imediato  |
| Autenticação por chave SSH                | Gerar par de chaves e desabilitar autenticação por senha                  | 7 dias    |
| Instalar e configurar Fail2Ban            | Banir IPs automaticamente após X tentativas falhas na porta 22            | 7 dias    |
| Restringir SSH por IP no UFW              | Permitir SSH apenas de IPs confiáveis via regras de firewall              | 15 dias   |
| Trocar senhas das contas afetadas         | Redefinir senhas de `yugo` e `root` com credenciais fortes                | Imediato  |
| Alerta automático no Splunk               | Disparar alerta para mais de 5 tentativas falhas em menos de 1 minuto     | 30 dias   |

---

## Relatório de Incidente

O relatório completo foi documentado seguindo o ciclo de resposta a incidentes do **NIST SP 800-61**:

>> [Visualizar report.md](./20-05-2026-Report_Incident.md)

---

## Referências

- [NIST SP 800-61 Rev. 2 — Computer Security Incident Handling Guide](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-61r2.pdf)
- [Splunk Enterprise Documentation](https://docs.splunk.com)
- [OpenSSH PerSourcePenalties](https://man.openbsd.org/sshd_config#PerSourcePenalties)
