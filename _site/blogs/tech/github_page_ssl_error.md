我在建立github-page本地文件的时候出现了错误。
1.  GitHub Metadata开头的警告：
    在github当中新建一个Personal access tokens，添加到系统环境变量，就可以解决。
2. Liquid Exception: SSL_connect returned=1 errno=0 state=error:..
    这个错误主要是由于Google需要验证证书。需要给系统添加一个证书，cacert.pem文件。

    下载http://curl.haxx.se/ca/cacert.pem
    复制到C:\Windows\
    运行下面命令:
    export SSL_CERT_FILE=c:/windows/cacert.pem
    重新运行jekyll serve

