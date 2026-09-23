# 2D 介质光线追迹 API

基于 **Python + FastAPI + SQLite** 的二维光线追迹服务：提交初始光线（起点、方向）、起始介质折射率，以及按顺序排列的界面（界面上一点、指向下一介质的法线、下一介质折射率），服务逐界面计算前向交点并依斯涅尔定律更新方向；遇到全反射（或光线与界面平行 / 交点在反方向）即停止并返回原因。追迹结果保存到 SQLite，可按编号查询各段交点与方向。

## 启动

```bash
docker compose up --build
```

启动时会安装依赖并自动建库建表（SQLite 文件存于命名卷 `raytrace-data` 中）。

- **API 地址**：http://localhost:8000
- **交互式文档（Swagger UI）**：http://localhost:8000/docs
- **端口**：`8000`

本地开发（不用 Docker）：

```bash
pip install -r requirements.txt
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

## API 一览

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | `/api/traces` | 提交光线与界面序列，执行追迹、保存并返回完整结果 |
| GET | `/api/traces` | 列出所有已保存追迹的摘要 |
| GET | `/api/traces/{trace_id}` | 按编号查询完整追迹（含各段交点与方向） |
| GET | `/api/traces/{trace_id}/segments/{seg_index}` | 按编号查询某一段的交点与方向 |
| POST | `/api/beam-evaluations` | 扇形采样多条光线逐条追迹，返回各光线发射角、状态、观察线交点及质心/RMS（不入库） |
| POST | `/api/power-evaluations` | 扇形采样并按菲涅耳公式逐界面累乘 s/p 透射率，返回逐光线透过率、交点、总入射功率、有效功率与功率加权质心（不入库） |
| POST | `/api/aim-solutions` | 扫描发射角并对偏差异号区间二分，求解命中观察线目标偏移量的光线（不入库） |

### 请求示例

```bash
curl -X POST http://localhost:8000/api/traces \
  -H "Content-Type: application/json" \
  -d '{
    "start": {"x": 0, "y": 0},
    "direction": {"x": 1, "y": 0.5},
    "n_start": 1.0,
    "interfaces": [
      {"point": {"x": 2, "y": 0}, "normal": {"x": 1, "y": 0}, "n_next": 1.5},
      {"point": {"x": 4, "y": 0}, "normal": {"x": 1, "y": 0}, "n_next": 1.0}
    ]
  }'
```

### 响应说明

- `status`：`completed`（穿过所有界面）、`total_internal_reflection`（全反射停止）、`no_intersection`（与界面平行或交点在反方向）
- `reason`：停止原因（完成时为 `null`）
- `segments[]`：每段包含 `origin`（段起点）、`direction`（段方向）、`n_medium`（所在介质折射率）、`hit_point`（与下一界面的交点，出射段为 `null`）、`event`（`refraction` / `total_internal_reflection` / `no_intersection` / `null`）

### 全反射示例

```bash
curl -X POST http://localhost:8000/api/traces \
  -H "Content-Type: application/json" \
  -d '{
    "start": {"x": 0, "y": 0},
    "direction": {"x": 1, "y": 2},
    "n_start": 1.5,
    "interfaces": [
      {"point": {"x": 2, "y": 0}, "normal": {"x": 1, "y": 0}, "n_next": 1.0}
    ]
  }'
# status = "total_internal_reflection"，reason 说明入射角超过临界角
```

## 光束评估

`POST /api/beam-evaluations` 以中心方向为 0°，在对称半角 `[-half_angle_deg, +half_angle_deg]`（度，要求 `0 < a < 90`）内**含端点等间距**旋转生成 `sample_count`（2～101）条光线，逐条复用同一套追迹逻辑穿过给定的顺序界面，并在追迹完成后取最终出射段与**观察线**求前向交点。该接口为即时计算，结果不写入数据库。

- 发射角正方向为数学上的**逆时针**方向，端点光线的发射角精确为 `-a` 与 `+a`。
- 仅当追迹状态为 `completed` 且出射段与观察线存在**前向**交点时返回交点；追迹未完成（全反射 / 中途无交点）、与观察线平行或交点位于光线反方向时 `observation_point` 为 `null`。
- `centroid` / `rms_radius` 只统计非空交点：质心为交点坐标均值，RMS 半径为各交点到质心距离平方均值的开方；没有任何命中时二者均为 `null`。

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | `/api/beam-evaluations` | 光束扇形采样、逐条追迹并评估观察线上的光斑（质心与 RMS 半径） |

### 光束评估请求示例

```bash
curl -X POST http://localhost:8000/api/beam-evaluations \
  -H "Content-Type: application/json" \
  -d '{
    "start": {"x": 0, "y": 0},
    "center_direction": {"x": 1, "y": 0},
    "half_angle_deg": 10,
    "sample_count": 21,
    "n_start": 1.0,
    "interfaces": [
      {"point": {"x": 2, "y": 0}, "normal": {"x": 1, "y": 0}, "n_next": 1.5},
      {"point": {"x": 4, "y": 0}, "normal": {"x": 1, "y": 0}, "n_next": 1.0}
    ],
    "observation_line": {"point": {"x": 10, "y": 0}, "normal": {"x": 1, "y": 0}}
  }'
