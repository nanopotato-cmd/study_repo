# Terraform: модули и окружения (практический кейс)

## Что делали

Разработали Terraform-модуль для развёртывания двух ВМ (web и api) с дополнительными дисками в Yandex Cloud. Создали два окружения: `dev` и `prod`.

## Структура проекта

```test
├── environments/
│   ├── dev/                  # окружение разработки
│   │   ├── main.tf           # вызов корневого модуля
│   │   ├── terraform.tfvars  # значения переменных (dev)
│   │   └── variables.tf
│   └── prod/                 # продакшен окружение
│       ├── main.tf
│       ├── terraform.tfvars  # значения переменных (prod)
│       └── variables.tf
├── modules/
│   ├── compute/              # модуль ВМ
│   │   ├── main.tf           # ресурс ВМ
│   │   ├── variables.tf      # входные параметры
│   │   └── outputs.tf
│   └── disk/                 # модуль дисков
│       ├── main.tf           # ресурс дисков + dynamic блок
│       ├── variables.tf
│       └── outputs.tf
├── main.tf                   # корневой модуль (собирает дочерние)
├── variables.tf
├── outputs.tf
└── README.md
```

# Основные шаги

## 1. Модуль дисков (`modules/disk/`)

Создаёт дополнительные диски с помощью `for_each`:

```hcl
resource "yandex_compute_disk" "extra_disks" {
  for_each = toset(var.disk_names)
  name     = each.key
  type     = var.disk_type
  size     = var.disk_size
}
```
**Для чего:** Позволяет создать любое количество дисков из списка имён.

## 2. Модуль ВМ (`modules/compute/`)

Создаёт одну ВМ. Подключает дополнительные диски через `dynamic` блок:

```hcl
dynamic "secondary_disk" {
  for_each = var.extra_disks
  content {
    disk_id = secondary_disk.value
  }
}
```
**Для чего:** dynamic генерирует блоки secondary_disk для каждого переданного диска.

## 3. Корневой модуль (`main.tf`)

Собирает дочерние модули вместе:

```hcl
data "yandex_vpc_subnet" "default" {
  name = "default-ru-central1-a"
}

module "api_disks" {
  source = "./modules/disk"
  disk_names = var.extra_disk_names
  # ...
}

module "web_vm" {
  source = "./modules/compute"
  subnet_id = data.yandex_vpc_subnet.default.id
  # ...
}

module "api_vm" {
  source = "./modules/compute"
  extra_disks = module.api_disks.disk_ids  # передаём ID созданных дисков
  # ...
}
```

**Для чего:** `data` блок получает информацию о существующей подсети. Модули обмениваются данными через outputs.

## 4. Переменные и валидация

```hcl
variable "vm_specification" {
  type = map(object({
    cores     = number
    memory    = number
    disk_size = number
    boot_disk = string
  }))

  validation {
    condition     = contains([2, 4], var.vm_spec.cores)
    error_message = "cores must be 2 or 4"
  }
}
```

**Для чего:** `validation` отлавливает ошибки на этапе `plan`, до применения.

## 5. Окружения (`dev` / `prod`)

Два набора `.tfvars` файлов с разными значениями:

| Параметр   | dev    | prod         |
|------------|--------|--------------|
| cores      | 2      | 4            |
| memory     | 2      | 4            |
| disk_type  | network-hdd | network-ssd |

**Для чего:** Один код — разные конфигурации для разработки и продакшена.

## 6. Outputs

```hcl
output "disk_ids" {
  value = module.api_disks.disk_ids
}

output "private_ips" {
  value = {
    "api_vm" = module.api_vm.internal_ip
    "web_vm" = module.web_vm.internal_ip
  }
}
```

**Для чего:** Показывают важную информацию после apply.

## Полезные команды

```bash
# Инициализация окружения
cd environments/dev
terraform init

# Посмотреть план
terraform plan

# Применить
terraform apply

# Посмотреть outputs
terraform output

# Уничтожить всё
terraform destroy

# Переключиться на prod
cd ../prod
```

## Ключевые выводы

1. Модули — переиспользуемый код. Один модуль compute создаёт любые ВМ.
2. for_each — создаёт ресурсы из списка/карты, не сдвигает индексы при удалении.
3. dynamic — генерирует вложенные блоки динамически.
4. data — получает информацию о существующих ресурсах.
5. Окружения — один код, разные значения переменных.
6. validation — проверяет переменные на этапе планирования.
7. Outputs — передают данные между модулями и наружу.

**Кратко:** Модули = Lego для инфраструктуры. `for_each` и `dynamic` = гибкость. Окружения = один код для `dev` и `prod`.