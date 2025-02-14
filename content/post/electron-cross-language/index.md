---
title: Electron跨语言通信
description: Electron 中实现与 Python 等其他语言通信的解决方案
date: 2024-10-23
categories:
    - frontend-world
tags:
    - electron
    - python
    - egg
image: cover.png
---
> 需求

项目中有些复杂操作，后面考虑使用python来处理。其实更好的跨语言方案我觉得用tauri wails之类的比较好，看起来会专业一些。不过我打算用electron，其他语言只是作为一个服务。我记得我在大学时候写electron就特别想把python集成进来，但是当时技术比较菜，也没想好的办法，这次使用egg来做，它集成了跨语言的解决方案。我看了一下源码，实际解决方案也很朴素，没有我想的会很高深。实际上就是实现了一个进程的控制。例如我打包一个fastapi程序，一个没有窗口的程序。electron中会启动这个进程，可以进行一些配置比如端口等，然后就正常通信，程序关闭的时候也关闭服务。

> 操作

### python端

新建一个python文件夹放入相关代码，然后配置打包脚本

```python
from cx_Freeze import setup, Executable

# 定义额外需要包含的包
includes = [
    "uvicorn",
    "uvicorn.loops",
    "uvicorn.loops.auto",
    "uvicorn.protocols",
    "uvicorn.protocols.http",
    "uvicorn.lifespan",
    "uvicorn.lifespan.on",
    "uvicorn.lifespan.off",
    "uvicorn.logging",
    "fastapi",
    "starlette",
    "pydantic"
]

# 创建可执行文件的配置
executableApp = Executable(
    script="main.py",
    target_name="pyapp",
)



# 打包的参数配置
options = {
    "build_exe": {
        "build_exe": "./dist/",
        "excludes": ["*.txt"],
        "optimize": 2,
        "includes": includes,
        "packages": includes,
        "zip_include_packages": "*",
        "zip_exclude_packages": "",
    }
}

setup(
    name="pyapp",
    version="1.0",
    description="python app",
    options=options,
    executables=[executableApp]
)
```

electron中配置启动

```javascript
  async createPythonServer() {
    // method 1: Use the default Settings
    //const entity = await Cross.run(serviceName);

    // method 2: Use custom configuration
    const serviceName = "python";
    const opt = {
      name: 'pyapp',
      cmd: path.join(Ps.getExtraResourcesDir(), 'py', 'pyapp'),
      directory: path.join(Ps.getExtraResourcesDir(), 'py'),
      args: ['--port=10305'],
      windowsExtname: true,
      appExit: true,
    }
    const entity = await Cross.run(serviceName, opt);
    Log.info('server name:', entity.name);
    Log.info('server config:', entity.config);
    Log.info('server url:', entity.getUrl());

    return;
  }
}
```
