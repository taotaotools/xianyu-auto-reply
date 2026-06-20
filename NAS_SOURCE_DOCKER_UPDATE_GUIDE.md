# 飞牛 NAS 本地源码 Docker 部署更新指南

适用场景：项目源码放在飞牛 NAS 上，通过 `docker compose up -d --build` 本地构建运行。

当前示例源码目录：

```bash
/vol3/1000/docker/xianyu-reply-github/xianyu-auto-reply-source
```

当前访问端口：

```text
http://你的NAS_IP:9100
```

## 一、更新前说明

程序正在运行时可以更新，不需要提前手动停止。

执行：

```bash
docker compose up -d --build
```

Docker 会重新构建镜像，并按需替换容器。更新过程中可能会有短暂不可访问，通常等容器重新启动健康后即可恢复。

注意：不要执行下面这种命令：

```bash
docker compose down -v
```

`-v` 会删除数据卷，可能导致数据库、Redis、浏览器数据等被清空。

## 二、标准更新流程

进入源码目录：

```bash
cd /vol3/1000/docker/xianyu-reply-github/xianyu-auto-reply-source
```

查看当前是否有本地修改：

```bash
git status
```

备份当前 Docker 配置文件：

```bash
cp docker-compose.yml docker-compose.yml.bak
```

拉取 GitHub 最新代码：

```bash
git pull
```

检查关键配置是否还在：

```bash
grep -n "HOST=0.0.0.0\|FRONTEND_PORT\|9100\|9000" docker-compose.yml
```

重新构建并启动：

```bash
docker compose up -d --build
```

查看容器状态：

```bash
docker compose ps
```

测试前端反代后端是否正常：

```bash
curl -i http://127.0.0.1:9100/health
```

如果返回 `200 OK`，浏览器访问：

```text
http://你的NAS_IP:9100
```

## 三、需要保留的自定义配置

因为你之前遇到过前端访问后端 `502 Bad Gateway`，所以 `backend-web`、`websocket`、`scheduler` 三个服务的 `environment` 里建议保留：

```yaml
- HOST=0.0.0.0
```

示例：

```yaml
backend-web:
  environment:
    - ENVIRONMENT=production
    - HOST=0.0.0.0
```

```yaml
websocket:
  environment:
    - ENVIRONMENT=production
    - HOST=0.0.0.0
```

```yaml
scheduler:
  environment:
    - ENVIRONMENT=production
    - HOST=0.0.0.0
```

前端端口建议保留为 `9100`：

```yaml
frontend:
  ports:
    - "${FRONTEND_PORT:-9100}:80"
```

如果 `.env` 文件里有：

```env
FRONTEND_PORT=9020
```

需要改成：

```env
FRONTEND_PORT=9100
```

可以用：

```bash
sed -i 's/^FRONTEND_PORT=.*/FRONTEND_PORT=9100/' .env
```

如果 `.env` 里没有 `FRONTEND_PORT`，不用处理，`docker-compose.yml` 里的默认 `9100` 会生效。

## 四、如果 git pull 出现冲突

如果执行 `git pull` 后提示冲突，先不要继续构建。

查看冲突文件：

```bash
git status
```

如果冲突主要在 `docker-compose.yml`，重点保留这几处：

```yaml
- HOST=0.0.0.0
```

以及：

```yaml
- "${FRONTEND_PORT:-9100}:80"
```

处理完冲突后执行：

```bash
git add docker-compose.yml
git commit -m "Resolve local docker compose config"
```

然后再构建：

```bash
docker compose up -d --build
```

如果不想处理冲突，也可以先把当前修改临时保存：

```bash
git stash
git pull
git stash pop
```

如果 `git stash pop` 仍提示冲突，按上面的方式处理。

## 五、常见问题

### 1. 端口被占用

如果提示：

```text
Bind for 0.0.0.0:9100 failed: port is already allocated
```

查看谁占用了端口：

```bash
docker ps | grep 9100
```

如果是其他服务占用，把前端端口改成其他端口，例如 `9110`：

```bash
sed -i 's/^FRONTEND_PORT=.*/FRONTEND_PORT=9110/' .env
```

如果 `.env` 不存在：

```bash
echo "FRONTEND_PORT=9110" > .env
```

然后：

```bash
docker compose up -d
```

### 2. 前端能打开，但登录提示网络错误

先测试：

```bash
curl -i http://127.0.0.1:9100/health
```

如果返回 `502 Bad Gateway`，检查 `docker-compose.yml` 里三个后端服务是否都有：

```yaml
- HOST=0.0.0.0
```

然后重建容器：

```bash
docker compose up -d --force-recreate backend-web websocket scheduler frontend
```

### 3. 查看日志

查看所有服务日志：

```bash
docker compose logs -f --tail=100
```

查看前端日志：

```bash
docker compose logs --tail=100 frontend
```

查看后端日志：

```bash
docker compose logs --tail=100 backend-web
```

### 4. 镜像拉取失败

如果构建时报类似：

```text
401 Unauthorized
failed to resolve source metadata
```

一般是 NAS 的 Docker 镜像加速源失效。需要在飞牛 NAS 的 Docker 设置里删除或更换失效的镜像加速地址，然后重启 Docker，再重新执行：

```bash
docker compose up -d --build
```

## 六、一键复制版命令

正常更新时可以直接按顺序执行：

```bash
cd /vol3/1000/docker/xianyu-reply-github/xianyu-auto-reply-source
git status
cp docker-compose.yml docker-compose.yml.bak
git pull
grep -n "HOST=0.0.0.0\|FRONTEND_PORT\|9100\|9000" docker-compose.yml
docker compose up -d --build
docker compose ps
curl -i http://127.0.0.1:9100/health
```

确认正常后访问：

```text
http://你的NAS_IP:9100
```
