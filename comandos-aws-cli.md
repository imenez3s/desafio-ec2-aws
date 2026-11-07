# Comandos AWS CLI para EC2, AMI e Snapshots

Antes de tudo: `aws configure` (Access Key, Secret Key, região padrão, ex.: `us-east-1`).

## Instâncias

```bash
# Listar instâncias
aws ec2 describe-instances \
  --query "Reservations[].Instances[].[InstanceId,State.Name,InstanceType,PublicIpAddress]" \
  --output table

# Criar instância
aws ec2 run-instances \
  --image-id ami-xxxxxxxx \
  --instance-type t2.micro \
  --key-name minha-chave \
  --security-group-ids sg-xxxxxxxx \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab-ec2-dio}]'

# Parar, iniciar e encerrar
aws ec2 stop-instances --instance-ids i-xxxxxxxx
aws ec2 start-instances --instance-ids i-xxxxxxxx
aws ec2 terminate-instances --instance-ids i-xxxxxxxx
```

## AMIs

```bash
# Criar AMI a partir de uma instância
aws ec2 create-image \
  --instance-id i-xxxxxxxx \
  --name "ami-webserver-v1" \
  --description "Servidor web com Apache"

# Listar minhas AMIs
aws ec2 describe-images --owners self \
  --query "Images[].[ImageId,Name,State]" --output table

# Copiar AMI para outra região
aws ec2 copy-image \
  --source-region us-east-1 \
  --source-image-id ami-xxxxxxxx \
  --region sa-east-1 \
  --name "ami-webserver-v1-copia"

# Remover AMI (depois apagar os snapshots dela)
aws ec2 deregister-image --image-id ami-xxxxxxxx
```

## Snapshots EBS

```bash
# Criar snapshot de um volume
aws ec2 create-snapshot \
  --volume-id vol-xxxxxxxx \
  --description "backup-lab-ec2"

# Listar meus snapshots
aws ec2 describe-snapshots --owner-ids self \
  --query "Snapshots[].[SnapshotId,VolumeId,State,StartTime]" --output table

# Criar volume a partir do snapshot
aws ec2 create-volume \
  --snapshot-id snap-xxxxxxxx \
  --availability-zone us-east-1a \
  --volume-type gp3

# Anexar volume à instância
aws ec2 attach-volume \
  --volume-id vol-yyyyyyyy \
  --instance-id i-xxxxxxxx \
  --device /dev/sdf

# Apagar snapshot
aws ec2 delete-snapshot --snapshot-id snap-xxxxxxxx
```
