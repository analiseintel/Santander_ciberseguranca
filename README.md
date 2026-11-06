# Laboratório de Segurança Ofensiva: Kali Linux, Medusa e Ambientes Vulneráveis


## 1. Visão geral

Auditoria de autenticação em serviços **FTP**, **HTTP (DVWA)** e **SMB**, utilizando **Kali Linux**, **Medusa**, **Metasploitable 2** e **VirtualBox**. O objetivo é compreender como senhas fracas e ausência de controles de autenticação podem permitir tentativas automatizadas e, principalmente, como reduzir esses riscos.

### Objetivos

- Preparar um ambiente virtual isolado para estudos de segurança.
- Identificar serviços expostos e compreender seus mecanismos de autenticação.
- Simular tentativas de força bruta em FTP e em formulário web.
- Demonstrar o conceito de *password spraying* em SMB.
- Interpretar resultados simulados e propor medidas de mitigação.
- Registrar o aprendizado em um repositório técnico no GitHub.

## 2. Ambiente proposto

| Componente | Papel | Configuração ilustrativa |
|---|---|---|
| VirtualBox | Hipervisor | Duas máquinas virtuais |
| Kali Linux | Máquina de auditoria | `192.168.56.10` |
| Metasploitable 2 | Alvo intencionalmente vulnerável | `192.168.56.20` |
| DVWA | Aplicação web de treinamento | Instalada em VM de laboratório, acessível apenas na rede isolada |
| Medusa | Teste de autenticação | Módulos FTP, HTTP e SMB, conforme disponibilidade |

**Isolamento:** usar rede **Host-only** ou rede interna do VirtualBox, sem encaminhamento de portas para redes externas. Host-only e rede interna são modos distintos; ambos podem ser usados para isolamento, desde que configurados corretamente. O DVWA não deve ser exposto à internet.

### Topologia simulada

```text
                  VirtualBox — rede isolada
             192.168.56.0/24                        |
             +----------+----------+
             |                     |
       Kali Linux             Metasploitable 2
       192.168.56.10          192.168.56.20
       Medusa / Nmap          FTP / SMB / DVWA*

```

## 3. Preparação e reconhecimento

Após iniciar as VMs, a verificação da interface de rede e da conectividade poderia ser feita com:

```bash
ip -br addr
ping -c 4 192.168.56.20
```

 Descoberta dos serviços no alvo autorizado:

```bash
nmap -sV -p 21,80,139,445 192.168.56.20
```

**Saída simulada:**

```text
PORT    STATE SERVICE
21/tcp  open  ftp
80/tcp  open  http
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
```



## 4. Cenário 1 — Força bruta controlada em FTP

**Objetivo:** demonstrar como uma conta de treinamento com senha fraca poderia ser identificada por um teste de autenticação limitado.

### Dados fictícios

```text
Usuário: aluno_lab
Senhas de teste:
123456
laboratorio
Treino@2026
```



```bash
printf '123456\nlaboratorio\nTreino@2026\n' > senhas_lab.txt
```

Comando para o serviço FTP **exclusivamente no alvo de laboratório**:

```bash
medusa -h 192.168.56.20 -u aluno_lab -P senhas_lab.txt -M ftp -t 1 -f
```

| Parâmetro | Finalidade |
|---|---|
| `-h` | Define o host de laboratório |
| `-u` | Define o usuário de teste |
| `-P` | Informa o arquivo de senhas de teste |
| `-M ftp` | Seleciona o módulo FTP |
| `-t 1` | Limita a concorrência |
| `-f` | Interrompe após a primeira credencial válida encontrada para o host |

**Resultado simulado:** a senha fraca `laboratorio` seria aceita para a conta fictícia `aluno_lab`, evidenciando o risco de credenciais previsíveis. Isso **não** representa um acesso efetivamente validado.

**Mitigações:** senhas fortes e únicas, restrição de tentativas, monitoramento de falhas, desativação de FTP legado e preferência por protocolos seguros, como SFTP.

## 5. Cenário 2 — Tentativas automatizadas em formulário web (DVWA)

**Objetivo:** analisar a proteção de um formulário de login contra repetidas tentativas de autenticação.

### Procedimento conceitual

1. Disponibilizar o DVWA apenas na rede isolada.
2. Identificar o formulário de treinamento e os parâmetros enviados em uma autenticação normal.
3. Criar um conjunto pequeno de credenciais fictícias para testes autorizados.
4. Verificar, de maneira controlada, como a aplicação reage a tentativas inválidas e válidas.
5. Comparar o comportamento com e sem controles de limitação de tentativas, quando o exercício permitir.

