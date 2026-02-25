Домашнее задание к занятию «Отказоустойчивость в облаке» Николай Чернов

Задание 1

Возьмите за основу решение к заданию 1 из занятия «Подъём инфраструктуры в Яндекс Облаке».

Теперь вместо одной виртуальной машины сделайте terraform playbook, который:
создаст 2 идентичные виртуальные машины. Используйте аргумент count для создания таких ресурсов;
создаст таргет-группу. Поместите в неё созданные на шаге 1 виртуальные машины;
создаст сетевой балансировщик нагрузки, который слушает на порту 80, отправляет трафик на порт 80 виртуальных машин и http healthcheck на порт 80 виртуальных машин.
Рекомендуем изучить документацию сетевого балансировщика нагрузки для того, чтобы было понятно, что вы сделали.

Установите на созданные виртуальные машины пакет Nginx любым удобным способом и запустите Nginx веб-сервер на порту 80.

Перейдите в веб-консоль Yandex Cloud и убедитесь, что:

созданный балансировщик находится в статусе Active,
обе виртуальные машины в целевой группе находятся в состоянии healthy.
Сделайте запрос на 80 порт на внешний IP-адрес балансировщика и убедитесь, что вы получаете ответ в виде дефолтной страницы Nginx.
В качестве результата пришлите:

1. Terraform Playbook.

terraform {
  required_providers {
    yandex = {
      source  = "yandex-cloud/yandex"
      version = "~> 0.90"
    }
  }
}

provider "yandex" {
  token     = var.yc_token
  cloud_id = var.yc_cloud_id
  folder_id = var.yc_folder_id
}

# Сетевые ресурсы
resource "yandex_vpc_network" "default" {
  name = "lb-network"
}

resource "yandex_vpc_subnet" "default" {
  name           = "lb-subnet"
  zone           = var.zone
  network_id     = yandex_vpc_network.default.id
  v4_cidr_blocks = ["10.128.0.0/24"]
}

resource "yandex_compute_instance" "web-server" {
  count = 2

  name = "web-server-${count.index + 1}"
  platform_id = "standard-v3"
  zone  = var.zone

  resources {
    core_fraction = 5
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = var.image_id
      size = 10
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.default.id
    nat      = true
  }

  metadata = {
    ssh-keys = "ubuntu:${var.ssh_public_key}"
    user-data = file("${path.module}/userdata.sh")
  }
}

resource "yandex_lb_target_group" "default" {
  name = "web-target-group"

  dynamic "target" {
    for_each = [for instance in yandex_compute_instance.web-server : instance.network_interface[0].primary_v4_address.0.one_to_one_nat.0.address]
    content {
      address = target.value
      subnet_id = yandex_vpc_subnet.default.id
    }
  }
}

resource "yandex_lb_network_load_balancer" "nlb" {
  name = "web-nlb"

  listener {
    name = "http-listener"
    port = 80

    external_address_spec {
      ip_version = "ipv4"
    }
  }

  attached_target_group {
    target_group_id = yandex_lb_target_group.default.id

    healthcheck {
      name = "http"
      http_options {
        port = 80
        path = "/"
      }
    }
  }
}

2. Скриншот статуса балансировщика и целевой группы.

3. Скриншот страницы, которая открылась при запросе IP-адреса балансировщика.
