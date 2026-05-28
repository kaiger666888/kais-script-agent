# kais-script-agent

## 职责
AI 编剧 Skill — 从主题/大纲生成完整结构化剧本（kais-movie-agent 管线 Step 5）。

## 输入格式

```json
{
  "theme": "string — 主题/故事核心",
  "style": "string — 电影风格（如：悬疑/喜剧/科幻/文艺）",
  "duration_minutes": 3,
  "character_count": 2,
  "language": "zh",
  "outline": "string? — 可选的大纲/故事梗概",
  "tone": "string? — 基调（如：温暖/紧张/荒诞）",
  "audience": "string? — 目标受众"
}
```

## 输出格式

```json
{
  "title": "string — 剧本标题",
  "logline": "string — 一句话梗概",
  "synopsis": "string — 简短摘要（100字内）",
  "characters": [
    {
      "name": "string",
      "role": "protagonist|antagonist|supporting|extra",
      "description": "string — 简要描述",
      "personality": "string — 性格特征",
      "arc": "string — 角色弧线"
    }
  ],
  "scenes": [
    {
      "scene_id": 1,
      "location": "string — 场景地点",
      "time_of_day": "string — 时间",
      "description": "string — 场景描述",
      "mood": "string — 情绪氛围",
      "beats": [
        {
          "beat_id": 1,
          "type": "dialogue|action|narration|transition",
          "character": "string? — 说话角色（dialogue时必填）",
          "content": "string — 台词/动作描述/旁白内容",
          "emotion": "string? — 情感标记",
          "duration_seconds": 5
        }
      ],
      "estimated_duration_seconds": 30
    }
  ],
  "narration": [
    {
      "scene_id": 1,
      "text": "string — 旁白文本",
      "position": "start|end|interior",
      "voice_style": "string — 旁白风格建议"
    }
  ],
  "total_estimated_seconds": 180,
  "metadata": {
    "genre": "string",
    "rating": "string — 建议分级",
    "themes": ["string"],
    "keywords": ["string"]
  }
}
```

## 工具映射

| 用途 | 工具 | 说明 |
|------|------|------|
| 剧本生成 | `hermes_llm` | 主力文本生成 |
| 质量审核 | `hermes_llm` | 用不同 prompt 做自审 |

## 执行步骤

### Step 1: 角色设计
读取输入，调用 `hermes_llm` 生成角色列表。

**Prompt 模板：**
```
你是一位专业编剧。请根据以下要求设计 {character_count} 个角色：

主题：{theme}
风格：{style}
基调：{tone}
时长：{duration_minutes} 分钟短片

要求：
1. 每个角色必须有独特的性格和明确的动机
2. 角色之间要有冲突或互补关系
3. 主角需要有清晰的弧线（开始→变化→结局）
4. 角色数量精简，每个角色都有不可替代的作用

输出 JSON 格式：
{
  "characters": [
    {
      "name": "角色名",
      "role": "protagonist/antagonist/supporting/extra",
      "description": "外貌和身份简述（30字内）",
      "personality": "性格特征（3-5个关键词+解释）",
      "arc": "角色在故事中的变化弧线"
    }
  ]
}

仅输出 JSON，不要其他文字。
```

### Step 2: 场景规划
基于角色，调用 `hermes_llm` 生成场景列表和节奏。

**Prompt 模板：**
```
你是专业编剧。基于以下角色，规划一部 {duration_minutes} 分钟 {style} 短片的场景结构。

角色：
{characters_json}

主题：{theme}
基调：{tone}
场景数建议：{duration_minutes} 到 {duration_minutes * 2} 个场景

三幕结构要求：
- 第一幕（25%）：建立世界观和角色，引出核心冲突
- 第二幕（50%）：冲突升级，转折点，高潮
- 第三幕（25%）：解决和结局

每个场景输出：
{
  "scene_id": 序号,
  "location": "具体地点（要有视觉画面感）",
  "time_of_day": "白天/黄昏/夜晚等",
  "description": "场景中发生什么（50字内）",
  "mood": "情绪氛围关键词",
  "estimated_duration_seconds": 预估秒数
}

仅输出场景列表 JSON。
```

### Step 3: 剧本撰写
逐场景填充详细台词、动作和旁白。

