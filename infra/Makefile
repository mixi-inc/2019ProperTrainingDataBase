# 設定は ./infra/ec2.tf を参照
SERVER_IP=10.0.1.100

# terraformなどインフラ管理に必要なコマンドを実行するときに使う
.PHONY: infra-tools
infra-tools:
	mkdir -p ./tmp
	aws s3 cp s3://mixi-db-training/config/keys/id_db_training.pub ./tmp/
	aws s3 cp s3://mixi-db-training/config/keys/hello.public.gpg ./tmp/
	docker build -t infra-tools ../docker/infra-tools
	docker run \
		-ti \
		-v $(PWD):/app \
		-v $(HOME)/.aws:/root/.aws \
		-w /app \
		infra-tools \
		bash

~/.ssh/id_db_training:
	aws s3 cp s3://mixi-db-training/config/keys/id_db_training ~/.ssh/
	chmod go-rwx ~/.ssh/id_db_training

# サーバーにログインする
.PHONY: server/ssh
server/ssh: ~/.ssh/id_db_training
	ssh -i ~/.ssh/id_db_training ubuntu@$(SERVER_IP)

# サーバーのセットアップ
# Terraformの適用後に実行してください
.PHONY: server
server: ~/.ssh/id_db_training
	scp -i ~/.ssh/id_db_training ./Makefile ubuntu@$(SERVER_IP):
	scp -i ~/.ssh/id_db_training -r ../docker ubuntu@$(SERVER_IP):
	ssh -i ~/.ssh/id_db_training ubuntu@$(SERVER_IP) make server/docker
	ssh -i ~/.ssh/id_db_training ubuntu@$(SERVER_IP) make server/tutorial_db
	ssh -i ~/.ssh/id_db_training ubuntu@$(SERVER_IP) make server/benchapp

# サーバーにDockerをインストールする
.PHONY: server/docker
server/docker:
	sudo apt update && sudo apt install -y docker.io
	sudo usermod -a -G docker ubuntu
	sudo service docker restart

/swapfile:
	sudo dd if=/dev/zero of=/swapfile bs=128M count=10
	sudo chmod go-rwx /swapfile
	sudo mkswap /swapfile
	sudo swapon /swapfile

# チュートリアル: Jupyter Notebookからのクエリを実行する MySQL サーバー
.PHONY: server/tutorial_db
server/tutorial_db: /swapfile
	docker build -t tutorial_db ./docker/tutorial
	docker run \
		-d \
		-v $(PWD)/docker/tutorial/docker-entrypoint-initdb.d:/docker-entrypoint-initdb.d \
		-p 3306:3306 \
		tutorial_db

# インデックス演習: isucon5qのベンチマークを実行するためのWebアプリ
.PHONY: server/benchapp
server/benchapp:
	docker build -t bench ./docker/bench
	docker run \
		-d \
		--rm \
		-v $(PWD)/docker/bench/webapp:/opt/webapp \
		-w /opt/webapp \
		-p 8080:8080 \
		bench \
		bundle exec rackup --host 0.0.0.0 --port 8080