# Домашнее задание: [Отказоустойчивость в облаке]
**Студент:** [Захарян Николай Артурович]

---

## Задание: Развертывание Network Load Balancer (NLB) в Yandex Cloud с помощью Terraform

### 1. Terraform Playbook (main.tf)
Ниже представлен финальный код конфигурации `main.tf`, который создает сеть, две ВМ с Nginx, целевую группу и сетевой балансировщик:

```hcl
terraform {
  required_providers {
    yandex = {
      source  = "yandex-cloud/yandex"
      version = "= 0.229.0"
    }
  }
}

provider "yandex" {
  zone = "ru-central1-a"
}

resource "yandex_vpc_network" "nlb-network" {
  name = "nlb-network"
}

resource "yandex_vpc_subnet" "nlb-subnet" {
  name           = "nlb-subnet"
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.nlb-network.id
  v4_cidr_blocks = ["192.168.10.0/24"]
}

data "yandex_compute_image" "ubuntu" {
  family = "ubuntu-2204-lts"
}

resource "yandex_compute_instance" "nlb-vm" {
  count = 2
  name  = "nlb-vm-${count.index}"

  platform_id = "standard-v3"
  
  resources {
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = data.yandex_compute_image.ubuntu.id
      type     = "network-hdd"
      size     = 10
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.nlb-subnet.id
    nat       = true
  }

  metadata = {
    user-data = <<-EOT
      #cloud-config
      package_update: true
      packages:
        - nginx
      runcmd:
        - systemctl enable nginx
        - systemctl start nginx
    EOT
  }
}

resource "yandex_lb_target_group" "nlb-tg" {
  name      = "nlb-target-group"
  region_id = "ru-central1"

  dynamic "target" {
    for_each = yandex_compute_instance.nlb-vm
    content {
      subnet_id = yandex_vpc_subnet.nlb-subnet.id
      address   = target.value.network_interface[0].ip_address
    }
  }
}

resource "yandex_lb_network_load_balancer" "nlb" {
  name = "nlb-balancer"

  listener {
    name        = "http-listener"
    port        = 80
    target_port = 80
    protocol    = "tcp"

    external_address {
      ip_version = "ipv4"
    }
  }

  attached_target_group {
    target_group_id = yandex_lb_target_group.nlb-tg.id

    healthcheck {
      name = "http-check"
      http_options {
        port = 80
        path = "/"
      }
    }
  }
}

output "nlb_external_ip" {
  value       = tolist(yandex_lb_network_load_balancer.nlb.listener)[0].external_address_spec[0].address
  description = "Внешний IP-адрес сетевого балансировщика"
}
