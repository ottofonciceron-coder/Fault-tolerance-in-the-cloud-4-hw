# Домашнее задание к занятию "Отказоустойчивость в облаке" - Марчук Кирилл



### Задание 1 Вместо одной виртуальной машины сделайте terraform playbook, который:
создаст 2 идентичные виртуальные машины. Используйте аргумент count для создания таких ресурсов;
создаст таргет-группу. Поместите в неё созданные на шаге 1 виртуальные машины;
создаст сетевой балансировщик нагрузки, который слушает на порту 80, отправляет трафик на порт 80 виртуальных машин и http healthcheck на порт 80 виртуальных машин.
Установите на созданные виртуальные машины пакет Nginx любым удобным способом и запустите Nginx веб-сервер на порту 80.
Перейдите в веб-консоль Yandex Cloud и убедитесь, что:
созданный балансировщик находится в статусе Active,
обе виртуальные машины в целевой группе находятся в состоянии healthy.
Сделайте запрос на 80 порт на внешний IP-адрес балансировщика и убедитесь, что вы получаете ответ в виде дефолтной страницы Nginx.



1. `Устанавливаем Terraform и YC CLI`
2. `Создаём папку и файлы main.tf ,variables.tf ,outputs.tf ,install_nginx.sh`
3. `Запускаем Terraform`
4. `Делаем запрос к балансировщику`



```
#variables.tf
variable "folder_id" {
  description = "Yandex Cloud Folder ID"
  type        = string
}

variable "zone" {
  description = "Availability zone"
  default     = "ru-central1-a"
}

variable "vm_count" {
  description = "Number of identical VMs"
  default     = 2
}

variable "image_id" {
  description = "Ubuntu 24 image ID"
  default     = "fd81gsj7pb9oi8ks3cvo"
}
```


```
#main.tf
terraform {
  required_providers {
    yandex = {
      source  = "yandex-cloud/yandex"
      version = ">= 0.95"
    }
  }
}

provider "yandex" {
  service_account_key_file = "key.json"
  folder_id                = var.folder_id
  zone                     = var.zone
}

resource "yandex_vpc_network" "net" {
  name = "ha-network"
}

resource "yandex_vpc_subnet" "subnet" {
  name           = "ha-subnet"
  zone           = var.zone
  network_id     = yandex_vpc_network.net.id
  v4_cidr_blocks = ["10.2.0.0/24"]
}

resource "yandex_compute_instance" "vm" {
  count = var.vm_count
  name  = "ha-vm-${count.index}"
  resources {
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = var.image_id
      size     = 10
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.subnet.id
    nat       = true
  }

  metadata = {
    ssh-keys  = "ubuntu:${file("~/.ssh/id_rsa.pub")}"
    user-data = file("install_nginx.sh")
  }
}

resource "yandex_lb_target_group" "tg" {
  name = "ha-target-group"

  dynamic "target" {
    for_each = yandex_compute_instance.vm[*]
    content {
      subnet_id = yandex_vpc_subnet.subnet.id
      address   = target.value.network_interface.0.ip_address
    }
  }
}

resource "yandex_lb_network_load_balancer" "lb" {
  name = "ha-load-balancer"

  listener {
    name        = "http-listener"
    port        = 80
    target_port = 80
  }

  attached_target_group {
    target_group_id = yandex_lb_target_group.tg.id

    healthcheck {
      name = "http"
      http_options {
        port = 80
        path = "/"
      }
    }
  }
}

```


```
#outputs.tf
output "load_balancer_ip" {
  value = flatten([
    for listener in yandex_lb_network_load_balancer.lb.listener :
    [for spec in listener.external_address_spec : spec.address]
  ])
}

output "vm_ips" {
  value = [for vm in yandex_compute_instance.vm : vm.network_interface[0].nat_ip_address]
}
```


```
#install_nginx.sh
#!/bin/bash
sudo apt update -y
sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx

```

---


![zadanie1](https://github.com/ottofonciceron-coder/Fault-tolerance-in-the-cloud-4-hw/blob/main/balancer.png)`

![zadanie1.2](https://github.com/ottofonciceron-coder/Fault-tolerance-in-the-cloud-4-hw/blob/main/nginx.png)`


---
