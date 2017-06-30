本地mysql

username: root

password: 126917

重置密码：

* 停止mysql服务

* 打开终端：

  `$ sudo /usr/local/mysql/bin/mysqld_safe --skip-grant-tables`

* 打开另一终端

  `$ sudo /usr/local/mysql/bin/mysql -u root`

  输入密码即可



不区分大小写：

显示数据库：mysql> show databases；

重置密码： mysql> SET PASSWORD = PASSWORD(‘新密码’)；

显示当前选择的数据库：mysql> select database();

显示当前选择的数据库的版本： mysql> select version();

显示当前时间：mysql> select now();