```

### 光束评估响应说明

```json
{
  "half_angle_deg": 10.0,
  "sample_count": 21,
  "ray_count": 21,
  "hit_count": 21,
  "centroid": {"x": 10.0, "y": 0.0},
  "rms_radius": 0.9922,
  "rays": [
    {
      "emission_angle_deg": -10.0,
      "direction": {"x": 0.9848, "y": -0.1736},
      "status": "completed",
      "reason": null,
      "observation_point": {"x": 10.0, "y": -0.9922}
    }
  ]
}
```

- `ray_count`：实际追迹的光线数（等于 `sample_count`）；`hit_count`：观察线上存在有效前向交点的光线数。
- `rays[]` 按发射角从 `-a` 到 `+a` 排列，每条含 `emission_angle_deg`（相对中心方向的发射角，度）、`direction`（归一化后的实际发射方向）、`status` / `reason`（复用单光线追迹的状态与停止原因）、`observation_point`（观察线前向交点，无效时为 `null`）。
- 无任何命中时：`hit_count` 为 0，`centroid` 与 `rms_radius` 均为 `null`。

## 功率评估

`POST /api/power-evaluations` 在光束请求（起点、中心方向、折射率、顺序界面、观察线、半角、采样数）的基础上增加 `s_fraction`（0～1，s 偏振入射功率占比，p 偏振占 `1 - s_fraction`），复用同一套扇形采样与追迹逻辑，并按**菲涅耳公式**逐界面计算 s/p 功率透射率。该接口为即时计算，结果不写入数据库。

- 每条光线**入射功率归一为 1**：s 分量 `s_fraction`、p 分量 `1 - s_fraction`。
- 每个界面以入射方向与（指向入射侧的）法线求入射角、折射角余弦 `cos_i`、`cos_t`：
  - `Rs = [(n1·cos_i − n2·cos_t)/(n1·cos_i + n2·cos_t)]²`
  - `Rp = [(n2·cos_i − n1·cos_t)/(n2·cos_i + n1·cos_t)]²`
  - s/p 的功率透射率分别按 `1 − R` 逐界面累乘（界面无吸收）。
- **全反射**（该界面透射率为 0）或**漏过界面**（平行 / 交点在反方向）时光线的 s/p 透射率均记 `0`，`observation_point` 为 `null`。
- 仅追迹 `completed` 且与观察线存在**前向**交点的光线计入**有效功率**：该光线的透射功率为 `s_fraction·T_s + (1−s_fraction)·T_p`；有效功率为所有命中光线透射功率之和，**功率加权质心**为 `Σ(功率·交点)/Σ功率`。有效功率为零时 `weighted_centroid` 为 `null`。
- `total_incident_power` 为全部采样光线的入射功率之和，即光线数（每条 1）。

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | `/api/power-evaluations` | 扇形采样、菲涅耳 s/p 透射率逐界面累乘，返回逐光线透过率与交点、总/有效功率及功率加权质心 |

### 功率评估请求示例

```bash
curl -X POST http://localhost:8000/api/power-evaluations \
  -H "Content-Type: application/json" \
  -d '{
    "start": {"x": 0, "y": 0},
    "center_direction": {"x": 1, "y": 0},
    "half_angle_deg": 10,
    "sample_count": 21,
    "n_start": 1.0,
    "interfaces": [
      {"point": {"x": 2, "y": 0}, "normal": {"x": 1, "y": 0}, "n_next": 1.5},
      {"point": {"x": 4, "y": 0}, "normal": {"x": 1, "y": 0}, "n_next": 1.0}
    ],
    "observation_line": {"point": {"x": 10, "y": 0}, "normal": {"x": 1, "y": 0}},
    "s_fraction": 0.5
  }'
