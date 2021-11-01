# Дипломный практикум в Cloud: Amazon Web Services"

## Описание решения

### Студент: Иван Жиляев

Решение буду описывать поэтапно, придерживаясь стуктуры задания.

### Создание облачной инфраструктуры

Последнюю стабильную версию Terraform я выкачал в виде бинарного файла и расположил в директории `/bin/`:

```
ivan@kubang:~/study$ terraform --version
Terraform v1.0.8
on linux_amd64
```

Я обнаружил, что в доп.материалах kubespray уже имеется манифест Terraform для организации инфраструктуры в AWS. Он будет основой для моего проекта.  
Содержимое репозитория kubespray я скопировал в каталог kubespray в своём репозитории. Перейдём в каталог с шаблонами манифестов Terraform:

```
cd kubespray/contrib/terraform/aws/
```

В качестве backend я выбрал Terraform Cloud (TFC) в варианте использования [CLI-driven Run Workflow](https://www.terraform.io/docs/cloud/run/cli.html) и типа [Standard](https://www.terraform.io/docs/language/settings/backends/index.html#backend-types) (удаленный backend только хранит состояние). Описание подходов к организации нескольких окружений в официальной документации доступно по [ссылке](https://www.terraform.io/docs/cloud/workspaces/configurations.html#organizing-multiple-environments-for-a-configuration).  
Доступ к удаленному backend для CLI-утилиты будет обеспечен за счёт указания токена пользователя TFC в файле `~/.terraformrc` ([справка](https://www.terraform.io/docs/cli/config/config-file.html#credentials)).

В AWS IAM я создал service account `terraform`. Реквизиты доступа для этого пользователя я добавил в файл `~/.aws/credentials`, а также добавил их в файл с переменными для Terraform.

Из папки с манифестами Terraform произведём инициацию бэкенда и создание workspace:
```
terraform init
terraform workspace new stage
```

Теперь, когда workspace создан, нужно переключить его в режим локального исполнения команд (делаем workspace типа Standard). Для этого в web-консоли TFC зайдём в настройки workspace-а и там активируем `Settings - General - Execution Mode - Local`.

---

SSH-ключ для подключения к создаваемым инстансам я сгенерировал локально, публичную и закрытую части расположил в папке `kubespray/key`.



```
cd contrib/terraform/aws/
terraform plan -var-file=credentials.tfvars
terraform apply -var-file=credentials.tfvars
```


### Создание Kubernetes кластера

Для хранения всех манифестов Ansible был создан [отдельный репозиторий](https://github.com/nimlock/netology-diploma-ansible).

Для создания кластера k8s будет использоваться Kubespray. 

Скопировал kubespray/inventory/sample/group_vars в kubespray/inventory/group_vars для установки своих значений параметров. Значения `minimal_master_memory_mb` и `minimal_node_memory_mb` переопределял на 970 (ansible_memtotal_mb == 973 on t2.micro).

Для снижения нагрузки на облачные инстансы я изменил дефолтный CNI с calico на flannel.

```
pip install -r requirements.txt
ansible-playbook -i ./inventory/hosts ./cluster.yml -e ansible_user=ubuntu -b --become-user=root --flush-cache -e ansible_ssh_private_key_file=key/instance-ssh-key -e kube_network_plugin=flannel
```

Работать с кластером будем с мастер-ноды из окружения root-а, ведь kubespray создал в нём конфигурацию для подключения:

```
ssh -F ./ssh-bastion.conf ubuntu@${master_private_ip} -i key/instance-ssh-key 
sudo -i
```
