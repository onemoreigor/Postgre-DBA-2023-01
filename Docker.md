# Postgre-DBA-2023-01

Поставить на нем Docker Engine

```sh
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
Сделать каталог /var/lib/postgres
```sh
sudo mkdir /var/lib/postgres
```
Развернуть контейнер с PostgreSQL 14 смонтировав в него /var/lib/postgres
```sh
sudo docker run --name pg-srv --network pg-net -p 5432:5432 -e POSTGRES_PASSWORD=Password -v /var/lib/postgres:/var/lib/postgresql/data -d postgres:14
```
Развернуть контейнер с клиентом postgres
```sh
sudo docker run -it --rm --name pg-cli --network pg-net  postgres:14 psql -h pg-srv -U postgres
```
Подключиться из контейнера с клиентом к контейнеру с сервером и сделать
таблицу с парой строк
```sh
Использовал пример из предыдущего ДЗ
```
Подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP/ЯО/места установки докера
```sh
PS C:\Program Files\PostgreSQL\14\bin> .\psql.exe -h VM-IP-address -p 5432 -U postgres
VM-IP-address - айпи адрес докер-хоста (ВМ)
```
Удалить контейнер с сервером
```sh
sudo docker stop pg-srv
sudo docker rm pg-srv
```
- создать его заново
- подключиться снова из контейнера с клиентом к контейнеру с сервером
- проверить, что данные остались на месте
```sh
Данные на месте
```

Проблемы, с которыми столкнулся :

- Не создал сеть. Из-за чего не мог создать временный контейнер для подключения к серверу БД. Была проблема с разрешением имени контейнера с БД.