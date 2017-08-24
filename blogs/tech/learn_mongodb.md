1. 进入[官网下载](https://www.mongodb.org/?_ga=1.6778743.416898270.1442579015)：

下载32bit版本（只能使用最多2G空间）;

2. 安装，将bin目录加入环境变量

3. 建立数据库，同时也会开启这个数据库
```shell
mongod --dbpath D:\develop_kit\MongoDB\data
```
4. 在网站下载测试使用的数据集resturant

5. 导入测试数据
```shell
mongoimport --db test --collection restaurants --drop --file primer-dataset.json
```
6. 使用python进行测试：

[测试数据所在网页](https://docs.mongodb.org/getting-started/python/import-data/)

[需要安装pymongo](https://docs.mongodb.org/getting-started/python/client/)

7. [具体命令参考](https://docs.mongodb.org/getting-started/python/client/)

[back](../../index.md)