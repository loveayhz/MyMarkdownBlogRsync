前天我们学习了github actions的文档，并且使用docker作为运行容器执行mc的备份操作

定时任务的时间设置可以参考：crontab：https://crontab.guru/

从下图可以看到两个CI的执行，因为昨天修改了一下，修改完手动执行了一次

同时最后一次执行是16个小时前，现在是上午10:30多，因为使用的是UTC时间，所以就是8个小时前，也就是北京时间凌晨两点半，而且登录树莓派minio也可以看到img数据都被完全同步了过来。

![githubactions运行截图png.png](http://ayhz.art:8084/zh-east-1/img/githubactions运行截图png.png)

但是这里有个问题，昨天执行的时候报错了：

```
Run mc cp --recursive sourceBucket/$SOURCE_MINIO_BUCKET  destBucket/$DEST_MINIO_BUCKET 2>&1 /dev/null
  mc cp --recursive sourceBucket/$SOURCE_MINIO_BUCKET  destBucket/$DEST_MINIO_BUCKET 2>&1 /dev/null
  shell: sh -e {0}
mc: <ERROR> Unable to prepare URL for copying. Unable to guess the type of copy operation.
```

再次手动执行一遍进行验证还是同样的报错，一通操作后发现重定向的语法少了个>

---

关于重定向的问题：

[2>/dev/null和>/dev/null 2>&1和2>&1>/dev/null的区别](https://ljlazyl.blog.csdn.net/article/details/90519690)

> 基本概念：
>
> **1、文件描述符**
> Linux系统预留可三个文件描述符：0、1和2，他们的意义如下所示：
> 0——标准输入（stdin）
> 1——标准输出（stdout）
> 2——标准错误（stderr）
>
> 2、重定向
> 重定向的符号有两个：>或>>，两者的区别是：前者会先清空文件，然后再写入内容，后者会将重定向的内容追加到现有文件的尾部。
>
> 一、区别：
> 2>/dev/null
> 意思就是把错误输出到“黑洞”
>
>> /dev/null 2>&1
>> 默认情况是1，也就是等同于1>/dev/null 2>&1。意思就是把标准输出重定向到“黑洞”，还把错误输出2重定向到标准输出1，也就是标准输出和错误输出都进了“黑洞”
>>
>
> 2>&1 >/dev/null
> 意思就是把错误输出2重定向到标准出书1，也就是屏幕，标准输出进了“黑洞”，也就是标准输出进了黑洞，错误输出打印到屏幕

这里我们使用的是2>&1 >/dev/null，即把标准输出扔进黑洞（不显示），只显示标准错误

---

其实这里minio/mc有一个全局命令--quiet不输出结果

但是更好的办法是&>mcBackup.log，将结果都输出到指定文件中，然后将指定文件同步到minio

---

修改后，删除无用的echo语句，成功执行，但是问题是执行了长达2min43s，因为是全量备份，这样会做很多无用功，最主要的是带宽浪费而我们服务器带宽本身又很小。

![成功且为输出详细日志.png](http://ayhz.art:8084/zh-east-1/img/成功且为输出详细日志.png)

修改后的

```
# This is a basic workflow to help you get started with Actions

name: CI

# Controls when the workflow will run
on:
  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:
  schedule:
    # * is a special character in YAML so you have to quote this string
    - cron:  '44 18 * * *'
# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  container_job:
    runs-on: ubuntu-latest
    container:
      image: minio/mc:RELEASE.2021-04-22T17-40-00Z
    steps:
      - name: alies minio bucket
        env:
          DEST_MINIO_HOST: ${{ secrets.DEST_MINIO_HOST }}
          DEST_MINIO_ACCESS_KEY: ${{ secrets.DEST_MINIO_ACCESS_KEY }}
          DEST_MINIO_SECRET_KEY: ${{ secrets.DEST_MINIO_SECRET_KEY }}
          DEST_MINIO_BUCKET: ${{ secrets.DEST_MINIO_BUCKET }}
          SOURCE_MINIO_HOST: ${{ secrets.SOURCE_MINIO_HOST }}
          SOURCE_MINIO_ACCESS_KEY: ${{ secrets.SOURCE_MINIO_ACCESS_KEY }}
          SOURCE_MINIO_SECRET_KEY: ${{ secrets.SOURCE_MINIO_SECRET_KEY }}
          SOURCE_MINIO_BUCKET: ${{ secrets.SOURCE_MINIO_BUCKET }}
        run: (mc alias set destBucket $DEST_MINIO_HOST $DEST_MINIO_ACCESS_KEY $DEST_MINIO_SECRET_KEY && mc alias set sourceBucket $SOURCE_MINIO_HOST $SOURCE_MINIO_ACCESS_KEY $SOURCE_MINIO_SECRET_KEY)
      - name: buckupAll
        run: mc cp --recursive sourceBucket/$SOURCE_MINIO_BUCKET  destBucket/$DEST_MINIO_BUCKET 2>&1 >/dev/null
```

所以还是考虑增量同步以及镜像同步双重保险，

树莓派上依然开启镜像，但是使用docker-compose部署

github actioons增量同步每周一次

将日志输出到指定文件，再将指定文件用mc cp到目标minio对应的bucket中去：

```
# This is a basic workflow to help you get started with Actions

name: CI

# Controls when the workflow will run
on:
  push:
    branches:
      - 'main'
      - 'feat/workflows/**'
  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:
  schedule:
    # * is a special character in YAML so you have to quote this string
    - cron:  '30 10 * * *'
# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  container_job:
    runs-on: ubuntu-latest
    container:
      image: minio/mc:RELEASE.2021-04-22T17-40-00Z
    steps:
      - name: alies minio bucket
        env:
          DEST_MINIO_HOST: ${{ secrets.DEST_MINIO_HOST }}
          DEST_MINIO_ACCESS_KEY: ${{ secrets.DEST_MINIO_ACCESS_KEY }}
          DEST_MINIO_SECRET_KEY: ${{ secrets.DEST_MINIO_SECRET_KEY }}
          DEST_MINIO_BUCKET: ${{ secrets.DEST_MINIO_BUCKET }}
          SOURCE_MINIO_HOST: ${{ secrets.SOURCE_MINIO_HOST }}
          SOURCE_MINIO_ACCESS_KEY: ${{ secrets.SOURCE_MINIO_ACCESS_KEY }}
          SOURCE_MINIO_SECRET_KEY: ${{ secrets.SOURCE_MINIO_SECRET_KEY }}
          SOURCE_MINIO_BUCKET: ${{ secrets.SOURCE_MINIO_BUCKET }}
        run: |
          mc alias set destBucket $DEST_MINIO_HOST $DEST_MINIO_ACCESS_KEY $DEST_MINIO_SECRET_KEY
          mc alias set sourceBucket $SOURCE_MINIO_HOST $SOURCE_MINIO_ACCESS_KEY $SOURCE_MINIO_SECRET_KEY
      - name: buckupAll
        run: |
          mc cp --recursive sourceBucket/$SOURCE_MINIO_BUCKET  destBucket/$DEST_MINIO_BUCKET &>>mcBackup.log
          mc cp ./mcBackup.log  destBucket/mcbackup
```

注意：mc客户端cp复制时，如果目前的bucket不存在会出错。

同步的action全部内容：

```
# This is a basic workflow to help you get started with Actions

name: CI


# Set the access for individual scopes, or use permissions: write-all
permissions: write-all
# Controls when the workflow will run
on:
  # Triggers the workflow on push or pull request events but only for the "main" branch
  push:
    branches: [ "main" ]
  #pull_request:
    # branches: [ "main" ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:
  schedule:
    # * is a special character in YAML so you have to quote this string
    - cron:  '30 21 * * *'

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  build:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest

    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - uses: actions/checkout@v3
        with:
         #ref: ${{ github.head_ref }}
         fetch-depth: 0
      # Runs a single command using the runners shell
      - name: chmod mc
        run: chmod u+x ./minio-binaries/mc

      # Runs a set of commands using the runners shell
      - name: Cp markdown files to local fileSystem
        env:
          DEST_MINIO_HOST: ${{ secrets.DEST_MINIO_HOST }}
          DEST_MINIO_ACCESS_KEY: ${{ secrets.DEST_MINIO_ACCESS_KEY }}
          DEST_MINIO_SECRET_KEY: ${{ secrets.DEST_MINIO_SECRET_KEY }}
          DEST_MINIO_BUCKET: ${{ secrets.DEST_MINIO_BUCKET }}
        run: |
          ./minio-binaries/mc alias set remote $DEST_MINIO_HOST $DEST_MINIO_ACCESS_KEY $DEST_MINIO_SECRET_KEY
          ./minio-binaries/mc cp --recursive remote/$DEST_MINIO_BUCKET/`date +"%Y%m%d"` ./$DEST_MINIO_BUCKET/ &> today.log
          echo copy today markdown file to local folder!
      - name: Commit files
        run: |
          git config --local user.email "github-actions[bot]@users.noreply.github.com"
          git config --local user.name "github-actions[bot]"
          git add .
          git commit -a -m "Add changes"
      - name: Push changes
        uses: ad-m/github-push-action@master
        with:
          #branch: ${{ github.head_ref }}
          github_token: ${{ secrets.GHTO }}
      - name: "Send mail"
        uses: dawidd6/action-send-mail@master
        with:
          server_address: smtp.163.com
          server_port: 465
          username: ${{ secrets.MAILUSERNAME }}
          password: ${{ secrets.MAILPASSWORD }}
          subject: BLOG sync success
          body: file://today.log
          to: ${{ secrets.NOTICFICATION_EMAIL }}
          from: GitHub Actions


```

![1613798142934spH5SFph.jpeg](http://ayhz.art:8084/zh-east-1/img/1613798142934spH5SFph.jpeg)
