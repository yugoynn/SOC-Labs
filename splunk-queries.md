#  Splunk Queries — SSH Brute Force Detection

Queries utilizadas durante a investigação do incidente de brute force SSH em 22/05/2026.

---

## 1. Detectar tentativas de autenticação falha

```
spl

index=* penalty: "failed authentication"

```
---

## 2. Filtrar por IP atacante

```
spl

index=* src_ip=192.168.3.153 "failed authentication"

```
---

## 3. Contar tentativas por IP de origem

```
spl

index=* "failed password" sourcetype=syslog
| stats count by src_ip
| sort -count

```
---

## 4. Identificar usuários-alvo do ataque

```
spl

index=* "failed password" sourcetype=syslog
| stats count by user
| sort -count
```
---

## 5. Timeline do ataque

```
spl

index=* src_ip=192.168.3.153 sourcetype=syslog
| timechart span=1m count
```
---

## 6. Detectar logins aceitos durante o ataque (possível comprometimento)

```
spl

index=* "Accepted password" sourcetype=syslog
| table _time, src_ip, user, host
```
---

## 7. Alerta recomendado — mais de 5 falhas em 1 minuto

```spl
index=* "failed password" sourcetype=syslog
| bucket _time span=1m
| stats count by _time, src_ip
| where count > 5
```
> Usei está query para criar um **alerta automático** no Splunk em `Alerts > New Alert`.
