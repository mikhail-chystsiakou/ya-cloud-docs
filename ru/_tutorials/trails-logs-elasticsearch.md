# Загрузка аудитных логов в {{ mes-full-name }}

Создайте трейл, который будет загружать аудитные логи ресурсов отдельного облака в бакет {{ objstorage-full-name }}. Затем настройте непрерывную доставку логов в кластер {{ mes-full-name }}.

Решение, которое описано в руководстве, работает по следующей схеме:

1. [Трейл](../audit-trails/concepts/trail.md) загружает логи в бакет {{ objstorage-name }}.
1. [Бакет](../storage/concepts/bucket.md) монтируется к папке на промежуточной ВМ.
1. На промежуточной ВМ работает скрипт, который по расписанию забирает логи из бакета и отправляет их в кластер {{ mes-name }}.

В кластер {{ mes-name }} загружены данные для анализа логов:

* дашборд с вариантами использования и статистикой;
* набор данных `Saved Queries` для поиска событий безопасности;
* набор правил корреляции `Detection Rules`, на которые предварительно настроены оповещения.

Все [исходные файлы](https://github.com/yandex-cloud/yc-solution-library-for-security/tree/master/auditlogs/export-auditlogs-to-ELK_k8s) для решения находятся в [{{ yandex-cloud }} Security Solution Library](https://github.com/yandex-cloud/yc-solution-library-for-security).

{% include [Yc-solution-library-for-security](../_includes/security-solution-library.md) %}

Чтобы использовать решение для логов не только облака, но и организации или каталога, создайте трейл по инструкции для [организации](../audit-trails/operations/export-organization-bucket.md) или [каталога](../audit-trails/operations/export-folder-bucket.md).

Чтобы экспортировать аудитные логи:

1. [Подготовьте облако к работе](#before-begin).
1. [Подготовьте окружение](#environment-preparing).
1. [Создайте трейл](#create-trail).
1. [Разверните кластер {{ ES }} и промежуточную ВМ](#create-cluster-vm).

Если созданные ресурсы вам больше не нужны, [удалите их](#clear-out).

## Перед началом работы {#before-begin}

{% include [before-you-begin](_tutorials_includes/before-you-begin.md) %}

{% include [cli-install](../_includes/cli-install.md) %}

Часть шагов выполняется в [{{ TF }}](https://www.terraform.io/intro). Если у вас ещё нет {{ TF }}, [установите его и настройте провайдер {{ yandex-cloud }}](../tutorials/infrastructure-management/terraform-quickstart.md#install-terraform).

Часть функций {{ yandex-cloud }} из руководства находится на стадии [Preview](../overview/concepts/launch-stages.md). [Запросите в технической поддержке]({{ link-console-support }}/create-ticket) доступ к этим функциям:

* [{{ at-name }}](../audit-trails/).
* [Группы безопасности](../vpc/concepts/security-groups.md).

### Необходимые платные ресурсы {#paid-resources}

В стоимость поддержки инфраструктуры входит:

* Плата за вычислительные ресурсы и объем хранилища кластера {{ ES }} в редакции `Platinum` (см. [тарифы {{ mes-full-name }}](../managed-elasticsearch/pricing.md)).
* Плата за вычислительные ресурсы и диск промежуточной ВМ (см. [тарифы {{ compute-full-name }}](../compute/pricing.md)).
* Плата за хранение Docker-образа промежуточной ВМ в {{ container-registry-name }} (см. [тарифы {{ container-registry-full-name }}](../container-registry/pricing.md)).
* Плата за хранение данных в бакете и операции с ними (см. [тарифы {{ objstorage-full-name }}](../storage/pricing.md)).

## Подготовьте окружение {#environment-preparing}

### Создайте новый бакет для загрузки в него аудитных логов {#create-backet}

{% list tabs %}

- Консоль управления

   1. В [консоли управления]({{ link-console-main }}) выберите каталог, в котором хотите создать [бакет](../storage/concepts/bucket.md).
   1. Выберите сервис **{{ objstorage-name }}**.
   1. Нажмите **Создать бакет**.
   1. На странице создания бакета:
      1. Введите имя бакета в соответствии с [правилами именования](../storage/concepts/bucket.md#naming), например `trails-bucket`.
      1. Укажите максимальный размер бакета — **Без ограничения**.
      1. Выберите тип [доступа](../storage/concepts/bucket.md#bucket-access) — **Ограниченный**.
      1. Выберите [класс хранилища](../storage/concepts/storage-class.md) — **Стандартное**.
      1. Нажмите кнопку **Создать бакет** для завершения операции.

{% endlist %}

### Создайте сервисный аккаунт {#create-sa}

{% list tabs %}

- Консоль управления

   1. В [консоли управления]({{ link-console-main }}) перейдите в каталог, в котором хотите создать сервисный аккаунт.
   1. Перейдите на вкладку **Сервисные аккаунты**.
   1. Нажмите кнопку **Создать сервисный аккаунт**.
   1. Введите имя сервисного аккаунта, например `trails-bucket`.
   1. Нажмите кнопку **Создать**.

- CLI

   Чтобы создать сервисный аккаунт, выполните команду:

   ```bash
   yc iam service-account create --name <имя_сервисного_аккаунта>
   ```

   Где `name` — имя сервисного аккаунта, например `trails-sa`.

{% endlist %}

### Назначьте роли сервисному аккаунту {#add-roles}

Назначьте сервисному аккаунту роли `audit-trails.viewer` и `storage.uploader`:

{% list tabs %}

- Консоль управления

   1. [Роль](../storage/security/index.md#storage-uploader) `storage.uploader` на каталог с бакетом:

      1. В [консоли управления]({{ link-console-main }}) перейдите в каталог с бакетом.
      1. Перейдите на вкладку **Права доступа**.
      1. Нажмите кнопку **Назначить роли**.
      1. В окне **Настройка прав доступа** нажмите кнопку **Выбрать пользователя**.
      1. Перейдите на вкладку **Сервисные аккаунты**.
      1. Выберите сервисный аккаунт `trails-sa`.
      1. Нажмите кнопку **Добавить роль**.
      1. Выберите роль `storage.uploader`.
      1. Нажмите кнопку **Сохранить**.

   1. [Роль](../audit-trails/security/index.md#roles) `audit-trails.viewer` на облако, логи которого будут отправлены в кластер {{ ES }}:

      1. В [консоли управления]({{ link-console-main }}) перейдите в облако.
      1. Перейдите на вкладку **Права доступа**.
      1. Нажмите кнопку **Назначить роли**.
      1. В окне **Настройка прав доступа** нажмите кнопку **Выбрать пользователя**.
      1. Перейдите на вкладку **Сервисные аккаунты**.
      1. Выберите сервисный аккаунт `trails-sa`.
      1. Нажмите кнопку **Добавить роль**.
      1. Выберите роль `audit-trails.viewer`.
      1. Нажмите кнопку **Сохранить**.

- CLI

   1. Назначьте [роль](../storage/security/index.md#storage-uploader) `storage.uploader` на каталог с бакетом:

      ```bash
      yc resource-manager folder add-access-binding \
      --role storage.uploader \
      --id <идентификатор_каталога> \
      --service-account-id <идентификатор_сервисного_аккаунта_trails-sa>
      ```

      Где:
      * `role` — назначаемая роль.
      * `id` — идентификатор каталога, в котором находится бакет.
      * `service-account-id` — идентификатор сервисного аккаунта.

   1. Назначьте [роль](../audit-trails/security/index.md#roles) `audit-trails.viewer` на облако, логи которого будут отправлены в кластер {{ ES }}:

      ```bash
      yc resource-manager cloud add-access-binding \
      --role audit-trails.viewer \
      --id <идентификатор_облака> \
      --service-account-id <идентификатор_сервисного_аккаунта_trails-sa>
      ```

      Где:
      * `role` — назначаемая роль.
      * `id` — идентификатор облака, логи которого будут отправлены в кластер {{ ES }}.
      * `service-account-id` — идентификатор сервисного аккаунта.

{% endlist %}

### Проверьте ваши роли {#check-roles}

{% list tabs %}

- Консоль управления

   1. В [консоли управления]({{ link-console-main }}) выберите облако, с которого будут собираться аудитные логи.
   1. Перейдите на вкладку **Права доступа**.
   1. Справа сверху включите опцию **Наследуемые роли**, чтобы отобразились роли, унаследованные от организации.
   1. Убедитесь, что у вас есть роли:
      * `iam.serviceAccounts.user` на сервисный аккаунт.
      * `audit-trails.editor` на каталог, где будет находиться трейл.
      * `audit-trails.viewer` на облако, с которого будут собираться аудитные логи.
      * `storage.viewer` на бакет или каталог.

{% endlist %}

### Создайте облачную сеть {#create-network}

Если у вас еще нет [облачной сети](../vpc/concepts/network.md), создайте ее:

{% list tabs %}

- Консоль управления

   1. В [консоли управления]({{ link-console-main }}) перейдите в каталог, где требуется создать облачную сеть.
   1. В списке сервисов выберите **{{ vpc-name }}**.
   1. Нажмите кнопку **Создать сеть**.
   1. Задайте имя сети, например `trails-network`.
   1. Выберите опцию **Создать подсети**.
   1. Нажмите кнопку **Создать сеть**.

- CLI

   Чтобы создать облачную сеть выполните команду:

   ```bash
   yc vpc network create --name <имя_облачной_сети>
   ```

   Где `name` — имя облачной сети, например `trails-network`.

{% endlist %}

### Создайте подсети {#create-subnets}

{% list tabs %}

- Консоль управления

   1. В [консоли управления]({{ link-console-main }}) перейдите в каталог, в котором требуется создать подсеть.
   1. В списке сервисов выберите **{{ vpc-name }}**.
   1. Выберите облачную сеть, например `trails-network`.
   1. Нажмите кнопку **Добавить подсеть**.
   1. Укажите имя подсети, например `trails-subnet-1`.
   1. Выберите зону доступности, например `{{ region-id }}-a`.
   1. Введите CIDR подсети: IP-адрес и маску подсети, например `10.128.0.0/24`.
   1. Нажмите кнопку **Создать подсеть**.
   1. Также создайте еще две подсети:
      * `trails-subnet-2` в зоне доступности `{{ region-id }}-b`, CIDR подсети – `10.129.0.0/24`.
      * `trails-subnet-3` в зоне доступности `{{ region-id }}-c`, CIDR подсети – `10.130.0.0/24`.

- CLI

   1. Узнайте имя облачной сети, в которой требуется создать подсеть:

       ```bash
       yc vpc network list --folder-id <идентификатор_каталога>
       ```

      Где `folder-id` — идентификатор каталога, в котором размещена облачная сеть.

      Результат:

      ```text
      +----------------------+----------------+
      |          ID          |      NAME      |
      +----------------------+----------------+
      | enpavfmgapumnl7cqin8 | trails-network |
      +----------------------+----------------+
      ```

   1. Создайте подсеть:

      ```bash
      yc vpc subnet create \
      --name <имя_подсети> \
      --folder-id <идентификатор_каталога> \
      --network-name <имя_облачной_сети> \
      --zone <зона_доступности> \
      --range <CIDR_подсети>
      ```

      Где:
      * `name` — имя подсети, например `trails-subnet-1`.
      * `folder-id` — идентификатор каталога, в котором размещена облачная сеть.
      * `network-name` — имя облачной сети, например `trails-network`.
      * `zone` — зона доступности, например `{{ region-id }}-a`.
      * `range` — CIDR подсети, например `10.128.0.0/24`.
   1. Также создайте еще две подсети:
      * `trails-subnet-2` в зоне доступности `{{ region-id }}-b`, CIDR подсети – `10.129.0.0/24`.
      * `trails-subnet-3` в зоне доступности `{{ region-id }}-c`, CIDR подсети – `10.130.0.0/24`.

{% endlist %}

### Настройте подсеть {#configure-subnet}

1. [Настройте](../vpc/operations/create-nat-gateway.md) NAT-шлюз для подсети, в которой будет развернута промежуточная ВМ.
1. Настройте разрешения сетевого трафика в [группе безопасности по умолчанию](../vpc/concepts/security-groups.md#default-security-group). Если группа безопасности недоступна, для кластера {{ ES }} будет разрешен любой входящий и исходящий трафик.

   Если группа безопасности доступна, [добавьте](../vpc/operations/security-group-update.md#add-rule) в нее следующие правила:

   Направление<br>трафика | Описание | Диапазон<br>портов | Протокол | Тип<br>источника | Источник/Назначение
   --- | --- | --- | --- | --- | ---
   Входящий | incoming-https | 443 | TCP | CIDR | 0.0.0.0/0
   Входящий | incoming-9002 | 9200 | TCP | CIDR | 0.0.0.0/0

{% note info %}

Вы также можете развернуть необходимое окружение с помощью {{ TF }}. См. [пример файла конфигурации](https://github.com/yandex-cloud/yc-solution-library-for-security/blob/master/auditlogs/export-auditlogs-to-ELK_main/terraform/example/main.tf).

{% endnote %}

## Создайте трейл {#create-trail}

{% list tabs %}

- Консоль управления

   1. В [консоли управления]({{ link-console-main }}) выберите каталог, в котором вы хотите разместить трейл.
   1. Выберите сервис **{{ at-name }}**.
   1. Нажмите кнопку **Создать трейл**.
   1. В поле **Имя** укажите имя создаваемого трейла.
   1. В блоке **Фильтр** задайте параметры области сбора аудитных логов:
      * **Ресурс** — выберите `Облако`.
      * **Облако** — не требует заполнения, содержит имя облака, в котором будет находиться трейл.
      * **Каталоги** — оставьте значение по умолчанию — `все каталоги`.
   1. В блоке **Назначение** задайте параметры объекта назначения:
      * **Назначение** —  `{{ objstorage-name }}`.
      * **Бакет** — выберите бакет, в который будут загружаться аудитные логи, например `trails-bucket`.
      * **Префикс объекта** — необязательный параметр, участвует в [полном имени](../audit-trails/concepts/format.md#log-file-name) файла аудитного лога.
   1. В блоке **Сервисный аккаунт** выберите сервисный аккаунт, от имени которого трейл будет загружать файлы аудитного лога в бакет, например `trails-sa`.
   1. Нажмите кнопку **Создать**.

{% endlist %}

{% note warning %}

Представленное решение будет удалять аудитные логи из бакета после экспорта в кластер {{ ES }}.

Чтобы одновременно хранить логи в бакете и кластере {{ ES }}, создайте два трейла для двух бакетов: один для хранения, другой для экспорта в кластер {{ ES }}.

{% endnote %}

## Разверните кластер {{ ES }} и промежуточную ВМ {#create-cluster-vm}

{% list tabs %}

- {{ TF }}

   1. Скачайте директорию с [модулями {{ TF }}](https://github.com/yandex-cloud/yc-solution-library-for-security/tree/master/auditlogs/export-auditlogs-to-ELK_main/terraform/modules):
      * `yc-managed-elk` создает кластер {{ ES }} и назначает пароль пользователю `admin`. Параметры кластера:
         * количество хостов — три, по одному в каждой зоне доступности;
         * [класс хостов](../managed-elasticsearch/concepts/instance-types.md#available-flavors) — s2.medium;
         * редакция — `Platinum`;
         * хранилище — network-hdd, 1 ТБ;
         * количество реплик индексов — 2;
         * политика создания новых индексов — [rollover](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-rollover.html) (новые индексы создаются каждые 30 дней или по достижению объема в 50 ГБ). Подробнее см. [Рекомендации для высокой доступности данных](https://github.com/yandex-cloud/yc-solution-library-for-security/blob/master/auditlogs/export-auditlogs-to-ELK_main/CONFIGURE-HA_RU.md).

      * `yc-elastic-trail`:
         * создает статический ключ доступа для работы сервисного аккаунта с объектами JSON в бакете, шифрования и расшифрования секретов;
         * создает промежуточную ВМ на базе образа `{{ registry }}/sol/s3-elk-importer:latest` со скриптом для передачи аудитных логов из бакета в кластер {{ ES }};
         * создает пару [ssh-ключей](../glossary/ssh-keygen.md) и сохраняет закрытую часть на диск, публичную в ВМ;
         * создает {{ kms-short-name }}-ключ;
         * назначает роль `kms.keys.encrypterDecrypter` сервисному аккаунту на ключ для шифрования секретов.
         * шифрует секреты и передает их в Docker-контейнер.

   1. Создайте файл `main.tf` и скопируйте в него следующую конфигурацию {{ TF }}:

      ```hcl
      terraform {
      required_providers {
         yandex = {
         source = "yandex-cloud/yandex"
         }
      }
      required_version = ">= 0.13"
      }

      provider "yandex" {
      token     = "<OAuth>"
      cloud_id  = "<идентификатор_облака>"
      folder_id = "<идентификатор_каталога>"
      }

      module "yc-managed-elk" {
      source     = "<путь_к_директории_yc-managed-elk>"
      folder_id  = "<идентификатор_каталога>"
      subnet_ids = ["<идентификатор_подсети_1>", "<идентификатор_подсети_2>", "<идентификатор_подсети_3>"]
      network_id = "<идентификатор_облачной_сети>"
      elk_edition = "platinum"
      elk_datanode_preset = "s2.medium"
      elk_datanode_disk_size = 1000
      elk_public_ip = false
      }

      module "yc-elastic-trail" {
      source          = "<путь_к_директории_yc-elastic-trail>"
      folder_id       = "<идентификатор_каталога>"
      elk_credentials = module.yc-managed-elk.elk-pass
      elk_address     = module.yc-managed-elk.elk_fqdn
      bucket_name     = "<имя_бакета>"
      bucket_folder   = "<префикс_логов_в_бакете>"
      sa_id           = "<идентификатор_сервисного_аккаунта>"
      coi_subnet_id   = "<идентификатор_подсети_1>"
      }

      output "elk-pass" {
      value     = module.yc-managed-elk.elk-pass
      sensitive = true
      }
      output "elk_fqdn" {
      value = module.yc-managed-elk.elk_fqdn
      }

      output "elk-user" {
      value = "admin"
      }
       ```

      Где:
      * `token` — [OAuth-токен](../iam/concepts/authorization/oauth-token.md).
      * `cloud_id` — идентификатор облака.
      * `folder_id` — идентификатор каталога.
      * `source` в блоке модуля `yc-managed-elk` — путь к директории `yc-managed-elk`.
      * `source` в блоке модуля `yc-elastic-trail` — путь к директории `yc-elastic-trail`.
      * `subnet_ids` — идентификаторы подсетей, в которых будет размещен кластер {{ ES }}, например подсетей`trails-subnet-1`, `trails-subnet-2` и `trails-subnet-3`.
      * `network_id` — идентификатор облачной сети, в которой будет размещен кластер {{ ES }}, например облачной сети`trails-network`.
      * `elk_public_ip` — укажите значение `true`, если нужен публичный доступ к кластеру {{ ES }}, или `false`.
      * `bucket_name` — имя бакета, в который передаются аудитные логи, например `trails-bucket`.
      * `bucket_folder` — префикс аудитных логов в бакете. Если логи пишутся в корневую директорию бакета — оставьте незаполненным.
      * `sa_id` — идентификатор сервисного аккаунта, от имени которого передаются аудитные логи, например сервисного аккаунта `trails-sa`.
      * `coi_subnet_id` — идентификатор подсети, в которой будет размещена промежуточная ВМ, например подсети `trails-subnet-1`.

   1. Инициализируйте {{ TF }}. В терминале перейдите в директорию, где вы создали конфигурационный файл, и выполните команду:

      ```bash
      terraform init
      ```

   1. Проверьте корректность конфигурационных файлов:

      ```bash
      terraform plan
      ```

      Если конфигурация описана верно, в терминале отобразится список создаваемых ресурсов и их параметров. Если в конфигурации есть ошибки, {{ TF }} на них укажет.

   1. Разверните облачные ресурсы:

      ```bash
      terraform apply
      ```

   1. Подтвердите создание ресурсов: введите в терминале `yes` и нажмите **Enter**.

      В выводе команды будут указаны параметры для доступа к кластеру {{ ES }}:
      * `elk_fqdn` — сетевой адрес кластера {{ ES }}.
      * `elk-user` — имя пользователя кластера {{ ES }}.

      Проверить созданные ресурсы можно в [консоли управления]({{ link-console-main }}).

   1. Получите пароль для доступа к кластеру {{ ES }}:

      ```bash
      terraform output elk-pass
      ```

{% endlist %}

Чтобы подключиться к кластеру {{ ES }}, введите в браузере адрес кластера, на открывшейся странице укажите имя пользователя и пароль.

### Поддерживайте инфраструктуру в актуальном состоянии {#actualization}

Для того, чтобы использовать самую актуальную версию решения по экспорту аудитных логов в кластер {{ ES }}:

* Поддерживайте актуальную версию Docker-контейнера промежуточной ВМ, убедитесь, что вы используете последнюю доступную версию образа `{{ registry }}/sol/s3-elk-importer:latest`. Docker-контейнер можно обновить одним из следующих способов:
   * Удалите и заново создайте промежуточную ВМ с помощью {{ TF }}.
   * Остановите и удалите Docker-контейнер `audit-trail-worker-*` на промежуточной ВМ, удалите образ `s3-elk-importer`, перезапустите промежуточную ВМ — после запуска будет скачан новый образ и создан новый контейнер.
* Поддерживайте в актуальном состоянии правила корреляции и запросы событий безопасности. Чтобы обновить данные в Kibana, запустите контейнер `elk-updater`:

   ```bash
   docker run \
        -it \
        --rm \
        -e ELASTIC_AUTH_USER='<имя_пользователя_кластера_{{ ES }}>' \
        -e ELASTIC_AUTH_PW='<пароль_для_доступа_к_кластеру_{{ ES }}>' \
        -e KIBANA_SERVER='<сетевой_адрес_кластера_{{ ES }}>' \
        --name elk-updater {{ registry }}/sol/elk-updater:latest
   ```

   Где:
   * `ELASTIC_AUTH_USER` — имя пользователя кластера {{ ES }}.
   * `ELASTIC_AUTH_PW` — пароль для доступа к кластеру {{ ES }}.
   * `KIBANA_SERVER` — сетевой адрес кластера {{ ES }}.

{% note tip %}

Также за обновлениями решения вы можете следить в репозитории [yandex-cloud/yc-solution-library-for-security](https://github.com/yandex-cloud/yc-solution-library-for-security/tree/master/auditlogs/export-auditlogs-to-ELK_main).

{% endnote %}

## Как удалить созданные ресурсы {#clear-out}

Удалите ресурсы, которые вы больше не будете использовать, во избежание списания средств за них:

* Удалите ресурсы, созданные с помощью {{ TF }}:

   ```bash
   terraform destroy
   ```

   Подтвердите удаление ресурсов: введите в терминале `yes` и нажмите **Enter**.

* [Удалите](../storage/operations/buckets/delete.md) бакет.
