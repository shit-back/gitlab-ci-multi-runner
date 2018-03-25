# 仅记录部署的yml编写，前提是用户已经安装了gitlab-runner

## 注册gitlab-ci-multi-runner

- $  gitlab-ci-multi-runner register #引导会让你输入gitlab的url，输入自己的url，例如http://gitlab.example.com/
- 引导会让你输入token，去相应的项目下找到token，例如ase12c235qazd32
- 引导会让你输入tag，一个项目可能有多个runner，是根据tag来区别runner的，输入若干个就好了，比如web,hook,deploy
- 引导会让你输入executor，这个是要用什么方式来执行脚本，图方便输入shell就好了。

## 项目根目录编写.gitlab-ci.yml，gitlab-ci会自动识别解析
- .gitlab-ci.yml
```
stages:
    - deploy

deploy_production:
    stage: deploy
    script:
        - echo "production to staging server"
        - ~/deploy test-cli master
        - ~/node_build test-cli master
    only:
        - master
    tags:
        - release
    environment: production

deploy_staging:
    stage: deploy
    script:
        - echo "develop to staging server"
        - ~/deploy test-cli develop
        - ~/node_build test-cli develop
    only:
        - develop
    tags:
        - release
    environment: develop
```
- 说明
我们只有一个stage是deploy。only指定了只有在masterh或者develop分支push的时候才会被执行。tags是release，对应了刚才注册runner的时候的tags。script部分deploy和node_build shell指令。

## shell
```
$ su gitlab-runner
$ cd ~/
$ touch deploy
$ touch node_build
```

- deploy
```
#!/bin/bash
if [ $# -ne 2 ]
then
      echo "arguments error!"
      exit 1
else
      deploy_path="/var/www/html/$1-$2"
      if [ ! -d "$deploy_path" ]
      then
            project_path="git@gitlab.com:xxx"/$1".git"
            git clone $project_path $deploy_path
      else
            cd $deploy_path
            git checkout $2
            git stash
            git pull
      fi
fi

```
- node_build
```
#!/bin/bash
if [ $# -ne 2 ]
then
      echo "arguments error!"
      exit 1
else
      deploy_path="/var/www/html/$1-$2
      if [ ! -d "$deploy_path" ]
      then
            echo "can not find "
            exit 2
      else
            cd $deploy_path
            /usr/local/src/node/bin/cnpm install
            /usr/local/src/node/bin/cnpm run develop
            /usr/local/bin/composer install
            chmod -R 777 $deploy_path/storage/*
            chmod -R 777 $deploy_path/bootstrap/*

            if [ ! -f $deploy_path"/.env" ]
            then
                cp $deploy_path"/.env.example"  $deploy_path"/.env"
                php artisan key:generate
            else
                echo $deploy_path"/.env"
            fi
      fi
fi
```
增加执行权限 chmod +x ~/deploy ~/node_build
本项目已laravel为例，npm和composer用户自行安装

## 配置ssh登录
```
$ mkdir ~/.ssh
$ cd ~/.ssh
$ ssh-keygen
# 提示输入一直按回车默认就可以了
$ cat id_rsa.pub
```

## 设置gitlab-runner文件执行权限

```
$ chown -hR gitlab-runner:gitlab-runner /var/www/html
```



