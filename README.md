![example workflow](https://github.com/dwx82/rubysocial/actions/workflows/magic.yaml/badge.svg)
![example workflow](https://github.com/dwx82/rubysocial/actions/workflows/prodrelease.yml/badge.svg)
![example workflow](https://github.com/dwx82/rubysocial/actions/workflows/prodmagic.yaml/badge.svg)
![example workflow](https://github.com/dwx82/rubysocial/actions/workflows/build.yml/badge.svg)
```
Полный CI/CD используя GiHubActions, GCP, k8s, Terraform, DockerHub.
Мониторинг - Prometheus + Graphana.
Балансировщик - Nginx Ingress.
БД - Google cloud SQL/ Postgress.
Code Quality and Code Security - SonarQube

Проект основан на:
https://github.com/overeng/rubysocial

В описании будет достаточно много ссылок на другие ресурсы, и если вы (как и я когда это делал) не докнца понимаете , что делаете
и что получится - настоятельно рекомендую по ним пройтись, сэкономите время и не вылезут ошибки от которых впадаете в ступор.
А те которые вылезут сразу поймете, что сам дурак. 

Если не знаете как клонировать репозиторий и вообще с чего начать:
https://www.youtube.com/c/ADVIT4000/playlists

#=======================================================================================================================

Все действия выполнялись на:
MBP / Intel/ MacOS Monterey

#=======================================================================================================================

Для продолжения вам потребуется brew:
https://www.google.com/search?q=brew+install&rlz=1C5CHFA_enUA956UA956&oq=brew+insta&aqs=chrome.0.0i512j69i57j0i67j0i20i263i512j0i512l6.10866j0j7&sourceid=chrome&ie=UTF-8

Установим необходимые для создания кластеры инструменты: 
brew install kubectl
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
brew install kubernetes-helm

Установка GCP CLI:
https://cloud.google.com/sdk/docs/install-sdk

Регистрируемся в GCP.
Потому что при регистрации даёт $300 и их вполне хватит на этот проект.

Через web intetface создаём ведро для tf-state и проект.  # бестпрактис фо зе тимворк
Ведро обязательно с версионингом !!!

Далее необходмо выполнить команды (устанавливаем плагин, выбираем наш проект и получаем кредентиалс):

gcloud components install gke-gcloud-auth-plugin
gcloud auth application-default login
gcloud projects list
gcloud config set project youprojectid
gcloud auth application-default login --project staging-357319
gcloud container clusters get-credentials primary -z=us-central1-a
#gcloud container clusters get-credentials primary --zone us-central1-a --project staging-357319

kubectl config current-context

Перед описанием состояния кластера в терраформ, сначала проделал все действия вручную.
И вам осветую - для лучшего понимания происходящего.

Сначала зададим пароль и имя пользователя для БД.

Set values with environment variables
When Terraform runs, it looks in your environment for variables that match the pattern TF_VAR_<VARIABLE_NAME>,
and assigns those values to the corresponding Terraform variables in your configuration.

Assign values to the database administrator username and password using environment variables.

export TF_VAR_db_username=admin TF_VAR_db_password=adifferentpassword

если не задавать, установится пароль из файла с переменными.

#============================================Terraform===============================================================
https://github.com/k-mitevski/terraform-gke
https://blog.knoldus.com/how-to-create-gke-on-gcp-using-terraform/
https://learnk8s.io/terraform-gke
https://antonputra.com/google/create-gke-cluster-using-terraform/#create-cloud-nat-in-gcp-using-terraform
https://kubernetes.io/docs/tasks/tools/
https://github.com/gavinbunney/terraform-provider-kubectl/issues/26
https://github.com/gavinbunney/terraform-provider-kubectl/issues/61

Код для терраформа лежит:
https://github.com/dwx82/TF-for-rubysocial

Из папки Terraform необходимо выполнить следующие комманды:

terraform init
terraform apply

Терраформ здесь описывать не буду, там и так комментов больше чем кода. 
Опишу назначение каждого файла:
0   - переменные, это один из двух файлов который вам необходимо отредактировать под себя. Второй - tlsingress.yaml
1   - Провайдеры терраформ (плагины для работы с внешними API)
2   - Создаём свою сеть (VPC)
3   - Подсети
4,5 - маршрут в интернет через нат.
6   - установка ингресс контроллера/сервиса/правил, letsencrypt (для https), стека прометеус-графана.
7,8 - создание GKE
9   - создание БД, с расписанием бэкапов и пользователя БД.  
10  - создаём ведро и копируем в него статичные файлы сайта. И не просто копируем, а сначала проверяем изменился ли файл.
В результате копирование терраформом не использовал. Но модуль решил не удалять, может кому пригодится.

Ждем примерно 40 минут и инфраструктура готова.

В принципе файлы 2,3,4,5 лучше объеденить в один, но я оставил так для лучшего понимания происходящего.

!!!!!!!! ПОСЛЕ УДАЛЕНИЯ ИНФРАСТРУКТУРЫ ЧЕРЕЗ terraform destroy БД НЕ УДАЛИТСЯ !!!!!!!!
!!!!!!!! УДАЛЯЕМ ЧЕРЕЗ web. Это очень недешево забыть удалить инстанс с БД    !!!!!!!! 

Через примерно 40 минут будет готова наша инфраструктура. 

После большого кол-ва создания удалений инфраструктуры возможны различные глюки.
У меня в какой-то момент престало создаваться правило для ингресс контроллера, решение:
https://stackoverflow.com/questions/61616203/nginx-ingress-controller-failed-calling-webhook

kubectl get svc

в ответ вы должны получить список сервисов в namespace по умолчанию.
результат:
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.52.0.1    <none>        443/TCP   48m

#====================================================Prometheus=========================================================
https://www.youtube.com/watch?v=dk2-_DbWb80
https://inostudio.com/blog/articles-devops/nastroyka-kube-prometheus-stack/
https://stackoverflow.com/questions/61121046/ingress-routing-rules-to-access-prometheus-server

Архитектура Prometheus
Разница между prometheus и prometheus-operator в том, что в prometheus-operator дополнительно включена Grafana с готовыми дашбордами 
и ServiceMonitors для сбора метрик из сервисов кластера таких как: CoreDNS, API Server, Scheduler и другие.

Prometheus — главная составляющая всей конструкции. Система мониторинга, которая собирает показатели из ваших служб и сохраняет их в базе данных временных рядов.
Если кратко, то она собирает метрики, на которые можно посмотреть при помощи Prometheus Web.

Grafana — веб-приложение для аналитики и интерактивной визуализации

Alertmanager — инструмент для обработки оповещений. Он устраняет дубликаты, группирует и отправляет нотификации.

Экспортёры (агенты) — их задача собирать метрики из сервисов и переводить их в понятный вид для Prometheus. 
Видов экспортёров существует огромное количество например: mysql_exporter, memcached_exporter, jms_exporter и так далее.

В стеке используются экспортёры:
Node-exporter — собирает метрики о вычислительных ресурсах Node в Kubernetes.
Kube-state-metrics — собирает метрики со всех узлов Kubernetes, обслуживаемых Kubelet через Summary API. Создатели экспортёра были вдохновлены проектом Heapster.

Я устанавливаю стек мониторинга используя терраформ. Что аналогично вводу следующей последовательности команд:

helm repo add prometheus-community \
https://prometheus-community.github.io/helm-charts
helm repo add stable https://charts.helm.sh/stable

helm repo update && \
helm install monitoring prometheus-community/kube-prometheus-stack \
--namespace monitoring \
--create-namespace

После утсановки можно проверить результат:

kubectl get po -n monitoring

NAME                                                     READY   STATUS    RESTARTS   AGE
alertmanager-monitoring-kube-prometheus-alertmanager-0   2/2     Running   0          41s
monitoring-grafana-7556fbb79-lgt4v                       3/3     Running   0          47s
monitoring-kube-prometheus-operator-9c7595c4-mhn7v       1/1     Running   0          47s
monitoring-kube-state-metrics-7878dbb459-cnqgs           1/1     Running   0          47s
monitoring-prometheus-node-exporter-6vs85                1/1     Running   0          48s
monitoring-prometheus-node-exporter-gq9tt                1/1     Running   0          48s
prometheus-monitoring-kube-prometheus-prometheus-0       2/2     Running   0          41s

kubectl port-forward monitoring-grafana-7556fbb79-lgt4v 3000 -n monitoring

Теперь можем проверить, что у нас всё получилось:
Стандартный логин пароль — admin / prom-operator (войдя в веб-интерфейс, рекомендую сменить пароль), url —  localhost:3000

#====================================================Ingress============================================================
https://cloud.google.com/community/tutorials/nginx-ingress-gke
https://www.youtube.com/watch?v=f3TfVN5-BLQ
https://www.youtube.com/watch?v=9sLHoEyRq8w
https://kubernetes.github.io/ingress-nginx/user-guide/monitoring/
https://devopscube.com/configure-ingress-tls-kubernetes/
https://www.youtube.com/watch?v=D87z6v9xqW8
https://cloud.yandex.ru/docs/managed-kubernetes/tutorials/ingress-cert-manager
https://nickjanetakis.com/blog/configuring-a-kind-cluster-with-nginx-ingress-using-terraform-and-helm

Установка NGINX Ingress-контроллера с менеджером для сертификатов Let's Encrypt®

Для начала необходимо зарегистрировать домен. Я использовал бесплатный от регистратора nic.ua.
У него интуитивно понятный интрфейс - регистрация не составит проблем.

Ingress controller также утсанавливается терраформом, установка руками описана ниже:

В любом случае отредактируйте tlsingress.yaml
Поменяйте почту и выберите необходимый сертификат - prod/stage.
Prod - нельзя выпустить более 5 сертификатов в неделю. Ну и подписан он как следует.

Установка nginx Ingress controller используя helm:

helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update && \
helm install my-ingress nginx-stable/nginx-ingress \
--namespace ingress \
--create-namespace \
--values nginx-values.yaml

Проверяем:

kubectl get pods -n ingress 

NAME                                        READY   STATUS    RESTARTS   AGE
my-ingress-nginx-ingress-6dfb56b6b7-ncxsb   1/1     Running   0          8m21s

kubectl get svc -n ingress

NAME                       TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)                      AGE
my-ingress-nginx-ingress   LoadBalancer   10.52.6.163   34.68.42.250   80:31431/TCP,443:30027/TCP   8m59s

kubectl get ingressclass

NAME    CONTROLLER                     PARAMETERS   AGE
nginx   nginx.org/ingress-controller   <none>       9m29s

Теперь можем создать поддомены и добавить A-запись, указывающую на публичный IP-адрес Ingress-контроллера
в личном кабинете nic.ua.  

Установите менеджер сертификатов версии 1.9.0, настроенный для выпуска сертификатов от Let's Encrypt®
(проверьте наличие более новой версии на странице проекта):
https://github.com/cert-manager/cert-manager/releases

kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.9.0/cert-manager.yaml

В терраформе тоже ставил через манифест. Пробовал разные чарты, без нюансов работает только через манифест.

Убедитесь, что в пространстве имен cert-manager создано три пода с готовностью 1/1 и статусом Running:
kubectl get pods -n cert-manager

NAME                                      READY   STATUS    RESTARTS   AGE
cert-manager-55658cdf68-gzpfn             1/1     Running   0          11h
cert-manager-cainjector-967788869-84xsc   1/1     Running   0          11h
cert-manager-webhook-6668fbb57d-w8z56     1/1     Running   0          11h

создаст необходимые правила и установит менеджер сертификатов:
kubectl apply -f tlsingress.yaml

Если успею - запихну ингресс в терраформ:
https://www.patricia-anong.com/blog/2020/6/01/deploy-nginx-ingress-and-cert-manager-on-a-gke-cluster-using-terraform
https://registry.terraform.io/providers/hashicorp/helm/latest/docs
https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/resources/ingress
успел)

#===================================================ELK=================================================================

https://devopscounsel.com/elasticsearch-kibana-setup-on-kubernetes-cluster/
https://www.youtube.com/watch?v=TpMxGgAdDxU


Почему не елк:
http://iam-saminda.blogspot.com/2020/02/elk-vs-amazon-cloudwatch-vs-google.html

Для логов будем использовать облачного провайдера.

#====================================================db===============================================================

Базу данных тоже создаём в терраформе. Там же настраиваем бэкапы.

https://devapo.io/blog/terraform-how-to-create-an-infastructure-part-2/
https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/sql_database_instance
https://cloud.google.com/sql/docs/mysql/instance-settings
https://getbetterdevops.io/terraform-with-helm/

#=======================================================================================================================

Automatic upload to Google Cloud Storage with GitHub Actions
https://www.youtube.com/watch?v=vlySg5UPIm4
https://www.youtube.com/watch?v=6dLHcnlPi_U

https://sha.ws/automatic-upload-to-google-cloud-storage-with-github-actions.html

#=====================================================GitHub============================================================
https://guides.rubyonrails.org/v6.1/getting_started.html
https://docs.github.com/en/actions/using-workflows#adding-a-workflow-status-badge-to-your-repository

После того как склонировали репозиторий, перейдем в папку репозитория и установим зависимости 

bundle install

в зависимости от типа и версии операционной системы могут вылезти ошибки, которые лечатся установкой недостающих пакетов.

Необходимо установить следующие пакеты

Устанавливаем менеджер версий руби:
\curl -sSL https://get.rvm.io | bash -s stableruby-install ruby

Устанавливаем руби 2.7.1:
rvm install 2.7.1 

Переключаемся на нужную версию:
rvm use 2.7.1

устанавливаем rails и yarn:
gem install rails
gem install yarn

устанавливаем node.js:
https://nodejs.org/uk/download/

установим postgresql:
brew install postgresql

Теперь можем выполнить 

bundle install

и установятся все необходмые зависимости.

Чтобы сгенерировать master.key выполним
EDITOR="code --wait" rails credentials:edit

В результате создастся master.key (незашифрован - в репозиторей не добавлять) и
secret_key_base (зашифрован) который используется для шифрования куки.

Сохраняем значение secret_key_base, потом пригодится.

Теперь можем создать контейнер используя ключ из master.key
(естественно у вас должен быть установлен докер, если нет brew install --cask docker
затем запускаем приложение из Launchpad и соглашаемся с лицензией)

Проверяем наш докер файл, в качестве аргумента - наш master.key

docker build -t rubysocial . --build-arg RAILS_KEY=0766da949418945e5011bffddcbe1667

Регистрируемся на докер хабе и создаём там репозиторий rubysocial. Также создаем токен - чтобы гитхаб могу пушить в докерхаб.
Сохраняем токен и логин докерхаба как секреты в гитхабе - DOCKER_HUB_TOKEN и DOCKER_HUB_LOGIN

Для гитхаба генерируем отдельный production.key и сохраняем как секрет гитхаба - RAILS_KEY:

EDITOR="code --wait" rails credentials:edit --environment production

В открывшийся production.yml вставляем:

secret_key_base: d7cb426ab5e574ed77bd2897a15bdf882d7698d8a222880709c09b09e3946734b728a83ea895b8394fccab5d0abda478f6d316826214d2631a59871d0179fdc2

сохраняем и закрываем.

Добавляем все измененния в наш репозиторий:

git add .
git commit -m "production.key"
git push origin1 master

Проверяем:

git tag -a v0.1 -m "v0.1"
git push origin1 v0.1

Переходим в гитхаб и смотрим запустился ли наш вокфлоу.

#=====================================================DB================================================================

Теперь создадим нашу бд:
DATABASE_HOST - адрес БД созданной в терраформ.
DATABASE_USERNAME - переменная из терраформ vars
DATABASE_PASSWORD - переменная из терраформ vars

RAILS_ENV=production DATABASE_HOST=db_instance_ip DATABASE_USERNAME=user DATABASE_PASSWORD=password rails db:create

Мигрируем данные в неё:

RAILS_ENV=production DATABASE_HOST=db_instance_ip DATABASE_USERNAME=user DATABASE_PASSWORD=password rails db:migrate

Создадим секреты для доступа к бд чтобы не пердавать их в открытом виде:
kubectl create secret generic <YOUR-DB-SECRET> \ #lowercase
  --from-literal=username=<YOUR-DATABASE-USER> \
  --from-literal=password=<YOUR-DATABASE-PASSWORD> \
  --from-literal=database=<YOUR-DATABASE-NAME> \
  --from-literal=ip=<YOUR-DATABASE-IP>

#=======================================================================================================================

Создадим секрет с rails-master-key указываем значение production.key

kubectl create secret generic rails-master-key --from-literal='rails-master-key=value'

Создаём в GCS роль для доступа к ведру
https://sha.ws/automatic-upload-to-google-cloud-storage-with-github-actions.html

Navigate to: IAM & Admin > Roles > +Create Role
Enter a title for the role - I like to preface custom roles with Custom - so they're easy to list later on
(Optional) Enter a description
Set Role launch stage to General Availability
Assign permissions:

resourcemanager.projects.get
storage.buckets.get
storage.buckets.list
storage.objects.create
storage.objects.delete
storage.objects.get
storage.objects.list
storage.objects.update
See the GCP documentation for Creating and managing custom roles for more details.

Create a new service account
Navigate to: IAM & Admin > Service Accounts > +Create Service Account
Enter a name for the account - I like to preface service accounts with sa -
Accept the recommended service account id
(Optional) Enter a description
Click Create
On the second page, select the custom role created in the previous step - this will assign permissions to the account
Select all other defaults to finish creating the account


gcloud iam service-accounts keys create ~/key.json \
  --iam-account bucket-gha@staging-357319.iam.gserviceaccount.com

 cat ~/key.json |base64
 
#================================================GitHub=================================================================
Создаём секреты в гитхабе:

GCS_PROJECT
The name of the CGP project containing the Cloud Storage Bucket.

GCS_BUCKET
The name of the Cloud Storage bucket to deploy the site.

GCS_SA_KEY
The base64 encoded authentication key for the service account.

Base64 encode the key:

cat ~/key.json |base64

Всё работает. Продолжаем автоматизировать.

Сделаем из нашего rails.yaml шаблон где переменной будет версия релиза - таг который мы пушим.

helm create rubysocial

копируем в папку tepmlate  rails.yaml и меняем теги на {{ .Values.version }}
Если будут ошибки - считайте пробелы, это ямл )

Теперь создадим секрет PROD_CONFIG_DATA - в нём все данные нашего GCP

kubectl config current-context

kubectl config view --minify --flatten --context="output kubectl config current-context"

Из вывода удалить строчку с cmd-path:  !!!!

#================================================SonarQube==============================================================

helm repo add sonarqube https://SonarSource.github.io/helm-chart-sonarqube
helm repo update
kubectl create namespace sonarqube
helm upgrade --install -n sonarqube sonarqube sonarqube/sonarqube

1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace sonarqube -l "app=sonarqube,release=sonarqube" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl port-forward $POD_NAME 8080:9000 -n sonarqube

 export SONARQUBE_PASSWORD=$(kubectl get secret --namespace "sonarqube" sonar-sonarqube -o jsonpath="{.data.sonarqube-password}" | base64 -d) 

admin/admin
или 
user и пароль export SONARQUBE_PASSWORD=$(kubectl get secret --namespace "sonarqube" sonar-sonarqube -o jsonpath="{.data.sonarqube-password}" | base64 -d)
зависит от того каким чартом будете устанавливать.

Сайт должен быть с prod letsecrypt, или с любым другим нормально подписаным сертификатом, иначе не сможете подключиться из гитхаба.

Получение кода в сонаре настраивается через гитхаб экшенс, там прям очень легко. В сонаре запустить мастер и следовать инструкциям.

Из терраформа сонар через чарт установить не получилось, устанавлвается и крашится под сонара.

Вроде всё описал.

#==================================================TheEnd================================================================

```