**Requisição HTTP:**

```http
POST /login.php HTTP/1.1
Host: 192.168.56.20
Content-Type: application/x-www-form-urlencoded

username=aluno_lab&password=senha_teste&Login=Login
```


**Resultado:** sem proteção contra tentativas repetidas, o formulário aceitaria uma sequência de credenciais inválidas sem impor atraso. Com limitação de taxa e bloqueio temporário, as tentativas seriam restringidas.

**Mitigações:** MFA quando aplicável, limitação de taxa por conta e origem, bloqueio temporário proporcional ao risco, registro de eventos, proteção de sessão e mensagens de erro que não revelem a existência de usuários.

## 6. Cenário 3 — Password spraying em SMB

**Objetivo:** compreender a diferença entre força bruta tradicional e *password spraying*.

Na força bruta tradicional, várias senhas são testadas para uma conta. No *password spraying*, uma mesma senha comum é testada em diferentes contas, normalmente em baixa frequência, o que pode dificultar a identificação por controles que observam apenas muitas falhas em uma conta.

### Enumeração autorizada

Verificação inicial de serviços SMB:

```bash
nmap -sV -p 139,445 192.168.56.20
```

considera-se uma lista **pré-criada de usuários fictícios**:

```text
aluno01
aluno02
aluno03
```

E uma única senha de treinamento:

```text
Treino@2026
```

### Avaliação

| Usuário fictício | Senha testada | Resultado simulado |
|---|---|---|
| aluno01 | `Treino@2026` | Falha |
| aluno02 | `Treino@2026` | Sucesso |
| aluno03 | `Treino@2026` | Falha |

**Interpretação:** a reutilização de uma senha previsível em múltiplas contas aumenta a exposição do serviço.

**Mitigações:** senhas únicas, MFA nos pontos de acesso compatíveis, restrições de acesso à rede SMB, desativação de versões obsoletas, auditoria centralizada e detecção de falhas distribuídas por várias contas.

## 7. Comparação dos cenários

| Cenário | Vetor | Fragilidade observada na simulação | Controle recomendado |
|---|---|---|---|
| FTP | Autenticação de serviço | Senha fraca | Senhas robustas e restrição de tentativas |
| DVWA | Formulário HTTP | Ausência de limitação de tentativas | Rate limiting e MFA |
| SMB | Múltiplas contas | Senha comum reutilizada | Senhas únicas e monitoramento de spraying |

## 8. Evidências simuladas


- Captura da configuração de rede das VMs, sem exposição externa.
- Registro das portas identificadas no alvo de treinamento.
- Log com horário, serviço, quantidade de tentativas e resultado, sem publicar senhas reais.
- Captura das políticas de bloqueio, limitação de taxa e alertas configurados.
- Comparação entre comportamento anterior e posterior às mitigações.

## 9. Lições aprendidas

Os exercícios mostram que a segurança da autenticação depende de múltiplas camadas. Senhas fortes ajudam, mas não substituem monitoramento, controles de acesso, atualização dos serviços e restrição de tentativas. Também é importante distinguir o propósito das técnicas: varredura identifica serviços, enumeração coleta detalhes e testes de autenticação avaliam a resistência de credenciais e mecanismos de login.

O uso de ambientes deliberadamente vulneráveis é adequado para treinamento, desde que exista isolamento, autorização e documentação clara dos limites do exercício.

## 10. Conclusão

 s três cenários — FTP, DVWA e SMB — permitem compreender riscos relacionados a credenciais fracas, tentativas automatizadas e reutilização de senhas, além de identificar práticas defensivas para reduzir a superfície de ataque.

O projeto atende à proposta de registrar os conhecimentos adquiridos e serve como base para um laboratório prático futuro, com evidências verificáveis e resultados reais claramente diferenciados dos exemplos.

## 11. Referências

- [Kali Linux — Documentação oficial](https://www.kali.org/docs/)
- [Medusa — Ferramenta de auditoria de autenticação](https://www.kali.org/tools/medusa/)
- [Metasploitable 2 — Rapid7](https://docs.rapid7.com/metasploit/metasploitable-2/)
- [DVWA — Repositório oficial](https://github.com/digininja/DVWA)
- [OWASP — Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)

---


