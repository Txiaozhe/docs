* 注册GitHub账号，建立仓库

* 安装Hexo博客框架、GitHub客户端、NodeJs

  * ```shell
    $ sudo npm install hexo-cli -g //安装Hexo
    ```

* 创建博客

  * ```shell
    $ hexo init username.github.io
    ```

  * 更改配置

    * 安装主题 

    ```shell
    $ cd username.github.io
    $ git clone https://github.com/iissnan/hexo-theme-next themes/next
    ```

    [上述主题](https://github.com/monniya/hexo-theme-new-vno)

    [更多主题](https://hexo.io/themes/)

    * 更改配置：打开username.github.io/_config.yml设置以下键值对（注意冒号后面要加空格）

    ```json
    title: 我的个人博客  //你博客的名字
    author: xiaozhe  //作者名字
    language: zh-cn    //语言 中文，如果是英文则写 en-us
    theme: next   //刚刚安装的主题名称
    type: git    //使用Git 发布
    repo: https://github.com/username/username.github.io.git    // 刚创建的Github仓库
    ```

    [更多设置](https://hexo.io/zh-cn/docs/configuration.html)

    主题配置在 username.github.io/themes/next/_config.yml 中修改

    * 写文章: 在username.github.io/source/_posts下创建博客，例如创建一个名为First.md文件

    ```markdown
    ---
    title: First
    ---
    > 我是一只小小小鸟。。。
    ```

    * 测试

    ```shell
    $ hexo s
    ```

    * 安装[hexo-deployer-git](https://github.com/hexojs/hexo-deployer-git)自动部署发布工具

    ```shell
     $ npm install hexo-deployer-git --save
    ```

    测试服务启动，可以打开[https://localhost:4000/](https://localhost:4000/)访问

    * 测试没问题后，可以发布啦

    ```shell
    $ hexo clean && hexo g && hexo d
    ```

    ​