```

### 功率评估响应说明

```json
{
  "half_angle_deg": 10.0,
  "sample_count": 21,
  "s_fraction": 0.5,
  "ray_count": 21,
  "total_incident_power": 21.0,
  "effective_power": 19.353464,
  "weighted_centroid": {"x": 10.0, "y": 0.0},
  "rays": [
    {
      "emission_angle_deg": -10.0,
      "direction": {"x": 0.984807753012208, "y": -0.17364817766693033},
      "status": "completed",
      "reason": null,
      "transmittance_s": 0.918416539493318,
      "transmittance_p": 0.9247294040133414,
      "transmitted_power": 0.9215729717533296,
      "observation_point": {"x": 10.0, "y": -1.6437139662827691}
    }
  ]
}
```

- `total_incident_power`：总入射功率，等于 `ray_count`（每条光线入射功率 1）。
- `effective_power`：有效功率，仅累加追迹完成且与观察线有前向交点的光线。
- `weighted_centroid`：功率加权质心；有效功率为零时为 `null`。
- `rays[]` 按发射角从 `-a` 到 `+a` 排列，每条含：
  - `transmittance_s` / `transmittance_p`：穿过全部界面后 s/p 偏振的累积功率透射率（逐界面 `∏(1−R)`；全反射或漏过界面时为 `0`）；
  - `transmitted_power`：该光线计入有效功率的混合透射功率 `s_fraction·T_s + (1−s_fraction)·T_p`（无有效观察线交点时为 `0`）；
  - 其余字段（`emission_angle_deg`、`direction`、`status` / `reason`、`observation_point`）含义与光束评估一致。
- `s_fraction` 取 `0` / `1` 时分别为纯 p / 纯 s 偏振。

## 瞄准求解

`POST /api/aim-solutions` 在光束请求（起点、中心方向、折射率、顺序界面、观察线、半角、采样数）的基础上增加 `target_offset`（目标偏移量），求解使出射光线命中观察线上目标位置的发射角。该接口为即时计算，结果不写入数据库。

- **目标偏移量**：设观察线法线归一化后逆时针旋转 90° 得单位切向量 `t`，则目标点 = 观察线给定点 + `target_offset · t`；**目标偏差** = 观察线交点相对目标点沿 `t` 的有符号距离（即求解目标偏差为 0 的发射角）。
- **求解流程**：在 `[-half_angle_deg, +half_angle_deg]` 内含端点等间距取 `sample_count` 个发射角逐一追迹求偏差；仅对**相邻有效采样**（追迹 `completed` 且与观察线有前向交点）偏差**异号**的区间做二分；二分过程中遇无效追迹或无前向交点即丢弃该区间，**不跨无效采样连根**。
- **停止条件**：|目标偏差| ≤ 1e-6，或角度区间宽 ≤ 1e-7 度；单个区间最多迭代 60 次。采样点 |偏差| ≤ 1e-6 时直接记为解。
- **返回**：按角度升序、按 1e-7 度去重后的解列表；无解时返回空列表 `[]`。

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | `/api/aim-solutions` | 扫描 + 异号区间二分求解命中目标偏移量的发射角，返回角度、出射方向、观察线交点与目标偏差 |

### 瞄准求解请求示例

```bash
curl -X POST http://localhost:8000/api/aim-solutions \
  -H "Content-Type: application/json" \
  -d '{
    "start": {"x": 0, "y": 0},
    "center_direction": {"x": 1, "y": 0},
    "half_angle_deg": 10,
    "sample_count": 21,
    "n_start": 1.0,
    "interfaces": [
      {"point": {"x": 2, "y": 0}, "normal": {"x": 1, "y": 0}, "n_next": 1.5},
      {"point": {"x": 4, "y": 0}, "normal": {"x": 1, "y": 0}, "n_next": 1.0}
    ],
    "observation_line": {"point": {"x": 10, "y": 0}, "normal": {"x": 1, "y": 0}},
    "target_offset": 1.0
  }'
```

### 瞄准求解响应说明

```json
[
  {
    "emission_angle_deg": 6.118255615234375,
    "exit_direction": {"x": 0.994304035740236, "y": 0.1065808824821775},
    "observation_point": {"x": 10.0, "y": 0.999999463241611},
    "target_deviation": -5.367583889581695e-07
  }
]
```

- 响应为解的**数组**（无解时为 `[]`），按 `emission_angle_deg` 升序排列。
- `emission_angle_deg`：相对中心方向的发射角（度，逆时针为正）。
- `exit_direction`：穿过所有界面后的出射段方向（归一化）；无界面时即发射方向。
- `observation_point`：出射段与观察线的前向交点。
- `target_deviation`：该解的目标偏差，|值| ≤ 1e-6（因角度区间宽 ≤ 1e-7 度而停止时可能略超）。

服务地址：API 根路径 http://localhost:8000 ，交互式文档（可在线试调该接口）http://localhost:8000/docs 。

## 项目结构

```
app/
  main.py       FastAPI 路由与生命周期（启动时建库）
  geometry.py   二维求交、斯涅尔折射、菲涅耳 s/p 透射率、方向采样、光斑质心/RMS、功率加权质心与瞄准求解（扫描+二分）
  db.py         SQLite 持久化（traces / segments 两表）
  schemas.py    Pydantic 请求 / 响应模型
Dockerfile
docker-compose.yml
requirements.txt
```
