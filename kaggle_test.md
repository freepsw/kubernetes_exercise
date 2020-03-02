## 1. Install Docker environment

### Install docker
```
sudo yum remove docker \
docker-common \
container-selinux \
docker-selinux \
docker-engine

sudo yum install -y yum-utils device-mapper-persistent-data lvm2

sudo yum-config-manager \
--add-repo \
https://download.docker.com/linux/centos/docker-ce.repo

sudo yum-config-manager --enable docker-ce-edge

sudo yum makecache fast

sudo yum install docker-ce

sudo systemctl start docker
```

### Install docker-compose
```
sudo yum install epel-release
sudo yum install python-devel
sudo yum install -y python-pip
sudo pip install --upgrade pip
sudo pip install docker-compose
```

## 2. Run datascience docker
```
> sudo docker run --rm -p 10000:8888 -e JUPYTER_ENABLE_LAB=yes -v "$PWD":/home/jovyan/work jupyter/datascience-notebook:3772fffc4aa4
```


##. 3. Kaggle 사용하기
- https://www.kaggle.com/rochellesilva/simple-tutorial-for-beginners
## Install kaggle api
- https://github.com/Kaggle/kaggle-api

### Install pip
```
sudo yum install python-pip
sudo yum install python-devel
sudo yum groupinstall 'development tools'
```

### Install kaggle
```
sudo pip install kaggle
sudo pip install --upgrade pip
```


# logstash example
## collect twitter logs
```yaml
input {
  twitter {
    consumer_key       => "INSERT YOUR CONSUMER KEY"
    consumer_secret    => "INSERT YOUR CONSUMER SECRET"
    oauth_token        => "INSERT YOUR ACCESS TOKEN"
    oauth_token_secret => "INSERT YOUR ACCESS TOKEN SECRET"
    keywords           => [ "thor", "spiderman", "wolverine", "ironman", "hulk"]
    full_tweet         => true
  }
}

filter { }

output {
  stdout {
    codec => rubydebug{ }
  }
}

```
