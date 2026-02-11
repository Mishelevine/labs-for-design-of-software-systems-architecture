# Лабораторная работа №4
### Левин Михаил РИС-22-1
**Тема:** Проектирование REST API
**Цель работы:** Получить опыт проектирования программного интерфейса.
## Описание API
Так как в проекте ВКР, по которому я выполнял предыдущие задания, вместо REST API используются WEBSOCKET, для данной лабораторной работы я выбрал другой проект.   
Проект представляет собой LBS (Location-based Service). Сервис позволяет определить местоположение пользователя по скану базовых станций (например, WiFi или GSM сигналы вокруг). Сервис работает при помощи встроенной модели машинного обучения, позволяет загружать данные в БД и в дальнейшем обучает на них систему определения  координат. Также сервис интегрирован с системой Yandex Locator который выполняет предсказивание местоположения если системе не хватает данных.

## Документация по API

### 1) POST `/api/v1/put_data` - отправка измерения для обучения
![alt text](assets/put_data.png)

**Описание:** метод предназначен для отправки скана базовых станций и местоположения в систему. В дальнейшем на этих данных обучается модель предсказания. Внутри метод валидирует ввод, проверяет все полученные станции на предмет нахождения в БД, для имеющихся выщитывает новые параметры, остальные просто заносит в базу. Сейчас поддерживает ввод Wifi и GSM сигналов.

**Headers:**
- `API-Key: <company_api_key>` - ключ, выдающийся каждой компании использующей систему, с его помощью в дальнейшем производится выставление счетов.
- `Content-Type: application/json`

**Body:**
```json
{
  "timestamp": "2026-01-28T10:15:21.653Z", - время отправки
  "lat": 0, - широта
  "lon": 0, - долгота
  "heading": 0, - направление в градусах
  "alt": 0, - высота над уровнем моря
  "signals": [
    {
      "station_type_id": 1, - тип станции
      "bssid": "string", - уникальный идентификатор станции
      "ssid": "string", - имя станции
      "rssi": 0 - сила сигнала
    }
  ],
  "device_imei": "string" - уникальный идентификатор устройства
}
```

**Response:**
```json
{
  "status": "ok",
  "measurement_id": 0 -  id нового измерения в базе
}
```

### 2) POST `/api/v1/get_data` - предсказание точки по скану
![alt text](assets/get_data.png)
**Описание:** получить местоположение полученное с помошью ML модели. Внутри запрос проверяет возможность получения данных по API ключу. Также если в системе недостаточно данных для предсказания, доступна возможность предсказания с помощью сервис YandexLocator.

**Headers:**
- `API-Key: <company_api_key>` - ключ, выдающийся каждой компании использующей систему, с его помощью в дальнейшем производится выставление счетов.
- `Content-Type: application/json`

**Body:**
```json
{
  "signals": [
    {
      "station_type_id": 1, - тип станции
      "bssid": "string", - уникальный идентификатор станции
      "rssi": 0, - сила сигнала
      "ssid": "string" - имя станции
    }
  ],
  "provider_mode": "local_only", - режим предсказания
  "device_imei": "string", - уникальный идентификатор устройства
  "map_match": false, - включать ли опцию привязки к карте
  "timestamp": "2026-01-28T10:52:16.641Z" - текущее время
}
```

Доступные режимы `provider_mode`:
- `local_only` - только локальное предсказание
- `yandex_only` - только предсказание Yandex Locator
- `auto_on_low_overlap` - если не получается локально, пробуем Yandex

**Response:**
```json
{
  "lat": 0, - широта
  "lon": 0, - долгота
  "accuracy": 0, - точность предсказания в метрах
  "source": "local", - источник полученного ответа
  "source_info": "string" - доп информация
}
```

### 3)POST `/api/v1/admin/keys/create` - создание ключа компании
![alt text](assets/keys_create.png)
**Описание:** позволяет администратору создать ключ доступа к API для компании. Установить для него лимиты запросов.

**Headers:**
- `X-Admin-Token: <admin_token>` - ключ администратора, нужен для использования всех admin запросов.
- `Content-Type: application/json`

**Body:**
```json
{
  "company_uid": "string", - уникальный идентификатор компании (не из БД)
  "name": "string", - название для нового ключа
  "req_max_yandex": 0, - максимально-допустимое кол-во запросов к Yandex
  "req_max_local": 0, - максимально-допустимое кол-во локальных запросов
}
```
При указании `req_max_yandex`/`req_max_local` = null устанавливается безлимит.   

**Response:**
```json
{
  "api_key": "string", - новый ключ
  "usage": { - установленные лимиты
    "key_name": "string",
    "yandex": { "count": 0, "max": 0, "remaining": 0 },
    "local":  { "count": 0, "max": 0, "remaining": 0 }
  }
}
```

### 4)GET `/api/v1/statistics/key_usage` - статистика использования ключа
![alt text](assets/statistics_usage.png)
**Описание:** позволяет получить статистику по использованию ключа.

