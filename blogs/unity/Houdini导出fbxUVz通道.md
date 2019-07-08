# Houdini无法导出UV.z通道

目前希望在Houdini当中写入一些控制参数，在Unity中读取出来进行一些操作。

法线当在Houdini中给uv的z通道写入数据时，在Unity当中无法读取出来。

通过检查Houdini的输出FBX文件，发现Houdini的ROP FBXexport导出结点无法导出uv的z通道。

目前的解决方案就是不适用。

之后的方案可以考虑使用Houdini的API进行修改导出设置。

