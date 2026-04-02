# Подготовка к заданию

Сначала я арендовал VPS на Hostkey, подключился в шелле:

<img width="1179" height="578" alt="image" src="https://github.com/user-attachments/assets/f44faccb-5c59-4121-958f-49699475b183" />

По задани установил все зависимости.

Далее я скачал микротик, создал машину в VBOX:

<img width="1134" height="419" alt="Снимок экрана 2026-04-02 114153" src="https://github.com/user-attachments/assets/4de871b8-8d2e-4d2c-86d1-160bca9e69b4" />

Также сказал WinBox для удобства, для микротика выбрал сетевой мост, спокойно подключился через это приложение

# Ход работы

Генерация ключей:

```
sudo mkdir -p /etc/wireguard/keys
wg genkey | sudo tee /etc/wireguard/keys/server_private.key | wg pubkey | sudo tee /etc/wireguard/keys/server_public.key
```

В CHR настройка wireguard:

```
/interface wireguard add name=wireguard0 listen-port=51820
```

Назначение айпи адреса ваергарду:

```
/ip address add address=10.20.0.2/24 interface=wireguard0
```

Публичный и приватный ключ создались автоматически

Потом при помощи GUI я настроил peer:

<img width="647" height="898" alt="image" src="https://github.com/user-attachments/assets/67e3b6b3-cf22-4278-913e-250f954bbc04" />

Публичный ключ был взят из VPS

Далее посмотрел публичный ключ в CHR, это нужно, чтобы потом составить конфиг в VPS:

<img width="776" height="131" alt="image" src="https://github.com/user-attachments/assets/cf0ea2c1-fdda-40a7-9db1-fe860d8a89c1" />

Переходим в VPS, надо сделать конфиг, приватный ключ выводится при помощи команды cat, а публичный взялся из CHR:

<img width="1563" height="369" alt="image" src="https://github.com/user-attachments/assets/db74cc87-edf2-4996-a268-0706795f230b" />

Разрешение на форвардинг, открытие порта:

```
sudo sysctl -w net.ipv4.ip_forward=1
sudo ufw allow 51820/udp
```

Обновление и активация сервиса: 

<img width="1081" height="334" alt="image" src="https://github.com/user-attachments/assets/c33a0c4d-bbd0-4838-85cb-e8762016d01f" />

Пропингуемся, сначала в CHR:

<img width="828" height="189" alt="image" src="https://github.com/user-attachments/assets/606ac315-1bde-49c9-b61c-80c04465ec37" />

Теперь пропингуем CHR:

<img width="967" height="258" alt="image" src="https://github.com/user-attachments/assets/11d8da16-da4e-4873-bee2-3960849cc9fc" />

На этом всё.