**Headers:**
- `API-Key: <company_api_key>` - ключ, по которому требуется статистика.

**Body:** нет

**Response:**
```json
{
  "key_name": "string", - название ключа
  "yandex": { "count": 0, "max": 0, "remaining": 0 }, - лимит запросов Yandex
  "local":  { "count": 0, "max": 0, "remaining": 0 } - лимит локальных запросов
}
```

### 5)GET `/api/v1/statistics/company_devices` - количество устройств привязанных к компании
![alt text](assets/statistics_forein_devices.png)
**Описание:** позволяет получить сколько устройств уже подключенны к компании и сколько можно установить максимально. В данном случае учитываются только устройства от сторонних производителей.

**Headers:**
- `X-Admin-Token: <admin_token>` - ключ администратора.

**Query:**
- `company_uid` (string)

**Body:** нет

**Response:**
```json
{
  "company_uid": "string", - уникальный идентификатор компании
  "foreign_devices": 0, - количество привязанных устройств
  "device_limit": 0 - текущее ограничение на количество устройств
}
```
### 6)GET `/api/v1/admin/keys/list?company_uid=...` - список всех ключей компании
![alt text](assets/put_key.png)
**Описание:** позволяет получить информацию о всех ключах привязанных к конкретной компании

**Headers:**
- `X-Admin-Token: <admin_token>` - ключ администратора.
**Query:**
- `company_uid` (string)
**Body:** нет
**Response:**
```json
{
  "company_uid": "string", - уникальный идентификатор компании
  "keys": [
    {
      "key_name": "string", - название ключа
      "yandex": { "count": 0, "max": 0, "remaining": 0 },
      "local":  { "count": 0, "max": 0, "remaining": 0 }
    }
  ]
}
```

### 7)PUT `/api/v1/admin/keys/{company_uid}/{key_name}` - обновление лимитов ключа
![alt text](assets/put_key.png)
**Описание:** позволяет обновить текущие лимиты запросов для ключа, а также установить безлимит.

**Headers:**
- `X-Admin-Token: <admin_token>` - ключ администратора
- `Content-Type: application/json`

**Path params:**
- `company_uid` (string)
- `key_name` (string)

**Body:**
```json
{
  "req_max_yandex": 0,
  "req_max_local": 0
}
```
**Response:**
```json
{
  "key_name": "string",
  "yandex": { "count": 0, "max": 0, "remaining": 0 },
  "local":  { "count": 0, "max": 0, "remaining": 0 }
}
```

### 8)DELETE `/api/v1/admin/keys/{company_uid}/{key_name}` - удалить ключ API
![alt text](assets/delete_key.png)
**Описание:** позволяет удалить ключ из системы.

**Headers:**
- `X-Admin-Token: <admin_token>` - ключ администратора

**Path params:**
- `company_uid` (string)
- `key_name` (string)

**Body:**нет

**Response:**
```json
{
  "status": "ok"
}
```

