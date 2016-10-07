# 1. Создаем ноды для Docker'а

## Если VirtualBox

	docker-machine create --driver virtualbox swarm-00
	docker-machine create --driver virtualbox swarm-01
	docker-machine create --driver virtualbox swarm-02
	docker-machine create --driver virtualbox swarm-03

## Если DigitalOcean

	docker-machine create --driver digitalocean --digitalocean-access-token `cat ~/do-token` swarm-00
	docker-machine create --driver digitalocean --digitalocean-access-token `cat ~/do-token` swarm-01
	docker-machine create --driver digitalocean --digitalocean-access-token `cat ~/do-token` swarm-02
	docker-machine create --driver digitalocean --digitalocean-access-token `cat ~/do-token` swarm-03

# 2. Создаем менеджера кластера

Смотрим его IP:

	docker-machine inspect -f '{{ .Driver.IPAddress }}' swarm-00

Запускаем визуализатор (по желанию)

	docker docker $(docker-machine config swarm-00) \
		run -it -d -p 8080:8080 -e HOST=<ip> -v /var/run/docker.sock:/var/run/docker.sock manomarks/visualizer

Говорим Docker'у создать Swarm:

	docker $(docker-machine config swarm-00) \
		swarm init --listen-addr <ip>:2377 --advertise-addr <ip>:2377

Или так:

	eval $(docker-machine env swarm-00)
	docker swarm init --listen-addr <ip>:2377 --advertise-addr <ip>:2377

Или так:

	docker-machine ssh swarm-00 \
		docker swarm init --listen-addr <ip>:2377 --advertise-addr <ip>:2377

В ответ нам будет что-то вроде:

	Swarm initialized: current node (ebeayf6kd0paglg7c9jb6jq6d) is now a manager.

	To add a worker to this swarm, run the following command:
	    docker swarm join \
	    --token SWMTKN-1-6ax1kxfoivoaipytfjpbmiwjlkiigt2yau3e85pzv2icttekpa-8ipordkh0wx2dluuh5kw6xk86 \
	    192.168.99.100:2377

	To add a manager to this swarm, run the following command:
	    docker swarm join \
	    --token SWMTKN-1-6ax1kxfoivoaipytfjpbmiwjlkiigt2yau3e85pzv2icttekpa-av43wej4th4zzc7rwo3ukiuwq \
	    192.168.99.100:2377

# 3. Добавляем в кластер воркеров

Берём комманду, которую нам сказал swarm init и выполняем на каждой из нод:

	docker $(docker-machine config swarm-01) swarm join \
		--token SWMTKN-1-6ax1kxfoivoaipytfjpbmiwjlkiigt2yau3e85pzv2icttekpa-8ipordkh0wx2dluuh5kw6xk86 \
		192.168.99.100:2377

	docker $(docker-machine config swarm-02) swarm join \
		--token SWMTKN-1-6ax1kxfoivoaipytfjpbmiwjlkiigt2yau3e85pzv2icttekpa-8ipordkh0wx2dluuh5kw6xk86 \
		192.168.99.100:2377

	docker $(docker-machine config swarm-03) swarm join \
		--token SWMTKN-1-6ax1kxfoivoaipytfjpbmiwjlkiigt2yau3e85pzv2icttekpa-8ipordkh0wx2dluuh5kw6xk86 \
		192.168.99.100:2377

# 4. Теперь можно посмотреть статус кластера и нод

Выполняются команды на "менеджере"

Список нод в кластере:

	docker node ls

Состояние ноды (без `--pretty` будет json):

	docker node inspect --pretty swarm-01

Какие "задачи"/контейнеры запущены на ноде:

	docker node ps swarm-vm-01

# 5. Создание сервиса

Создаем простенький веб-сервис, который будет нам говорит хостнейм контейнера:

	docker service create --name demo --replicas 1 --publish 80:80 thshaw/helloworld

Посмотрим его состояние:

	docker service ls

Или ещё более подробно:

	docker service inspect demo

Посмотреть запущенные контейнеры этого сервиса:

	docker service ps demo

Проверим что сервис работает, сделаем запрос:

	curl 192.168.99.100

# 6. Масштабирование сервиса

Если мы хотим больше инстансов для этого сервиса:

	docker service scale demo=4

# 7. Отказоустойчивость сервиса

Для начала запустим "нагрузку" на наш сервис:

	go-wrk -d 50 http://192.168.99.100/

После чего "убьем" одну ноду:

	docker-machine stop swarm-02

Видим что все запросы обработаны и контейнеры перезапущены
Вернём ноду обратно:

	docker-machine start swarm-02

# 8. Обновление сервиса

Мы можем сделать rolling-update сервиса с заменой образа.
В данном случае мы меняем образ на вообще другой, в реальности это будет
просто новый тег с новой версией:

	docker service update --image instavote/vote:movies demo

