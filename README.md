### Материал представлен исключительно в целях ознакомления с принципом работы технологии Vless Reality и не предназначен для обхода блокировок. Эта технология может понадобиться вам например для того, чтобы ознакомиться с ресурсами в сети интернет в государстве, которое запрещает подключения к сайтам своих доменов извне, например к Дубайским сервисам, если вы там ведете бизнес удаленно. 

___
### Схема подключения

![connection scheme](https://github.com/malinborn/xray-xtls-reality-ru-config/blob/main/media/scheme.png)

___
### Общая информация
В первую очередь, нам нужно сделать несколько прокси, как минимум:
- **domestic_proxy** — это прокси вашего домашнего региона. В моем примере это будет Россия. 
	- К доместик прокси будут подключаться клиенты; 
	- Доместик прокси — основа маршрутизации в моей конфигурации. 
- **foreign_proxy** — прокси региона, в который вам нужно попасть. В моем примере, это Германия. 
	- **foreign_proxy** может быть несколько, на каждую необходимо купить VPS необходимого вам региона. Это может быть актуально для Дубая, Ирана или Китая. 
___
### Подготавливаем активы
Для создания подходящей прокси нужно приобрести VPS следующей конфигурации:
- 1 CPU
- 1 Gb RAM 
- \>=5 Gb SSD/HDD *(для логов и программ)*
- Ubuntu \>= 20.\*

Я приобрел европейскую VPS на https://my.aeza.net/ и российскую на https://timeweb.cloud/. На мой взгляд, на этих сервисах неплохое соотношение цена-качество, плюс есть оплата по СБП, что очень удобно. 
___
### Настраиваем виртуальные машины
Теперь необходимо настроить наши VPS, настройка каждой из них ничем не отличается, разница будет только в конфиге. 
Я не уверен кто именно будет читать эту инструкцию, поэтому напишу более менее подробный гайд по настройке VPS на Ubuntu под VPN: 
1. **Подключись к VPS по SSH**
	```bash
   ssh root@your.vps.ip.address
	```
1. **Смени пароль на надежный**
   После введения этой команды, система предложит ввести новый пароль:
	```bash
   passwd
	```

3. **Обнови систему и основные утилиты**
   ```bash
   apt update && apt upgrade
	```
   
4. **Добавь SSH-ключи**
	1. Если у тебя еще нет пары SSH-ключей на своем компьютере — сгенерируй их:
		```bash
		ssh-keygen -t ed25519
		```
	2. Отправь ключи на виртуальную машину:
	   ```bash
		ssh-copy-id user@your.vps.ip.address
		```
5. **Установи удобный текстовый редактор micro *(не обязательно, но это все упростит)***
   ```bash
   apt install micro
	```
6. **Отключи вход по паролю**
	1. Поправь конфиг сервера SSH VPS
	   ```bash
	   micro /etc/ssh/sshd_config
		```
	2. Тут нужно найти опцию `PasswordAuthentication`, убрать перед ней `#` если он там есть и написать `no` вместо `yes` напротив
	3. Проверь есть ли файлы в sshd_confid.d
	   ```bash
	   ls -la /etc/ssh/sshd_config.d
		```
	4. Если есть — открой файл в редакторе текста и если там есть опция `PasswordAuthentication`, сделай с ней все то же, что и в пункте 2. Таким образом, ты отключишь пароль, но все еще сможешь логиниться по SSH-ключам, которые мы сделали и сохранили ранее. 
	   ```bash
	   micro /etc/ssh/sshd_config.d/{file_name}
		```
        5. Перезагрузи виртуалку или демон ssh в systemctl. 
	   ```bash
	   sudo systemctl restart ssh
		```
	5. Взломать SSH-ключи перебором практически невозможно, но их все еще можно украсть с твоего компьютера, поэтому не забывай соблюдать правила информационной гигиены! 
7. **Поздравляю, твоя Ubuntu VPS настроена!** 🎉
   С базовыми приготовлениями закончили, теперь идем настраивать VPN! 
___
### Настраиваем VPN
*Настройка VPN на всех VPS будет одинаковая, разве что нужно будет положить туда разные конфиги.*
1. Устанавливаем Xray на машину:
   ```bash
	   bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install --beta -u root
	```
2. Получаем xray uuid для этой машины, его нужно куда-нибудь записать, дальше он потребуется нам в конфиге:
   ```bash
    /usr/local/bin/xray uuid
	```
3. Получаем ключи xray для этой машины, приватный и публичный, их тоже оба обязательно нужно где-нибудь сохранить:
   ```bash
    /usr/local/bin/xray x25519
	```
##### Дальше развилка:
##### Foreign Proxy VPS:
> Начать лучше с конфигурации Foreign Proxy

Выполни команду 
```bash
micro /usr/local/etc/xray/config.json
```
и приведи файл к следующему виду:
> Там, где значения обернуты в символы `<>`, нужно подставить свои значения. 

```json
{
	"log": {
		"loglevel": "info"
	},
	"routing": {
		"domainStrategy": "IPIfNonMatch",
		"rules": [
			{
				"type": "field",
				"domainMatcher": "linear",
				"domain": ["full:whatismyipaddress.com"], // just example
				"outboundTag": "block"
			}
		]
	},
	"inbounds": [
		{
			"listen": "0.0.0.0",
			"port": 443,
			"protocol": "vless",
			"tag": "vless_tls",
			"settings": {
				"clients": [
					{
						"id": "<foreign_proxy_xray_uuid>",
						"email": "<your_email>",
						"flow": "xtls-rprx-vision"
					}
				],
				"decryption": "none"
			},
			"streamSettings": {
				"network": "tcp",
				"security": "reality",
				"realitySettings": {
					"show": false,
					"dest": "www.google.com:443",
					"xver": 0,
					"serverNames": [
						"www.google.com"
					],
					"privateKey": "<foreign_proxy_xray_private_key>",
					"minClientVer": "",
					"maxClientVer": "",
					"maxTimeDiff": 0, 
					"shortIds": [
						"<any_8_HEX_digits>"
					]
				}
			},
			"sniffing": {
				"enabled": true,
				"destOverride": [
					"http",
					"tls"
				]
			}
		}
	],
	"outbounds": [
        {
            "protocol": "freedom",
            "tag": "direct"
        },
        {
            "protocol": "blackhole",
            "tag": "block"
        }
    ]
}

```
##### Domestic Proxy VPS:
Выполни команду 
```bash
micro /usr/local/etc/xray/config.json
```
и приведи файл к следующему виду:
> Там, где значения обернуты в символы `<>`, нужно подставить свои значения. 
```json
{
  "log": {
    "loglevel": "info"
  },
  "routing": {
    "domainStrategy": "AsIs",
    "rules": [
      {
        "type": "field",
        "domain": [
          "geosite:category-ads-all"
        ],
        "outboundTag": "block"
      },
      {
        "type": "field",
        "domain": [
          "regexp:.*\\.ru$"
        ],
        "outboundTag": "direct"
      },
      {
        "type": "field",
        "ip": [
          "geoip:ru"
        ],
        "outboundTag": "direct"
      },
      {
      	"type": "field",
      	"domain": [
      	    "youtube.com", 
      	    "youtu.be",
      	    "googlevideo.com"
      	],
      	"outboundTag": "direct"
      },
      {
      	"type": "field",
      	"domain": [
      		"google.com"
      	],
      	"outboundTag": "direct"
      },
      {
      	"type": "field",
      	"domain": [
      		"whatsmyip.org"
      	],
      	"outboundTag": "direct"
      }
    ]
  },
  "inbounds": [
    {
     	"listen": "0.0.0.0",
      "port": 443,
      "protocol": "vless",
      "tag": "vless_tls",
      "settings": {
        "clients": [
          {
            "id": "<domestic_proxy_xray_uuid>", // cmd xray uuid
            "email": "<your_email>", // optional
            "flow": "xtls-rprx-vision"
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "www.google.com:443",
          "xver": 0,
          "serverNames": [
            "www.google.com"
          ],
          "privateKey": "<domestic_proxy_xray_private_key>", 
          "minClientVer": "",
          "maxClientVer": "",
          "maxTimeDiff": 0,
          "shortIds": [
            "<any_8_HEX_digits>"
          ]
        }
      },
      "sniffing": {
        "enabled": true,
        "destOverride": [
          "http",
          "tls"
        ]
      }
    }
  ],
  "outbounds": [
    {
      "tag": "proxy",
      "protocol": "vless",
      "settings": {
        "vnext": [
          {
            "address": "<foreign_proxy_ip>", // foreign VPS IP
            "port": 443,
            "users": [
              {
                "id": "<foreign_proxy_uuid>", // xray uuid
                "encryption": "none",
                "flow": "xtls-rprx-vision"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "fingerprint": "chrome",
          "serverName": "www.google.com",
          "publicKey": "<foreign_proxy_public_key>",
          "shortId": "<foreign_proxy_short_id>",
          "spiderX": ""
        }
      }
    },
    {
      "protocol": "freedom",
      "tag": "direct"
    },
    {
      "protocol": "blackhole",
      "tag": "block"
    }
  ]
}
```
___
### Запускаем VPN
Пришли к самой простой части, нужно просто запустить Xray:
```bash
systemctl start xray
```
Команда для удобного отслеживания логов:
```bash
journalctl -u xray -n 100  // 100 последних записей
```
```bash
journalctl -u xray  // записи за все время
```
После обновления конфига, xray нужно перезапустить:
```bash
systemctl restart xray
```
___
### Для продвинутых, кастомный роутинг
Есть условно три варианта outbound-подключений, на которые будет отправляться трафик:
- block — название говорит само за себя 
- direct — отправка обычного трафика с VPS
- proxy — отправка трафика на прокси. Domestik Proxy таким образом передает трафик на Foreign Proxy. 

Примеры настройки правил роутинга есть в конфиге, дополнительная информация по настройке правил есть по ссылке:
https://xtls.github.io/config/routing.html#routingobject
___
### Приложения-клиенты для подключения VPN
##### Не так страшна их настройка как кажется, ведь это нужно сделать всего раз, включить VPN всего раз и забыть об этом чудесном путешествии. 
- **Macos, IOS**:
	- FoXray, его можно скачать в AppStore. Настройка подключения интуитивно понятна, тк вы эти значения уже подставляли в конфиг, они даже называются так же. 
- **Windows, linux**: 
	- Nekobox — https://github.com/MatsuriDayo/nekoray/releases
	- Переключи программу на использование движка sing-box в `Basic Settings` -> `Core`
	- Зайди в `Server` -> `New profile` и заполни по аналогии с конфигом.
- **Android**: 
	- HiddifyNG — https://github.com/hiddify/HiddifyNG
