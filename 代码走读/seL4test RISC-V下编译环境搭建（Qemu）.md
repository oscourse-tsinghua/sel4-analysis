## 安装依赖

按照 [Host Dependencies | seL4 docs](https://docs.sel4.systems/projects/buildsystem/host-dependencies.html) 安装相关依赖，特别是 **Cross-compiling for RISC-V targets**。

## 代码编译

下载代码：
```
mkdir seL4test
cd seL4test
repo init -u https://github.com/seL4/sel4test-manifest.git
repo sync
```

运行代码（在seL4test目录中）：
```
mkdir build && cd build
../init-build.sh -DPLATFORM=spike -DSIMULATION=TRUE
ninja
./simulation
```

## 配置VS code

在插件商店中安装 `CMake` 和 `CMake Tools` 。

在 .vscode/settings.json 中添加：
```json
	"cmake.configureOnOpen": true,
    "cmake.configureSettings": {
        "PLATFORM": "spike",
        "SIMULATION": true,
    },
    "cmake.sourceDirectory": "${workspaceFolder}/projects/sel4test"
```
