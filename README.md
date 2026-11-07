# Gerenciando Instâncias EC2 na AWS

Repositório de anotações do laboratório **Gerenciando Instâncias EC2 na AWS** da DIO (Formação AWS Cloud Foundations).

Aqui documento o que pratiquei sobre **instâncias EC2**, **AMIs (Amazon Machine Images)** e **Snapshots EBS**, com os passos que segui, comandos úteis e os principais aprendizados.

---

## Sumário

1. [Conceitos principais](#1-conceitos-principais)
2. [Criando uma instância EC2](#2-criando-uma-instância-ec2)
3. [Criação e uso de AMIs](#3-criação-e-uso-de-amis)
4. [Snapshots EBS](#4-snapshots-ebs)
5. [AMI x Snapshot: quando usar cada um](#5-ami-x-snapshot-quando-usar-cada-um)
6. [Boas práticas e custos](#6-boas-práticas-e-custos)
7. [Comandos AWS CLI](#7-comandos-aws-cli)
8. [Aprendizados](#8-aprendizados)
9. [Referências](#9-referências)

---

## 1. Conceitos principais

| Conceito | O que é |
|---|---|
| **EC2** | Serviço de servidores virtuais (instâncias) sob demanda na AWS. |
| **Tipo de instância** | Define CPU, memória e rede (ex.: `t2.micro`, `t3.micro`, elegíveis ao Free Tier). |
| **EBS** | Disco (volume) persistente anexado à instância. Funciona como um HD/SSD na nuvem. |
| **AMI** | "Molde" de uma máquina: sistema operacional, configurações e softwares. Serve para criar novas instâncias iguais. |
| **Snapshot** | Cópia de segurança pontual de um volume EBS, salva no S3 e incremental. |
| **Security Group** | Firewall virtual que controla as portas de entrada e saída da instância. |
| **Key Pair** | Par de chaves usado para acessar a instância via SSH. |

## 2. Criando uma instância EC2

Passo a passo no console:

1. Acessar **EC2 > Instances > Launch instances**.
2. Definir um **nome** (ex.: `lab-ec2-dio`).
3. Escolher a **AMI** (ex.: Amazon Linux 2023 ou Ubuntu).
4. Escolher o **tipo de instância** (`t2.micro` / `t3.micro` para ficar no Free Tier).
5. Criar ou selecionar um **Key Pair** (guardar o arquivo `.pem` com segurança).
6. Configurar o **Security Group** liberando apenas o necessário (ex.: SSH porta 22 só para o meu IP).
7. Definir o **armazenamento** (volume EBS gp3, 8 GB é suficiente para o lab).
8. Clicar em **Launch instance**.

Acesso via SSH:

```bash
chmod 400 minha-chave.pem
ssh -i minha-chave.pem ec2-user@<IP_PUBLICO>   # Amazon Linux
ssh -i minha-chave.pem ubuntu@<IP_PUBLICO>     # Ubuntu
```

Estados de uma instância: `pending` > `running` > `stopping` > `stopped` > `terminated`.

- **Stop**: desliga a máquina, mantém o volume EBS (cobra o armazenamento, não cobra computação).
- **Terminate**: apaga a instância. Por padrão o volume raiz também é apagado.

## 3. Criação e uso de AMIs

A AMI permite "congelar" uma máquina já configurada e replicá-la quantas vezes precisar.

### Criando uma AMI a partir de uma instância

1. Configurar a instância (ex.: instalar um servidor web).
   ```bash
   sudo yum update -y
   sudo yum install -y httpd
   sudo systemctl enable --now httpd
   echo "<h1>Servidor criado a partir da minha AMI</h1>" | sudo tee /var/www/html/index.html
   ```
2. No console: selecionar a instância > **Actions > Image and templates > Create image**.
3. Dar nome e descrição (ex.: `ami-webserver-v1`).
4. Aguardar o status da AMI mudar de `pending` para `available` em **EC2 > AMIs**.

> Observação: ao criar a AMI a instância é reiniciada por padrão, para garantir consistência do disco. Dá para marcar **No reboot**, mas com risco de inconsistência.

### Usando a AMI

1. Em **EC2 > AMIs**, selecionar a AMI > **Launch instance from AMI**.
2. A nova instância já sobe com tudo que estava configurado (no exemplo, o Apache já rodando).

### Outras operações úteis

- **Copiar AMI para outra região**: útil para recuperação de desastres ou para subir o mesmo ambiente em outro lugar.
- **Compartilhar AMI** com outras contas AWS.
- **Deregister**: remove a AMI. Atenção: os snapshots ligados a ela continuam existindo e cobrando, precisam ser apagados separadamente.

## 4. Snapshots EBS

Snapshots são backups dos volumes EBS.

### Características

- São **incrementais**: o primeiro é completo, os próximos salvam só os blocos alterados.
- Ficam armazenados no **S3** (gerenciado pela AWS, não aparece no seu bucket).
- São **regionais**, mas podem ser copiados para outras regiões.
- Um volume pode ser restaurado em **outra Zona de Disponibilidade** a partir do snapshot.

### Criando um snapshot

1. **EC2 > Elastic Block Store > Volumes**.
2. Selecionar o volume > **Actions > Create snapshot**.
3. Adicionar descrição e tags (ex.: `Name = backup-lab-ec2`).

### Restaurando um volume

1. **EC2 > Snapshots** > selecionar o snapshot > **Create volume from snapshot**.
2. Escolher a **Availability Zone** (precisa ser a mesma da instância onde vou anexar).
3. **Volumes** > selecionar o novo volume > **Actions > Attach volume**.
4. Dentro da instância, montar o disco:
   ```bash
   lsblk                         # ver o novo disco (ex.: /dev/xvdf)
   sudo mkdir /dados
   sudo mount /dev/xvdf /dados   # se já tiver sistema de arquivos
   ```

### Automação

- **Amazon Data Lifecycle Manager (DLM)** ou **AWS Backup** permitem agendar snapshots automáticos e definir retenção (ex.: diário, manter 7).

## 5. AMI x Snapshot: quando usar cada um

| | AMI | Snapshot EBS |
|---|---|---|
| **O que guarda** | A máquina inteira (volumes + configurações de boot) | Um volume específico |
| **Uso principal** | Criar novas instâncias iguais | Backup e restauração de dados |
| **Pode subir instância direto?** | Sim | Não (precisa virar volume ou AMI) |
| **Relação** | Uma AMI é formada por um ou mais snapshots | Pode ser usado para criar uma AMI |

Resumo: **AMI para replicar servidor, Snapshot para proteger dados.**

## 6. Boas práticas e custos

- Usar tipos elegíveis ao **Free Tier** durante os estudos.
- **Parar ou encerrar** instâncias ao terminar o lab.
- Apagar **AMIs, snapshots e volumes** que não serão mais usados (são cobrados por GB/mês).
- Usar **tags** para identificar recursos (projeto, ambiente, dono).
- Nunca deixar a porta 22 aberta para `0.0.0.0/0`; restringir ao seu IP.
- Guardar o arquivo `.pem` com segurança e nunca subir para o GitHub.
- Configurar um **alerta de orçamento** no AWS Budgets.

## 7. Comandos AWS CLI

Os mesmos passos podem ser feitos pelo terminal. Ver o arquivo [comandos-aws-cli.md](comandos-aws-cli.md).

## 8. Aprendizados

- Entendi a diferença prática entre **parar** e **encerrar** uma instância e o impacto disso no custo.
- A **AMI** economiza tempo: configuro o servidor uma vez e replico quantas vezes precisar, sem repetir instalação.
- O **snapshot** é a forma mais simples de backup de disco e também permite mover dados entre Zonas de Disponibilidade.
- Apagar uma AMI não apaga os snapshots dela. Isso é um ponto de atenção para não gerar custo escondido.
- Automatizar snapshots com DLM ou AWS Backup é essencial em ambiente de produção.

## 9. Referências

- [Documentação Amazon EC2](https://docs.aws.amazon.com/pt_br/AWSEC2/latest/UserGuide/concepts.html)
- [Amazon Machine Images (AMI)](https://docs.aws.amazon.com/pt_br/AWSEC2/latest/UserGuide/AMIs.html)
- [Snapshots do Amazon EBS](https://docs.aws.amazon.com/pt_br/ebs/latest/userguide/ebs-snapshots.html)
- [Amazon Data Lifecycle Manager](https://docs.aws.amazon.com/pt_br/ebs/latest/userguide/snapshot-lifecycle.html)
- [Guia de Markdown do GitHub](https://docs.github.com/pt/get-started/writing-on-github)

---

Projeto desenvolvido para o bootcamp da [DIO](https://www.dio.me/).
