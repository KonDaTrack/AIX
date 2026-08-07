# ROS 2 Docker 自动化搭建方案（含 GPU 加速与硬件穿透）

本教程提供一份针对 ROS 机器人开发者的完整 Docker 自动化搭建方案，覆盖 **CUDA 硬件穿透**、**GPU 硬件加速**、**串口通信设备挂载**、**局域网（DDS）通信**、**Gazebo 仿真器配置**、**非 Root 用户权限管理** 以及 **VS Code/Cursor Remote 开发环境适配**。

---

## 1. 宿主机前置准备 (Host Setup)

在创建 Docker 容器前，请先在宿主机配置好 NVIDIA 显卡驱动、NVIDIA Container Toolkit 以及必要的硬件访问权限。

### 1.1 串口访问权限配置

将宿主机当前用户加入 `dialout` 用户组，防止后续串口 / USB 设备（如 CH340、激光雷达）出现 `Permission Denied` 错误：

```
sudo usermod -aG dialout $USER
```

### 1.2 开启 X11 图形界面服务

运行以下命令允许 Docker 容器访问宿主机的图形显示服务（每次重启宿主机后需运行一次）：

```
xhost +local: >> /dev/null
```

> 
> 提示：可将 `xhost +local: >> /dev/null` 写入宿主机的 `~/.bashrc` 末尾，实现开机自动生效。

### 1.3 安装 NVIDIA Container Toolkit（用于 CUDA 与 GPU 加速）

确保宿主机已正确安装 NVIDIA 显卡驱动（运行 `nvidia-smi` 能够正常显示显卡状态）。随后安装 Toolkit：

```
# 1. 添加 apt 源
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb [arch=amd64] https://#deb [arch=amd64 signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# 2. 安装 Toolkit
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

# 3. 配置并重启 Docker
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

---

## 2. 一键创建专属镜像 (Image Customization)

ROS 官方基础镜像（如 `osrf/ros:jazzy-desktop-full`）未预装 Gazebo 仿真器本体，且默认缺失非 Root 开发用户 `robot`。我们将通过 3 个步骤构建专属镜像。

### 2.1 第一步：启动临时配置容器

使用官方 ROS 2 Jazzy 镜像启动临时容器：

```
sudo docker run -dit \
  --name=ros_jazzy_temp \
  --gpus all \
  --privileged \
  -v /dev:/dev \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -e DISPLAY=unix$DISPLAY \
  -e NVIDIA_VISIBLE_DEVICES=all \
  -e NVIDIA_DRIVER_CAPABILITIES=all \
  --net=host \
  osrf/ros:jazzy-desktop-full
```

### 2.2 第二步：进入容器并安装组件与用户配置

进入临时容器：

```
sudo docker exec -it ros_jazzy_temp bash
```

在容器内部终端中，一次性粘贴并运行以下配置脚本：

```
# 1. 更新源并安装 sudo 与 Gazebo Harmonic 仿真器
apt-get update && apt-get install -y \
    sudo \
    ros-jazzy-ros-gz

# 2. 创建 robot 开发用户，配置默认 bash shell 并生成主目录
useradd -m -s /bin/bash robot

# 3. 将 robot 加入 sudo 组（管理员）与 dialout 组（串口通信）
usermod -aG sudo,dialout robot

# 4. 配置 robot 用户 sudo 免密（方便 Agent 及自动化部署）
echo "robot ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers.d/robot
chmod 0440 /etc/sudoers.d/robot

# 5. 配置 ROS 2 Jazzy 环境变量自动加载
echo "source /opt/ros/jazzy/setup.bash" >> /home/robot/.bashrc
chown -R robot:robot /home/robot

# 6. 退出容器
exit
```

### 2.3 第三步：打包容器为专属本地镜像

在宿主机终端中执行打包并清理临时容器：

```
# 打包临时容器为自定义镜像 my_ros:jazzy
sudo docker commit ros_jazzy_temp my_ros:jazzy

# 清理临时容器
sudo docker rm -f ros_jazzy_temp
```

---

## 3. 运行最终版 Docker 容器 (Container Deployment)

基于自定义镜像 `my_ros:jazzy`，以普通用户 `robot` 身份启动最终的开发容器。

### 3.1 部署启动指令

```
sudo docker run -dit \
  --name=ros_jazzy \
  -u robot \
  -w /home/robot \
  --gpus all \
  --privileged \
  -v /dev:/dev \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -e DISPLAY=unix$DISPLAY \
  -e NVIDIA_VISIBLE_DEVICES=all \
  -e NVIDIA_DRIVER_CAPABILITIES=all \
  --net=host \
  my_ros:jazzy
```

### 3.2 核心参数解析清单

表格

| 参数 | 解决的关键问题 / 功能 |
| --- | --- |
| `-u robot` | 默认登录用户切换为非 Root 用户 robot，保障开发安全。 |
| `-w /home/robot` | 默认初始化工作路径设置为 `/home/robot`。 |
| `--gpus all` | 挂载宿主机显卡，提供 CUDA 算力穿透。 |
| `-e NVIDIA_DRIVER_CAPABILITIES=all` | 开启图形渲染（Graphics/OpenGL/Vulkan），为 Gazebo 3D 渲染与 RViz2 提供原生 GPU 加速。 |
| `--privileged + -v /dev:/dev` | 串口 / 硬件映射：完整挂载 `/dev` 设备目录并给予特权，支持 CH340 模块、激光雷达、深度相机热插拔访问。 |
| `-v /tmp/.X11-unix:... + -e DISPLAY=...` | GUI 图形界面：将宿主机 X11 Socket 传入容器，支持打开 RViz2、Gazebo、Gedit 等窗口软件。 |
| `--net=host` | 局域网通信：共享宿主机网络命名空间，ROS 2 DDS 节点无需额外映射即可直接同局域网内其他设备通信。 |

---

## 4. 日常开发与功能验证 (Usage & Verification)

### 4.1 快捷进入容器

在宿主机运行以下命令即可直接以 `robot` 身份进入容器：

```
sudo docker exec -it ros_jazzy bash
```

> 
> 建议：在宿主机 `~/.bashrc` 中添加别名：
> 
> 
> ```
> alias enter_ros="sudo docker exec -it ros_jazzy bash"
> ```
> 
> 
> 后续直接输入 `enter_ros` 即可一键进入。

### 4.2 功能验证测试

#### 验证 CUDA 与 GPU 硬件加速

进入容器终端运行：

```
nvidia-smi
```

能输出显卡型号与驱动信息即表明 GPU 穿透正常。

#### 验证 Gazebo 仿真器与 3D GPU 加速

在容器终端运行 Gazebo Sim：

```
gz sim
```

能够打开 Gazebo 3D 界面，同时在宿主机运行 `nvidia-smi` 可看到 gz-sim 或 ruby 进程占用显存，证明已成功调用 GPU 硬件渲染。

#### 验证 ROS 2 环境

进入容器终端直接验证：

```
ros2 topic list
```

能直接显示 Topic 列表而无需手动 source，表明 ROS 2 环境配置自动加载成功。

#### 切换 Root 权限

在容器内部如需升级软件或安装系统依赖，直接使用 `sudo` 即可：

```
sudo apt update
```

如需临时切换到 root 交互环境，运行 `sudo -i`（无需输入密码）。