# Руководство по развертыванию кластера Kubernetes

Данное руководство содержит подробные инструкции по развертыванию кластера Kubernetes с использованием VirtualBox, Debian 12, kubeadm, CNI Calico и ingress nginx. Кластер будет состоять из 1 мастер-ноды и 1 рабочей ноды, с настроенным доступом по SSH и возможностью управления через kubectl с хостовой машины. Так же будет установлен и настроен Kubernetes Dashboard.

## Содержание

1. [Настройка сети VirtualBox Host-Only](#настройка-сети-virtualbox-host-only)
2. [Подготовка виртуальных машин Debian 12](#подготовка-виртуальных-машин-debian-12)
3. [Инициализация кластера Kubernetes с помощью kubeadm](#инициализация-кластера-kubernetes-с-помощью-kubeadm)
4. [Установка CNI Calico и Ingress Nginx](#установка-cni-calico-и-ingress-nginx)
5. [Развертывание Nginx с Service и Ingress](#развертывание-nginx-с-service-и-ingress)
6. [Установка и настройка Kubernetes Dashboard](#установка-и-настройка-kubernetes-dashboard)
7. [Валидация кластера и проверка доступности](#валидация-кластера-и-проверка-доступности)

## Настройка сети VirtualBox Host-Only

Для начала необходимо настроить сеть VirtualBox Host-Only, которая будет использоваться для связи между нодами кластера и для доступа к кластеру с хостовой машины.

### Создание сети Host-Only в VirtualBox

1. Откройте VirtualBox
2. Перейдите в меню "Файл" -> "Настройки" (или "File" -> "Preferences")
3. Выберите вкладку "Сеть" (или "Network")
4. Перейдите на вкладку "Сети хоста" (или "Host-only Networks")
5. Нажмите на кнопку "Добавить сеть хоста" (значок с плюсом)
6. Настройте созданную сеть:
   - Имя: VirtualBox Host-Only Ethernet Adapter (это имя будет назначено автоматически при создании сети)
   - IPv4 адрес: 192.168.56.1
   - Маска сети: 255.255.255.0
   - DHCP-сервер: отключен (снимите галочку, если она установлена)
7. Нажмите "OK" для сохранения настроек

### Конфигурация IP-адресов для нод кластера

Для нашего кластера Kubernetes мы будем использовать следующие IP-адреса:

- Мастер-нода (master): 192.168.56.10
- Рабочая нода (worker1): 192.168.56.11

## Подготовка виртуальных машин Debian 12

### Требования к виртуальным машинам

#### Мастер-нода (master)
- **CPU**: 2 ядра
- **RAM**: 4 ГБ
- **Диск**: 25 ГБ
- **Сеть**: 2 сетевых адаптера (NAT и Host-Only)
- **Имя хоста**: k8s-master

#### Рабочая нода (worker1)
- **CPU**: 2 ядра
- **RAM**: 4 ГБ
- **Диск**: 35 ГБ
- **Сеть**: 2 сетевых адаптера (NAT и Host-Only)
- **Имя хоста**: k8s-worker1

### Создание виртуальных машин в VirtualBox

#### Создание мастер-ноды

1. Откройте VirtualBox и нажмите "Создать" (или "New")
2. Укажите следующие параметры:
   - Имя: k8s-master
   - Тип: Linux
   - Версия: Debian (64-bit)
   - Память: 4096 МБ (4 ГБ)
   - Жесткий диск: Создать новый виртуальный жесткий диск
   - Тип файла жесткого диска: VDI
   - Размер: 25 ГБ
   - Тип хранения: Динамически выделяемый
3. Нажмите "Создать"
4. Выберите созданную виртуальную машину и нажмите "Настроить" (или "Settings")
5. В разделе "Система" -> "Процессор" установите количество процессоров: 2
6. В разделе "Сеть":
   - Адаптер 1: Включен, тип подключения "NAT"
   - Адаптер 2: Включен, тип подключения "Виртуальный адаптер", выберите созданную ранее сеть VirtualBox Host-Only Ethernet Adapter
7. Нажмите "OK" для сохранения настроек

#### Создание рабочей ноды

1. Повторите шаги 1-3 из создания мастер-ноды, но укажите имя "k8s-worker1" и размер жесткого диска "35 ГБ"
2. Выполните шаги 4-7 аналогично мастер-ноде

### Установка Debian 12 на виртуальные машины

#### Подготовка к установке

1. Скачайте ISO-образ Debian 12 с официального сайта: https://cdimage.debian.org/debian-cd/current/amd64/iso-dvd/debian-12.11.0-amd64-DVD-1.iso
2. В VirtualBox выберите виртуальную машину k8s-master
3. Нажмите "Настроить" -> "Хранение"
4. В разделе "Контроллер IDE" выберите пустой диск
5. Нажмите на значок диска справа и выберите скачанный ISO-образ Debian 12
6. Нажмите "OK" для сохранения настроек

#### Установка Debian 12 на мастер-ноду

1. Запустите виртуальную машину k8s-master
2. Выберите "Install" в меню загрузки
3. Следуйте инструкциям установщика:
   - Выберите язык, местоположение и раскладку клавиатуры
   - Укажите имя хоста: k8s-master
   - Укажите доменное имя (можно оставить пустым)
   - Создайте пароль для root
   - Введите имя обычного пользователя: k8s. Создайте для него пароль
   - Выберите метод разметки диска: "Использовать весь диск"
   - Выберите схему разделов: "Все файлы в одном разделе"
   - Завершите разметку диска и запишите изменения
   - Дождитесь установки базовой системы
   - При выборе программного обеспечения отметьте только "SSH-сервер" и "стандартные системные утилиты"
   - Установите GRUB в MBR
   - Завершите установку и перезагрузите систему

4. После перезагрузки войдите в систему с учетными данными, которые вы создали

#### Установка Debian 12 на рабочую ноду

1. Повторите шаги из раздела "Подготовка к установке" для виртуальной машины k8s-worker1
2. Повторите шаги из раздела "Установка Debian 12 на мастер-ноду", но укажите имя хоста k8s-worker1

### Настройка сети на виртуальных машинах

#### Настройка сети на мастер-ноде

1. Войдите в систему на мастер-ноде под обычным пользователем k8s

2. Т.к. по умолчанию в установленной ОС отсутствует sudo, будем использовать su. И так, заходим под root (вводим пароль):
   ```bash
   su
   Password:
   ```

3. Откройте файл конфигурации сети:
   ```bash
   nano /etc/network/interfaces
   ```

4. Добавьте следующие строки:
   ```
   # K8s Net
   auto enp0s8
   iface enp0s8 inet static
       address 192.168.56.10
       netmask 255.255.255.0
   ```

5. Сохраните файл (Ctrl+O, затем Enter) и выйдите из редактора (Ctrl+X)

6. Перезапустите сетевую службу:
   ```bash
   systemctl restart networking
   ```

#### Настройка сети на рабочей ноде

1. Повторите шаги 1-6 из раздела "Настройка сети на мастер-ноде" для рабочей ноды, но укажите IP-адрес 192.168.56.11 вместо 192.168.56.10

### Настройка имен хостов и DNS

#### На обеих нодах выполните следующие команды:

1. Отредактируйте файл /etc/hosts:
   ```bash
   nano /etc/hosts
   ```

2. Добавьте следующие строки:
   ```
   127.0.0.1       localhost
   192.168.56.10   k8s-master
   192.168.56.11   k8s-worker1
   ```

3. Сохраните файл и выйдите из редактора

### Установка необходимых пакетов

#### На обеих нодах выполните следующие команды:

1. Отредактируйте файл /etc/apt/sources.list:
   ```bash
   nano /etc/apt/sources.list
   ```

2. Закомментируйте первую строку, начинающуюся на "deb cdrom":
   ```bash
   # deb cdrom:[Debian GNU/Linux 12.11.0 _Bookworm_ - Official amd64 DVD Binary-1 with firmware 20250517-09:52]/ bookworm contrib main non-free-firmware
   ```

3. Обновите список пакетов и сами пакеты (если обновления есть):
   ```bash
   apt update && apt upgrade -y
   ```

4. Установите необходимые пакеты:
   ```bash
   apt install -y apt-transport-https ca-certificates curl gnupg lsb-release software-properties-common sudo ufw
   ```

### Настройка команды sudo для пользователя k8s и настройка брандмауэра ufw

#### На обеих нодах выполните следующие команды:

1. Настройте sudo для работы без пароля (для удобства дальнейшей настройки):
   ```bash
   echo "k8s ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/k8s
   chmod 440 /etc/sudoers.d/k8s
   ```

2. Выйдите из сеанса root, а затем выйдите из системы и войдите снова под k8s, чтобы изменения вступили в силу

3. Проверьте работу sudo:
   ```bash
   sudo whoami
   ```
   Вы должны увидеть вывод "root", и система не должна запрашивать пароль (если вы настроили NOPASSWD).

4. Настройте базовые правила ufw:
   ```bash
   sudo ufw default deny incoming
   sudo ufw default allow outgoing
   ```

5. Разрешите SSH-соединения:
   ```bash
   sudo ufw allow ssh
   ```

#### Открытие портов для мастер-ноды

На мастер-ноде выполните следующие команды:

```bash
# Kubernetes API server
sudo ufw allow 6443/tcp

# etcd server client API
sudo ufw allow 2379:2380/tcp

# Kubelet API
sudo ufw allow 10250/tcp

# kube-scheduler
sudo ufw allow 10251/tcp

# kube-controller-manager
sudo ufw allow 10252/tcp

# NodePort Services
sudo ufw allow 30000:32767/tcp

# Calico BGP
sudo ufw allow 179/tcp

# Calico VXLAN
sudo ufw allow 4789/udp
sudo ufw allow 4789/tcp

# Calico Typha
sudo ufw allow 5473/tcp
```

#### Открытие портов для рабочей ноды

На рабочей ноде выполните следующие команды:

```bash
# Kubelet API
sudo ufw allow 10250/tcp

# NodePort Services
sudo ufw allow 30000:32767/tcp

# Calico BGP
sudo ufw allow 179/tcp

# Calico VXLAN
sudo ufw allow 4789/udp
sudo ufw allow 4789/tcp

# Calico Typha
sudo ufw allow 5473/tcp
```

#### Активация брандмауэра

На обеих нодах выполните:
```bash
sudo ufw enable
```
Подтвердите активацию, введя "y".

#### Проверка статуса брандмауэра

```bash
sudo ufw status verbose
```
Вы должны увидеть список разрешенных портов и сервисов.

#### Альтернативный вариант: отключение брандмауэра

Если вы работаете в изолированной тестовой среде, вы можете полностью отключить брандмауэр для упрощения настройки:
```bash
sudo ufw disable
```
> **Примечание**: Отключение брандмауэра не рекомендуется для производственных сред.

#### Проверка открытых портов

Чтобы убедиться, что порты открыты и доступны, вы можете использовать команду `netstat`:
1. Установите netstat:
   ```bash
   sudo apt install -y net-tools
   ```

2. Проверьте открытые порты:
   ```bash
   sudo netstat -tulpn
   ```

### Установка и настройка containerd

#### На обеих нодах выполните следующие команды:

1. Загрузите и распакуйте архив последней релизной версии containerd:
   ```bash
   wget https://github.com/containerd/containerd/releases/download/v2.1.1/containerd-2.1.1-linux-amd64.tar.gz
   sudo tar Cxzvf /usr/local containerd-2.1.1-linux-amd64.tar.gz
   ```

2. Загрузите и установите файл запуска службы containerd, затем запустите службу:
   ```bash
   sudo wget -O /etc/systemd/system/containerd.service https://raw.githubusercontent.com/containerd/containerd/main/containerd.service

   # Перечитаем файлы
   sudo systemctl daemon-reload

   # Запуск службы
   sudo systemctl enable --now containerd
   ```

3. Загрузите и установите последнюю релизную версию runc:
   ```bash
   wget https://github.com/opencontainers/runc/releases/download/v1.3.0/runc.amd64
   sudo install -m 755 runc.amd64 /usr/local/sbin/runc
   ```

4. Загрузите и установите последнюю версию CNI plugins:
   ```bash
   wget https://github.com/containernetworking/plugins/releases/download/v1.7.1/cni-plugins-linux-amd64-v1.7.1.tgz
   sudo mkdir -p /opt/cni/bin
   sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.7.1.tgz
   ```

5. Сгенерируйте дефолтный файл конфигурации containerd:
   ```bash
   sudo mkdir -p /etc/containerd
   sudo containerd config default | sudo tee /etc/containerd/config.toml
   ```

6. Отредактируйте конфигурацию containerd:
   ```bash
   sudo nano /etc/containerd/config.toml
   ```
   Найдите раздел `[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]` и добавьте параметр `SystemdCgroup = true` в конце раздела после параметра `ShimCgroup = ''`. Сохраните файл и перезапустите службу:
   ```bash
   sudo systemctl restart containerd
   ```

7. Включите автозапуск containerd:
   ```bash
   sudo systemctl enable containerd
   ```

### Настройка параметров ядра для Kubernetes

#### На обеих нодах выполните следующие команды:

1. Загрузите модуль overlay и br_netfilter:
   ```bash
   cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
   overlay
   br_netfilter
   EOF

   sudo modprobe overlay
   sudo modprobe br_netfilter
   ```

2. Настройте параметры sysctl для Kubernetes:
   ```bash
   cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
   net.bridge.bridge-nf-call-iptables = 1
   net.bridge.bridge-nf-call-ip6tables = 1
   net.ipv4.ip_forward = 1
   EOF

   sudo sysctl --system
   ```

3. Отключите swap:
   ```bash
   sudo swapoff -a
   ```

4. Отредактируйте файл /etc/fstab, чтобы отключить swap при загрузке:
   ```bash
   sudo nano /etc/fstab
   ```

5. Закомментируйте строку, содержащую swap, добавив символ # в начале строки

6. Сохраните файл и выйдите из редактора

### Настройка SSH-доступа

#### На обеих нодах выполните следующие команды:

1. Убедитесь, что SSH-сервер установлен и запущен:
   ```bash
   sudo apt install -y openssh-server
   sudo systemctl enable sshd
   sudo systemctl start sshd
   ```

2. Отредактируйте основной файл конфигурации sshd:
   ```bash
   sudo nano /etc/ssh/sshd_config
   ```

3. Закомментируйте следующие строки:
   ```bash
   #KbdInteractiveAuthentication no
   #UsePAM yes
   #X11Forwarding yes
   ```

4. Создайте и отредактируйте файл /etc/ssh/sshd_config.d/00-k8s.conf:
   ```bash
   sudo nano /etc/ssh/sshd_config.d/00-k8s.conf

   # Вставьте следующие строки и сохраните файл:
   Port 22
   AddressFamily inet
   AllowUsers k8s
   Protocol 2
   LoginGraceTime 60
   PermitRootLogin no
   MaxAuthTries 3
   MaxSessions 10
   PubkeyAuthentication yes
   AuthorizedKeysFile      .ssh/authorized_keys
   PasswordAuthentication yes
   PermitEmptyPasswords no
   KbdInteractiveAuthentication no
   UsePAM yes
   X11Forwarding yes
   TCPKeepAlive no
   ClientAliveInterval 20
   ClientAliveCountMax 3
   MaxStartups 3:30:10
   ```

5. Перезапустите SSH-сервер:
   ```bash
   sudo systemctl restart sshd
   ```

#### Настройка SSH-ключей (опционально, но рекомендуется)

1. На хостовой машине сгенерируйте SSH-ключи (если их еще нет):
   ```bash
   ssh-keygen -t ed25519 -C "k8s"
   ```

2. Скопируйте публичный ключ на обе ноды:
   ```bash
   cat ~/.ssh/k8s.pub | ssh k8s@192.168.56.10 'cat >> ~/.ssh/authorized_keys'  # Для мастер-ноды
   cat ~/.ssh/k8s.pub | ssh k8s@192.168.56.11 'cat >> ~/.ssh/authorized_keys'  # Для рабочей ноды
   ```

3. Проверьте подключение:
   ```bash
   ssh k8s@192.168.56.10 -i ~/.ssh/k8s
   ssh k8s@192.168.56.11 -i ~/.ssh/k8s
   ```

## Инициализация кластера Kubernetes с помощью kubeadm

### Установка kubeadm, kubelet и kubectl

#### На обеих нодах выполните следующие команды:

1. Добавьте ключ GPG для репозитория Kubernetes (в Debian 12 по умолчанию нет папки /etc/apt/keyrings):
   ```bash
   sudo mkdir -p -m 755 /etc/apt/keyrings
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
   ```

2. Добавьте репозиторий Kubernetes:
   ```bash
   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
   ```

3. Обновите список пакетов:
   ```bash
   sudo apt-get update
   ```

4. Установите kubeadm, kubelet и kubectl:
   ```bash
   sudo apt-get install -y kubelet kubeadm kubectl
   ```

5. Зафиксируйте версии пакетов, чтобы предотвратить их автоматическое обновление:
   ```bash
   sudo apt-mark hold kubelet kubeadm kubectl
   ```

### Инициализация мастер-ноды

#### На мастер-ноде выполните следующие команды:

1. Инициализируйте кластер Kubernetes:
   ```bash
   sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=192.168.56.10 --control-plane-endpoint=192.168.56.10
   ```

   > Примечание: Мы используем CIDR 192.168.0.0/16 для Calico. Если вы планируете использовать другой CNI, возможно, потребуется другой CIDR.

2. После успешной инициализации вы увидите сообщение с инструкциями. Выполните следующие команды для настройки kubectl:
   ```bash
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

3. Проверьте статус кластера:
   ```bash
   kubectl get nodes
   ```

   Вы должны увидеть мастер-ноду со статусом "NotReady" (статус изменится на "Ready" после установки сетевого плагина).

4. Сохраните команду присоединения, которая была выведена после инициализации. Она будет выглядеть примерно так:
   ```bash
   sudo kubeadm join 192.168.56.10:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
   ```

   > Примечание: Если вы потеряли команду присоединения, вы можете создать новый токен:
   > ```bash
   > sudo kubeadm token create --print-join-command
   > ```

### Присоединение рабочей ноды к кластеру

#### На рабочей ноде выполните следующие команды:

1. Выполните команду присоединения, которую вы получили на предыдущем шаге:
   ```bash
   sudo kubeadm join 192.168.56.10:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
   ```

2. После успешного присоединения вы увидите сообщение о том, что нода присоединилась к кластеру.

### Настройка доступа к кластеру с хостовой машины

Чтобы управлять кластером с хостовой машины, вам нужно скопировать файл конфигурации kubectl с мастер-ноды на хостовую машину.

#### На мастер-ноде:

1. Отобразите содержимое файла конфигурации:
   ```bash
   cat $HOME/.kube/config
   ```

2. Скопируйте содержимое файла.

#### На хостовой машине:

1. Установите kubectl, если он еще не установлен:
   - Для Linux:
     ```bash
     curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/v1.33.1/bin/linux/amd64/kubectl"
     chmod +x kubectl
     sudo mv kubectl /usr/local/bin/
     ```
   - Для macOS:
     ```bash
     brew install kubectl
     ```
   - Для Windows:
     ```powershell
     curl -LO "https://dl.k8s.io/v1.33.1/bin/windows/amd64/kubectl.exe"
     ```

2. Создайте директорию для конфигурации kubectl:
   - Для Linux/macOS:
     ```bash
     mkdir -p $HOME/.kube
     ```
   - Для Windows:
     ```powershell
     mkdir -Force $HOME\.kube
     ```

3. Создайте файл конфигурации:
   - Для Linux/macOS:
     ```bash
     nano $HOME/.kube/config
     ```
   - Для Windows:
     ```powershell
     notepad $HOME\.kube\config
     ```

4. Вставьте скопированное содержимое файла конфигурации.

5. Измените IP-адрес сервера API в файле конфигурации с `127.0.0.1` на `192.168.56.10` (IP-адрес мастер-ноды).

6. Сохраните файл и закройте редактор.

7. Проверьте подключение к кластеру:
   ```bash
   kubectl get nodes
   ```

#### Альтернативный способ: использование scp

Вместо ручного копирования и вставки, вы можете использовать scp для копирования файла конфигурации:

1. На хостовой машине выполните:
   ```bash
   mkdir -p $HOME/.kube
   scp username@192.168.56.10:~/.kube/config $HOME/.kube/config
   ```

2. Отредактируйте файл конфигурации, чтобы изменить IP-адрес сервера API:
   ```bash
   sed -i 's/127.0.0.1/192.168.56.10/g' $HOME/.kube/config
   ```

## Установка CNI Calico и Ingress Nginx

### Установка CNI Calico

#### На мастер-ноде выполните следующие команды:

1. Скачайте манифест Calico:
   ```bash
   curl https://raw.githubusercontent.com/projectcalico/calico/v3.30.0/manifests/calico.yaml -O
   ```

2. Примените манифест Calico к вашему кластеру:
   ```bash
   kubectl apply -f calico.yaml
   ```

   > Примечание: Этот манифест настроен для использования CIDR 192.168.0.0/16, который мы указали при инициализации кластера. Если вы использовали другой CIDR, вам нужно будет отредактировать этот файл перед применением.

3. Проверьте статус установки Calico:
   ```bash
   kubectl get pods -n calico-system
   ```

   Дождитесь, пока все поды перейдут в состояние "Running".

4. Проверьте статус нод:
   ```bash
   kubectl get nodes
   ```

   Теперь ноды должны иметь статус "Ready".

### Установка Ingress Nginx

#### На мастер-ноде выполните следующие команды:

1. Установите Ingress Nginx с помощью манифеста:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.2/deploy/static/provider/baremetal/deploy.yaml
   ```

2. Проверьте статус установки Ingress Nginx:
   ```bash
   kubectl get pods -n ingress-nginx
   ```

   Дождитесь, пока поды `admission-create` и `admission-patch` перейдут в состояние "Completed", а `controller` - в состояние "Running".

3. Проверьте, что сервис Ingress Nginx создан:
   ```bash
   kubectl get svc -n ingress-nginx
   ```

   Вы должны увидеть сервис "ingress-nginx-controller" типа NodePort.

### Настройка доступа к Ingress Nginx с хостовой машины

1. Получите информацию о сервисе Ingress Nginx:
   ```bash
   kubectl get svc -n ingress-nginx ingress-nginx-controller
   ```

   Вы увидите порты, на которых доступен Ingress Nginx. Обычно это порты в диапазоне 30000-32767.

2. Запомните порт HTTP (обычно это порт, сопоставленный с портом 80) и порт HTTPS (обычно это порт, сопоставленный с портом 443).

3. Теперь вы можете получить доступ к Ingress Nginx с хостовой машины по адресу:
   - HTTP: http://192.168.56.10:<HTTP_PORT>
   - HTTPS: https://192.168.56.10:<HTTPS_PORT>

   Где <HTTP_PORT> и <HTTPS_PORT> - это порты, которые вы получили на предыдущем шаге.

## Развертывание Nginx с Service и Ingress

### Создание манифеста deployment, service и ingress - Nginx

1. Создайте файл `nginx.yml` на мастер-ноде или на хосте:
   ```bash
   cat <<EOF > nginx.yml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
   name: nginx
   spec:
   replicas: 2
   selector:
      matchLabels:
         app: nginx
   template:
      metadata:
         labels:
         app: nginx
      spec:
         containers:
         - name: nginx
         image: nginx
         resources:
            limits:
               memory: "64Mi"
               cpu: "50m"
         ports:
         - containerPort: 80
   ---
   apiVersion: v1
   kind: Service
   metadata:
   name: nginx
   spec:
   selector:
      app: nginx
   ports:
   - port: 80
      targetPort: 80
   ---
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
   name: nginx-ingress
   annotations:
      nginx.ingress.kubernetes.io/rewrite-target: /
   spec:
   ingressClassName: nginx
   rules:
   - http:
         paths:
         - path: /
         pathType: Prefix
         backend:
            service:
               name: nginx
               port:
               number: 80
   EOF
   ```

2. Примените манифест:
   ```bash
   kubectl apply -f nginx.yml
   ```

3. Проверьте статус деплоймента:
   ```bash
   kubectl get deployments
   ```

   Вы должны увидеть деплоймент `nginx` с 2 репликами.

4. Проверьте статус сервиса:
   ```bash
   kubectl get services
   ```

   Вы должны увидеть сервис `nginx` типа ClusterIP.

5. Проверьте статус Ingress-ресурса:
   ```bash
   kubectl get ingress
   ```

   Вы должны увидеть Ingress-ресурс `nginx-ingress`.

### Проверка доступности Nginx через Ingress

1. Получите порт HTTP Ingress Nginx:
   ```bash
   kubectl get svc -n ingress-nginx ingress-nginx-controller
   ```

   Запомните порт, сопоставленный с портом 80 (обычно это порт в диапазоне 30000-32767).

2. Проверьте доступ к Nginx с хостовой машины:
   ```bash
   curl http://192.168.56.10:<HTTP_PORT>/
   ```

   Где <HTTP_PORT> - это порт HTTP, который вы получили на предыдущем шаге.

   Вы должны увидеть страницу приветствия Nginx.

## Установка и настройка Kubernetes Dashboard

### Установка Helm

#### На мастер-ноде выполните следующие команды:

1. Скачайте сценарий установки Helm:
   ```bash
   curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
   ```

2. Выдайте ему необходимые права для запуска:
   ```bash
   chmod 700 get_helm.sh
   ```

3. Запустите сценарий:
   ```bash
   ./get_helm.sh
   ```

### Установка Kubernetes Dashboard

#### На мастер-ноде выполните следующие команды:

1. Добавьте helm-репозиторий Kubernetes Dashboard:
   ```bash
   helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
   ```

2. Установите Kubernetes Dashboard с помощью Helm:
   ```bash
   helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
   ```

### Создание сервисного аккаунта и роли для доступа к Dashboard

1. Создайте файл `dashboard-adminuser.yaml` на мастер-ноде:
   ```bash
   cat <<EOF > dashboard-adminuser.yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: admin-user
     namespace: kubernetes-dashboard
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: admin-user
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: cluster-admin
   subjects:
   - kind: ServiceAccount
     name: admin-user
     namespace: kubernetes-dashboard
   EOF
   ```

2. Примените манифест:
   ```bash
   kubectl apply -f dashboard-adminuser.yaml
   ```

3. Создайте токен для сервисного аккаунта:
   ```bash
   kubectl -n kubernetes-dashboard create token admin-user
   ```

   Сохраните этот токен, он понадобится для входа в Dashboard.

### Настройка доступа к Dashboard с хостовой машины

#### Способ 1: Изменение типа сервиса на NodePort

1. Отредактируйте сервис kubernetes-dashboard-kong-proxy (измените тип сервиса с `ClusterIP` на `NodePort`):
   ```bash
   kubectl -n kubernetes-dashboard patch svc kubernetes-dashboard-kong-proxy -p '{"spec": {"type": "NodePort"}}'
   ```

2. Получите порт, на котором доступен Dashboard:
   ```bash
   kubectl -n kubernetes-dashboard get svc kubernetes-dashboard-kong-proxy
   ```

   Запомните порт, сопоставленный с портом 443 (обычно это порт в диапазоне 30000-32767).

5. Теперь Dashboard доступен по адресу:
   https://192.168.56.10:<NODE_PORT>/

   Где <NODEPORT> - это порт, который вы получили на предыдущем шаге.

   > Примечание: Браузер выдаст предупреждение о небезопасном соединении, так как используется самоподписанный сертификат. Вы можете проигнорировать это предупреждение и продолжить.

#### Способ 2: Создание Ingress для Dashboard

1. Создайте файл `dashboard-ingress.yaml` на мастер-ноде:
   ```bash
   cat <<EOF > dashboard-ingress.yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: kubernetes-dashboard
     namespace: kubernetes-dashboard
     annotations:
       nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
       nginx.ingress.kubernetes.io/ssl-passthrough: "true"
   spec:
     ingressClassName: nginx
     rules:
     - host: dashboard.example.com
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: kubernetes-dashboard
               port:
                 number: 443
   EOF
   ```

2. Примените манифест:
   ```bash
   kubectl apply -f dashboard-ingress.yaml
   ```

3. Добавьте запись в файл hosts на хостовой машине:
   ```
   192.168.56.10 dashboard.example.com
   ```

4. Получите порт HTTPS Ingress Nginx:
   ```bash
   kubectl get svc -n ingress-nginx ingress-nginx-controller
   ```

   Запомните порт, сопоставленный с портом 443 (обычно это порт в диапазоне 30000-32767).

5. Теперь Dashboard доступен по адресу:
   https://dashboard.example.com:<HTTPS_PORT>/

   Где <HTTPS_PORT> - это порт HTTPS, который вы получили на предыдущем шаге.

### Вход в Kubernetes Dashboard

1. Откройте Dashboard в браузере, используя один из способов, описанных выше.

2. На странице входа выберите "Token" и введите токен, который вы получили ранее.

3. Нажмите "Sign in".

   Теперь вы должны увидеть интерфейс Kubernetes Dashboard.

## Валидация кластера и проверка доступности

### Проверка работоспособности компонентов кластера

#### Проверка статуса нод

На мастер-ноде или на хостовой машине (если у вас настроен kubectl) выполните:

```bash
kubectl get nodes
```

Вы должны увидеть обе ноды (мастер и рабочую) со статусом "Ready".

#### Проверка статуса подов системы

```bash
kubectl get pods -A
```

Вы должны увидеть все поды системы в состоянии "Running".

### Проверка доступности Nginx через Ingress

1. Получите порт HTTP Ingress Nginx:
   ```bash
   kubectl get svc -n ingress-nginx ingress-nginx-controller
   ```

2. Проверьте доступ к Nginx с хостовой машины:
   ```bash
   curl http://192.168.56.10:<HTTP_PORT>/
   ```

   Где <HTTP_PORT> - это порт HTTP, который вы получили на предыдущем шаге.

   Вы должны увидеть страницу приветствия Nginx.

### Проверка доступности Dashboard через браузер

1. Получите порт HTTPS для Dashboard (если вы настроили Dashboard как NodePort):
   ```bash
   kubectl -n kubernetes-dashboard get svc kubernetes-dashboard-kong-proxy
   ```

2. Откройте в браузере:
   ```
   https://192.168.56.10:<NODEPORT>/
   ```

   Где <NODEPORT> - это порт, сопоставленный с портом 443.

   Вы должны увидеть страницу входа в Dashboard. Используйте токен, который вы создали ранее, для входа.

### Проверка доступа к кластеру через kubectl с хостовой машины

Если вы настроили kubectl на хостовой машине, выполните:

```bash
kubectl get nodes
kubectl get pods -A
kubectl get services -A
```

Вы должны увидеть те же результаты, что и при выполнении этих команд на мастер-ноде.

## Заключение

Поздравляем! Вы успешно развернули кластер Kubernetes с 1 мастер-нодой и 1 рабочей нодой, установили CNI Calico и Ingress Nginx, развернули официальный контейнер Nginx с Service и Ingress, а также установили и настроили Kubernetes Dashboard.

Теперь у вас есть полностью функциональный кластер Kubernetes, который вы можете использовать для развертывания и управления контейнерными приложениями.
