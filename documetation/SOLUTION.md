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

В качестве backend я выбрал Terraform Cloud (TFC) в варианте использования [CLI-driven Run Workflow](https://www.terraform.io/docs/cloud/run/cli.html) и типа [Standard](https://www.terraform.io/docs/language/settings/backends/index.html#backend-types) (удаленный backend только хранит состояние). Такой выбор Workflow продиктован удобством работы в среде с несколькими workspace: мы можем привычно менять workspace в локальной консоли и работать с единой версией конфигурации для разных окружений, в отличие от VCS-ориентированного подхода, когда требуется вести под каждое окружение собственную ветку. Описание подходов к организации нескольких окружений в официальной документации доступно по [ссылке](https://www.terraform.io/docs/cloud/workspaces/configurations.html#organizing-multiple-environments-for-a-configuration).  
Доступ к удаленному backend для CLI-утилиты будет обеспечен за счёт указания токена пользователя TFC в файле `~/.terraformrc` ([справка](https://www.terraform.io/docs/cli/config/config-file.html#credentials)).

В AWS IAM я создал service account `terraform`. Реквизиты доступа для этого пользователя я добавил в файл `~/.aws/credentials`. Из этого стандартного расположения Terraform будет брать реквизиты для работы провайдера AWS.

Из папки с манифестами Terraform произведём инициацию бэкенда и создание workspace-ов:
```
terraform init
terraform workspace new stage
terraform workspace new prod
terraform workspace select stage
```

Теперь, когда workspace-ы созданы, нужно переключить их в режим локального исполнения команд (делаем workspace типа Standard). Для этого в web-консоли TFC зайдём в настройки каждого workspace-а и там активируем `Settings - General - Execution Mode - Local`.

---

Для хранения всех манифестов Terraform был создан [отдельный репозиторий](https://github.com/nimlock/netology-diploma-terraform).

Как и было предложено в задании, VPC я создаю готовым модулем.

Правила для `security groups` задал методом *inline* - внутри описания единственного ресурса.

Для доступа к инстансам будет использоваться единый ключ SSH, он расположен в каталоге с манифестами Terraform.

В выводе Terraform предоставляет внешние и внутренние ip-адреса созданных инстансов. Проверить доступ до них можно командой:

```
ssh ubuntu@${INSTANCE_PUBLIC_IP} -i instance-ssh-key
```

---

TODO: Можно добавить схему полученного VPC с приватной и публичной частью, показать как каждая из них выходит в инет, какие объекты вообще были созданы модулем и для чего. [Пример схемы](https://gruntwork.io/assets/img/guides/vpc/vpc-diagram.png).

---

### Создание Kubernetes кластера

Для хранения всех манифестов Ansible был создан [отдельный репозиторий](https://github.com/nimlock/netology-diploma-ansible).

Для создания кластера k8s я решил воспользоваться Kubespray. Я добавлю этот инструмент как `git submodule` в свой репозиторий:

```
git submodule add https://github.com/kubernetes-incubator/kubespray
cd kubespray
git checkout v2.17.0
pip install -r requirements.txt
```

_Примечание: чтобы склонировать репозиторий с субмодулями можно воспользоваться командой `git clone --recurse-submodules https://github.com/nimlock/netology-diploma-ansible`._


Заполним [инвентарь kubespray](https://github.com/nimlock/netology-diploma-ansible/inventory.ini) для дальнейшей установки кластера в конфигурации "1 мастер, 2 воркера, 1 ингресс".

Для обеспечения связи с нодами поссредством Ansible сделаем следующее:
- возьмём подготовленный для kubespray [inventory-файл](inventory.ini), где укажем ip-адреса ВМ и добавим переменную `ansible_user=vagrant`
- скопируем ключ ssh на управляемые хосты с помощью подготовленной таски [copy_ssh_id.yml](ansible/roles/init_role/tasks/copy_ssh_id.yml).

Первый пункт выполнен вручную, а второй - с помощью команды:

```
ansible-playbook -i inventory.ini -t init -k -c paramiko ansible/playbook.yml
```

Проверим что Ansible может управлять целевым хостом:

```
ansible-playbook -i inventory.ini -t check-connection ansible/playbook.yml
```

Теперь можно установить кластер через kubespray:

```
virtualenv venv
source venv/bin/activate
git clone https://github.com/kubernetes-incubator/kubespray
echo kubespray/ > .gitignore
cd kubespray
git checkout v2.16.0
pip install -r requirements.txt
ansible-playbook -i ../inventory.ini -e kube_network_plugin=calico -b cluster.yml
deactivate
```

Для упрощения задачи работать с кластером будем с мастер-ноды из окружения root-а, ведь kubespray создал в нём конфигурацию для подключения:

```
ssh vagrant@192.168.88.26
sudo -i
```
