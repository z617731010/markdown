# Python3.x 配置原生虚拟环境

Python 3.4 之后支持原生的虚拟环境配置(3.3的虚拟环境不支持pip)，把配置过程记录一下备忘。

 

### 1.创建虚拟环境

在控制台中，使用cd目录，切换到需要创建虚拟环境的目录。

使用如下命令，在当前目录创建虚拟环境。

    python -m venv <name>
    python -m venv venv
 

创建成功后，在目录下，有如下文件：

Windows:
    
    F:.
    └─pygame
    ├─Include
    ├─Lib
    │  ├─site-packages
    │  └─tcl8.6
    └─Scripts


### 2.激活虚拟环境

切换到已成功创建虚拟环境的目录，执行如下命令，激活虚拟环境

Windows:
 
    Scripts\activate.bat
激活成功后，会在控制台中显示虚拟环境的名称。

 

### 3.在虚拟环境安装包

在成功激活虚拟环境的情况下，使用如下命令在线安装包

    pip install <package>
例如安装pygame

    pip install pygame
对于下载到本地的安装包，也在激活虚拟环境的情况下，使用cd命令切换到包所在目录，使用类似如下的命令进行安装

    python setup.py install
安装成功后，使用pip list命令查看已安装的包

    pip list