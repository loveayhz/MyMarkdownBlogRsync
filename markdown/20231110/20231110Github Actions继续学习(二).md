# Github Actions继续学习(二)

巩固一下github actions的知识

1通过github  actions checkout项目，项目中使用脚本或程序下载或生成内容，将生成内容提交到当前github并合并，即类似使用python爬虫获取数据并将数据保存到项目中。

2.chrome插件开发记录chrome浏览器历史，并同步到mysql或者webdav

---

参考：https://github.com/zjkwdy/bili_app_splash/blob/main/.github/workflows/bilibili_splash_download.yml

[使用Github Action自动merge pull request](https://www.huangyunkun.com/2020/01/29/github-action-merge-pull-request/)

将改动文件提交到当前仓库：https://github.com/ad-m/github-push-action

现在的问题是我们生成什么内容呢rsshub的订阅？

可以，但是我们先实现生成文件然后提交合并，也就是让一个github action主动去扩建内容。

---

问题出在：

```
- name: Push changes
        uses: ad-m/github-push-action@master
        with:
          branch: ${{ github.head_ref }}
          github_token: ${{ secrets.GHTO }}
```

https://github.com/ad-m/github-push-action/issues/96

一直提示权限不足，我已经使用了全部仓库的全部permissions

即使使用以前的token权限也不行

```
un ad-m/github-push-action@master
  with:
    github_token: ***
    github_url: https://github.com
    directory: .
Push to branch main
remote: Permission to loveayhz/MyMarkdownBlogRsync.git denied to github-actions[bot].
fatal: unable to access 'https://github.com/loveayhz/MyMarkdownBlogRsync.git/': The requested URL returned error: 403
Error: Invalid exit code: 128
    at ChildProcess.<anonymous> (/home/runner/work/_actions/ad-m/github-push-action/master/start.js:30:21)
    at ChildProcess.emit (node:events:514:28)
    at maybeClose (node:internal/child_process:1105:16)
    at ChildProcess._handle.onexit (node:internal/child_process:305:5) {
  code: 128
}
```

对于git我们需要更详细的了解：[Git 里面的 origin 到底代表啥意思?](https://www.zhihu.com/question/27712995)

了解完这些之后，又看完关于dependabot相关的内容：https://docs.github.com/zh/code-security/dependabot/dependabot-alerts/about-dependabot-alerts

因为我看到他的提交账户使用了github-actions[bot],搜索的时候有提到dependabot

然后发现dependabot可能涉及一些权限问题去主动提交，主动合并等等，目前还没有特别明白到底需要哪些权限，权限能开的都开了目前可以成功执行。

但是还有个问题就是在除了main分支之外的分支执行都没有成功。

但是main分支就可以，这点还需要再了解透彻。同时设置的on.schedule.cron也没有按时去执行。

但是合并到主分支执行成功的，没有错误，包括邮件也接收到了。

![actions配置.png](http://ayhz.art:8084/zh-east-1/img/actions配置.png)

```
# This is a basic workflow to help you get started with Actions

name: CI


# Set the access for individual scopes, or use permissions: write-all
permissions: write-all
  #pull-requests: write
  #issues: write
  #repository-projects: write
# Controls when the workflow will run
on:
  # Triggers the workflow on push or pull request events but only for the "main" branch
  push:
    branches: [ "main","workflow" ]
  #pull_request:
    # branches: [ "main" ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:
  schedule:
    # * is a special character in YAML so you have to quote this string
    - cron:  '55 12 * * *'

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
      - name: Run a one-line script
        run: echo Hello, world!

      # Runs a set of commands using the runners shell
      - name: Run a multi-line script
        run: |
          echo Add other actions to build,
          echo test, and deploy your project.
      - name: Get Markdown content and transfer to markdown file from My blog api
        run: |
          curl -o /home/runner/work/MyMarkdownBlogRsync/MyMarkdownBlogRsync/today.json ${{ secrets.BLOG_URL }}
          echo test!
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
          body: file://today.json
          to: ${{ secrets.NOTICFICATION_EMAIL }}
          from: GitHub Actions


```

接下来除了解决上面的问题外，我们还需要进一步实现我们的想法。

能力已经有了，但是如果说我们想实现从服务器获取当天发表的文章，同步markdown文件到github，

似乎通过shell还是有点难的，因为需要分好几步操作，第一步是获取权限，第二步是拿到文章列表，第二步是分别获取每个文件并存储为markdown文件。

或许我们可以把思路再打开一下，既然前面我们已经实现了通过minio/mc同步文件，那么我们的博客服务本身又与minio服务是有交互的，为什么不在保存，修改博客文章的时候，将数据推送到minio的指定bucket中比如article，但是权限私有，然后我们通过mc复制博客文章到gitpage，并进行提交和push，然后自动触发gitpage的发布deploy。

这样不是完美解决了同步的问题吗？

同时我们的评论也可以转化为issue，配合git talk进行使用。

---

📅2023-11-10 测试保存markdown原文件到minio

需要考虑文件名命名，因为我们肯定是需要增量的来拉取的，最好加上日期或者时间戳。

其实获取当天的或者本月发布的博客的逻辑更多的放在这边比较好，因为用shell执行命令尽可能简单的好，所以，我们可以用日期来yyyymmdd命名文件夹，这样读取的时候只要匹配当天的或者昨天的日期即可。
