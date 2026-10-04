---
name: dji-flight-pilot
description: "AI flight planning and automated ADB deployment for DJI Matrice 4T and DJI RC Plus 2. Generates standard WPML KMZ missions (orbit/POI) and pushes them to the remote controller."
metadata: { "openclaw": { "emoji": "🛸", "install": [] } }
---

# DJI Flight Pilot (Matrice 4T & RC Plus 2)

当用户想规划无人机航线、航拍照片/视频、环绕建筑或风景时使用本技能。

## 代码位置（跨主机约定）

飞行代理位于 **`$HOME/my4t/flight_agent.py`**：

- **mf2（手机，户外主用）**：`/data/data/com.termux/files/home/my4t/`（真实目录）
- **macmini（室内/开发）**：`~/my4t` → `~/Projects/my4t` 的软链

两条路径都解析成同一个 `$HOME/my4t`，所以命令写法在两地一致：

### 模式 A：围绕视线面前的目标环绕（现场零门槛）

用户在现场悬停/对准建筑，说**“围绕面前的建筑环绕”**、**“围绕这个建筑飞一圈”**时：

```bash
python3 "$HOME/my4t/flight_agent.py" orbit-front \
  --distance 40 \
  --height 35 \
  --speed 3.0 \
  --mode video \
  --name "面前目标环绕"
```

- 位姿优先级：**调用方显式给定 `--lat/--lon/--heading`** > 遥控器遥测。
  ⚠️ 遥测取自遥控器的 `dumpsys location`（**地面站自身**的定位/罗盘），
  **是否为飞机位置与机头朝向尚未在真机验证** —— 向用户回报时要说明这一点。
- 自动计算机头正前方 `--distance` 米处为圆心，以当前机位为起点规划 360° 闭环运镜。
- 机头全程对准圆心，云台自动下俯（俯仰角 = `-atan(高度/半径)`，非固定值）。
- **缺位置或缺机头朝向 → 直接报错退出**，不会伪造坐标、也不会静默取正北。
  此时**向用户询问**：要么传 `--heading`（0=正北，顺时针度数），要么改用地标坐标走模式 B。
  仅做链路演示才用 `--allow-mock`（使用演示坐标，**永远 `success: false`**，必须明确告知用户“不可飞”）。

### 模式 B：指定已知地标坐标环绕

用户提供地标经纬度或地名解析出坐标时：

```bash
python3 "$HOME/my4t/flight_agent.py" orbit \
  --lat <纬度> --lon <经度> \
  --radius <半径米> --height <高度米> \
  --speed <速度m/s> \
  --mode <photo|video> \
  --name "<任务名>"
```

**不要传 `--rc-ip`。** 控制器会自己按序探测：显式 IP（若有）→ 室内 `192.168.88.247`
→ 户外热点 `192.168.43.4` → 经 mf2 的隧道 `127.0.0.1:5556`，并校验设备确实是
`DJI RC PLUS`。只有自动探测失败时，才用 `--rc-ip <IP>` 指定。

## 参数默认与建议

> ⚠️ 本流程会在必要时 **`force-stop` 并冷启动 DJI Pilot 2**
> （把界面逼回已知首页；否则硬编码坐标会在错误页面上盲点，实测会冲进飞行界面）。
> 因此它**只能在起飞前**执行航线部署 —— 飞行中绝不调用本技能。

- `--radius` 默认 40m（建筑环绕建议 30–80m）
- `--height` 默认 30m（`orbit-front` 默认 35m；风景建议 25–60m）
- `--speed` 默认 2.0 m/s（视频宜 2.0，拍照可 3.0）
- `--points` 默认 16（`--mode photo` 的停靠点数）
- `--mode` `video`（连续环绕录像）或 `photo`（逐点拍照）
- `--name` 地标短名，如 `雷峰塔环绕录像`

## 执行前自检（失败就先修，别硬跑）

1. `adb devices` 能看到遥控器（室内直连，或 mf2 上 `adb connect 192.168.43.4:5555`）。
2. mf2 侧需 `sshd` 可登录、`adb` 可用（户外中转依赖）。
3. 遥控器电量充足、已解锁。

## 执行后必须核对（**不可跳过**）

命令返回两个**分开**的字段，绝不能混为一谈：

- `pushed: true` —— 文件已推送到遥控器
- `imported: true` —— 已用控件树**实证**「任务真的出现在航线库列表页」

旧版本只有一个 `success`，且是自我声明 —— 实测它会在卡住不动时照样回报成功。现在：

- `imported: false` 时看 `remote_sync.import_detail`，它写明卡在哪一步
  （如 `Pilot 2 home screen not reached`、`library page shown but mission not listed`）。
  此时**停手，不要重跑**（可能越点越乱），请用户手动导入：
  航线库 → + → 航线导入 → Download/FlightPlans → 全选 → 确认。
- 无论 `imported` 真假，都要请用户**看一眼遥控器屏幕**再起飞 ——
  自动化只保证「入库」，不保证航迹合理或起降点安全。
- 若航线**没有**出现在航线库中：**停手**，不要再重跑一遍（可能越点越乱），
  请用户在遥控器上手动「航线库 → + → 航线导入 → Download/FlightPlans → 全选 → 确认」。
- 坐标若整体失效（换遥控器 / 改 DPI / 升级 Pilot 2），在跑本技能的主机上执行
  `python3 "$HOME/my4t/calibrate_ui.py" --dump-ui` 重新标定。

## 向用户回报的格式

- 地标与坐标（经纬度）
- 环绕半径、飞行高度、速度、拍摄模式
- 遥控器同步状态：**已推送** / **已确认入库**（两者要分开说，不能混为一谈）
- 提醒：起飞前请在遥控器上目视核对航迹、净空、障碍物与电量，再由飞手手动确认起飞。

## 已知限制（如实告知，不要粉饰）

- **位姿来源未验证**：`orbit-front` 的自动位姿读自遥控器地面站，
  其是否等于飞机位置与机头朝向**尚未在真机实测**；拿不到就报错，不会伪造。
- **模拟坐标**：`--allow-mock` 仅为链路演示，`success` 恒为 false，**不可据此起飞**。
- 不包含三维地形 / 障碍物 / 电量校验；净空与电量由飞手现场目视核查。
- 导入为屏幕坐标点击，**状态相关**：代码已用「回首页/冷启动」把起点固定下来，
  但换遥控器 / 改 DPI / 升级 Pilot 2 仍可能失效，届时跑 `calibrate_ui.py` 重标。
- 每次部署会清空遥控器 `Download/FlightPlans/*.kmz`（本工具的暂存目录）：
  导入步骤是「全选」，残留文件会把旧任务重复导入并触发重名确认框。
- 户外模式依赖 mf2 的 ssh + adb 中转；出门前请自检。