### Интерактивная документация
Интерактивную документацию для API можно найти по [ссылке](https://lbs.fort-monitor.ru/docs).

![alt text](assets/docs.png)

## Тестирование API

### 1) POST `/api/v1/put_data`

**Body**:
```json
{
    "device_imei": "869132079252509",
    "timestamp": "2026-02-11T18:41:01",
    "lon": 39.664119466145834,
    "lat": 54.65038248697917,
    "heading": 166,
    "alt": 120,
    "signals": [
        {
            "station_type_id": 1,
            "bssid": "52FF20F18931",
            "ssid": "",
            "rssi": -83
        },
        {
            "station_type_id": 1,
            "bssid": "50FF20818931",
            "ssid": "",
            "rssi": -83
        },
        {
            "station_type_id": 1,
            "bssid": "C006C3B8D928",
            "ssid": "",
            "rssi": -84
        },
        {
            "station_type_id": 1,
            "bssid": "50FF20818881",
            "ssid": "",
            "rssi": -87
        },
        {
            "station_type_id": 1,
            "bssid": "E063DAD642DC",
            "ssid": "",
            "rssi": -88
        },
        {
            "station_type_id": 1,
            "bssid": "52FF20F18881",
            "ssid": "",
            "rssi": -88
        },
        {
            "station_type_id": 1,
            "bssid": "54AF972A0431",
            "ssid": "",
            "rssi": -90
        },
        {
            "station_type_id": 1,
            "bssid": "7E628B4A7BBF",
            "ssid": "",
            "rssi": -93
        },
        {
            "station_type_id": 1,
            "bssid": "5C628B4A7BBF",
            "ssid": "",
            "rssi": -94
        }
    ]
}
```

**Headers:**
![alt text](assets/postman/put_data/headers.png)

**Responce:**
![alt text](assets/postman/put_data/responce.png)

**Responce Headers:**
![alt text](assets/postman/put_data/responce_headers.png)

**Код автотестов и результат:**
![alt text](assets/postman/put_data/tests.png)

### 2) POST `/api/v1/get_data`

**Body:**
```json
{
    "device_imei": "868184067941461",
    "provider_mode": "local_only",
    "signals": [
        {
            "station_type_id": 1,
            "bssid": "245A4C0C68E4",
            "ssid": "",
            "rssi": -72
        },
        {
            "station_type_id": 1,
            "bssid": "E4FAC4F0E5CF",
            "ssid": "",
            "rssi": -81
        },
        {
            "station_type_id": 1,
            "bssid": "82AFCAA90363",
            "ssid": "",
            "rssi": -85
        },
        {
            "station_type_id": 1,
            "bssid": "80AFCA990363",
            "ssid": "",
            "rssi": -85
        },
        {
            "station_type_id": 1,
            "bssid": "B41C30A6831D",
            "ssid": "",
            "rssi": -87
        },
        {
            "station_type_id": 1,
            "bssid": "9017C8CC04FC",
            "ssid": "",
            "rssi": -89
        },
        {
            "station_type_id": 1,
            "bssid": "74C14FDDA03D",
            "ssid": "",
            "rssi": -90
        },
        {
            "station_type_id": 1,
            "bssid": "B0BE7680C9FA",
            "ssid": "",
            "rssi": -90
        },
        {
            "station_type_id": 1,
            "bssid": "74C14F614766",
            "ssid": "",
            "rssi": -92
        },
        {
            "station_type_id": 1,
            "bssid": "6083E78447C8",
            "ssid": "",
            "rssi": -92
        },
        {
            "station_type_id": 1,
            "bssid": "3460F9E66B62",
            "ssid": "",
            "rssi": -93
        },
        {
            "station_type_id": 1,
            "bssid": "9C9D7EC12E4E",
            "ssid": "",
            "rssi": -94
        },
        {
            "station_type_id": 1,
            "bssid": "304596DD4371",
            "ssid": "",
            "rssi": -95
        }
    ],
    "timestamp": "2026-02-11T03:12:46",
    "map_match": false
}
```

**Headers:**
![alt text](assets/postman/get_data/headers.png)

**Responce:**
![alt text](assets/postman/get_data/response.png)

**Responce Headers:**
![alt text](assets/postman/get_data/response_header.png)

**Код автотестов и результат:**
![alt text](assets/postman/get_data/tests.png)

### 3)POST `/api/v1/admin/keys/create`
**Body:**
```json
{
  "company_uid": "string",
  "name": "new_key",
  "req_max_yandex": 1000,
  "req_max_local": 1000
}
```

**Headers:**
![alt text](assets/postman/create_key/headers.png)

**Responce:**
![alt text](assets/postman/create_key/response.png)

**Responce Headers:**
![alt text](assets/postman/create_key/response_header.png)

**Код автотестов и результат:**
![alt text](assets/postman/create_key/tests.png)

### 4)GET `/api/v1/statistics/key_usage`
**Body:**нет

**Headers and Responce:**
![alt text](assets/postman/key_usage/headers_and_response.png)

**Responce Headers:**
![alt text](assets/postman/key_usage/response_header.png)

**Код автотестов и результат:**
![alt text](assets/postman/key_usage/tests.png)

### 5)GET `/api/v1/statistics/company_devices`
**Params:**
![alt text](assets/postman/company_devices/params.png)

**Headers and Responce:**
![alt text](assets/postman/company_devices/headers_and_response.png)

**Responce Headers:**
![alt text](assets/postman/company_devices/response_header.png)

**Код автотестов и результат:**
![alt text](assets/postman/company_devices/tests.png)

### 6)GET `/api/v1/admin/keys/list?company_uid=...`

**Params:**
![alt text](assets/postman/keys_list/params.png)

**Headers:**
![alt text](assets/postman/keys_list/headers.png)

**Responce:**
![alt text](assets/postman/keys_list/response.png)

**Responce Headers:**
![alt text](assets/postman/keys_list/respose_header.png)

**Код автотестов и результат:**
![alt text](assets/postman/keys_list/tests.png)

### 7)PUT `/api/v1/admin/keys/{company_uid}/{key_name}`
**Body:**
```json
{
    "req_max_yandex": 500,
    "req_max_local": 500
}
```

**Path Variables and Response:**
![alt text](assets/postman/put_keys/params_response.png)

**Headers and Response Headers:**
![alt text](assets/postman/put_keys/headers.png)

**Код автотестов и результат:**
![alt text](assets/postman/put_keys/tests.png)

### 8)DELETE `/api/v1/admin/keys/{company_uid}/{key_name}`
**Body:**
```json
{
    "req_max_yandex": 500,
    "req_max_local": 500
}
```

**Path Variables and Response:**
![alt text](assets/postman/delete_key/params_response.png)

**Headers and Response Headers:**
![alt text](assets/postman/delete_key/headers.png)

**Код автотестов и результат:**
![alt text](assets/postman/delete_key/tests.png)