**Prompt 模板：**
```
你是专业编剧。请为以下场景撰写详细剧本：

场景信息：
- 地点：{location}
- 时间：{time_of_day}
- 描述：{description}
- 氛围：{mood}
- 预估时长：{estimated_duration_seconds} 秒

角色：
{characters_json}

整体风格：{style}
整体基调：{tone}
这是第 {scene_id}/{total_scenes} 场，在故事中的位置：{act_position}

要求：
1. 台词要自然、有个性、推动剧情
2. 动作描述要具体、有画面感
3. 每个 beat 标注类型（dialogue/action/narration/transition）
4. 每个 beat 估算时长（秒）
5. 所有 beats 的总时长应接近 {estimated_duration_seconds} 秒

输出 JSON：
{
  "beats": [
    {
      "beat_id": 序号,
      "type": "dialogue|action|narration|transition",
      "character": "角色名（dialogue时）",
      "content": "台词/描述内容",
      "emotion": "情感标记",
      "duration_seconds": 秒数
    }
  ]
}

仅输出 JSON。
```

### Step 4: 旁白生成
为需要旁白的场景生成旁白文本。

**Prompt 模板：**
```
你是专业旁白撰写人。为以下短片生成旁白：

标题：{title}
主题：{theme}
风格：{style}

场景列表：
{scenes_summary}

要求：
1. 旁白应该画龙点睛，不重复画面已展示的内容
2. 旁白总时长不超过总片长的 30%
3. 旁白风格与影片基调一致
4. 选择最适合加旁白的位置（开头、结尾、或关键转折处）

输出 JSON：
{
  "narration": [
    {
      "scene_id": 场景ID,
      "text": "旁白文本",
      "position": "start|end|interior",
      "voice_style": "旁白风格建议（如：沉稳/轻快/感伤）"
    }
  ]
}

仅输出 JSON。
```

### Step 5: 组装与审核
将所有部分组装为完整剧本 JSON，然后调用 `hermes_llm` 做质量审核。

**审核 Prompt 模板：**
```
你是一位资深剧本审核编辑。请审核以下剧本，从 1-10 分评估：

{full_script_json}

评分维度：
1. **故事完整性**（1-10）：有无起承转合，逻辑是否自洽
2. **角色塑造**（1-10）：角色是否立体，弧线是否清晰
3. **台词质量**（1-10）：是否自然、有个性、不冗余
4. **节奏感**（1-10）：场景长度是否合理，张弛有度
5. **视觉化程度**（1-10）：描述是否有画面感，适合影像呈现
6. **主题表达**（1-10）：主题是否清晰但不刻意

总评分低于 7.5 时，列出具体改进建议。
总评分 >= 7.5 时，输出 "PASS" 和总分。

输出 JSON：
{
  "scores": { "维度": 分数 },
  "total": 总分,
  "verdict": "PASS|REVISE",
  "suggestions": ["改进建议"]
}
```

## 审核门规则

1. **审核分数 >= 7.5** → 通过，输出最终剧本
2. **审核分数 < 7.5** → 根据建议修改，最多重试 **2 次**
3. **3 次仍不通过** → 输出当前最佳版本 + 警告标记，交由人工审核
4. **时序校验**：所有场景的总 estimated_duration 应在 `duration_minutes * 50` 到 `duration_minutes * 75` 秒之间（合理冗余）
5. **角色一致性**：每个出现在 beats 中的角色必须在 characters 列表中

## 错误处理

| 错误 | 处理 |
|------|------|
| LLM 返回非 JSON | 提取 JSON 块或重新调用（最多 2 次） |
| 角色名不一致 | 自动修正为 characters 列表中的名称 |
| 时长偏差过大 | 调整 beats 的 duration_seconds 或增删 beats |
| 场景过少/过多 | 重新执行 Step 2 调整场景数 |
| LLM 超时 | 使用 hermes_llm 默认超时，失败后重试 1 次 |

## 使用示例

```
# Agent 调用方式
1. 读取本 SKILL.md
2. 准备输入 JSON
3. 按 Step 1-5 顺序执行，每步调用 hermes_llm
4. Step 5 审核通过后输出最终剧本 JSON
5. 将结果写入 workspace 文件
```
