---
name: talent-outreach
description: >
  触达技能：接收候选人档案 + 渠道约束 → 生成个性化触达消息。
  双模式：mode=framework 返回风格配置供批量调用方嵌入，mode=generate 直接生成完整消息。
  被所有招聘 skill（targeted-hunting、talent-sourcing、maimai-recruiter、rbt）调用。
  触发关键词："写触达消息"、"写外联邮件"、"draft outreach"、"生成一条触达"、
  "帮我想想怎么联系这个人"、"outreach message"。
---

## Purpose

Generate personalized outreach messages for recruitment. Not a standalone tool — designed to be
invoked by upstream recruitment skills (targeted-hunting, talent-sourcing, maimai-recruiter, rbt).

For each candidate, the message must be **specific to that person's work and evidence**, never a
template with names swapped.

## Two Modes

### Mode 1: `framework` — For batch callers (maimai, rbt, BOSS)

The caller already has candidate data and its own generation pipeline. This skill provides the
style fingerprint and generation rules so the caller can bake them into its own LLM prompts.

**Caller provides:**
- `channel`: which platform (maimai, boss, linkedin, etc.)
- Nothing else needed

**This skill returns a style framework:**
```yaml
global_style:
  tone: { ... }           # 语气指纹
  hook_pattern: "..."     # 钩子结构模式
  length_preference: "..." # 篇幅偏好
  forbidden: [ ... ]      # 禁用词/句式
  address_rules: { ... }  # 称呼规则

channel_adaptation:
  for: "maimai"           # 调用方传入的渠道
  extra_rules: [ ... ]    # 该渠道特有的生成约束

generation_prompt_fragment: |
  # A prompt snippet the caller can inject into its own
  # LLM call to generate personalized messages at scale.
  # Includes: how to use candidate evidence, how to structure the hook,
  # what to avoid, and the exact output format expected.
```

The caller embeds this in its per-candidate generation loop.

### Mode 2: `generate` — For low-frequency deep outreach (email, GitHub, LinkedIn DM)

This skill generates a complete, ready-to-send message for a single candidate.

**Caller provides a candidate profile:**
```yaml
candidate:
  name: "康炳易"
  current_org: "ByteDance Seed"
  current_role: "Research Scientist (3D Vision)"
  evidence:
    - type: paper
      title: "Depth Anything 3"
      detail: "CVPR 2024, 苹果CoreML收录"
    - type: project
      title: "Depth Anything 系列负责人"
    - type: background
      detail: "浙大本, UC Berkeley + NUS 联合硕博"
  contact:
    email: "..."  # optional, for generating email outreach
    github: "bingykang"
    scholar: "..."

channel: "email"           # email | github_issue | linkedin_dm | x_dm

constraints:               # provided by CALLER, not baked into this skill
  max_chars: null           # null = no limit
  allow_links: true
  allow_contact_info: true
  format: "plain_text"      # plain_text | markdown | html
```

**This skill returns:**
```yaml
message:
  subject: "..."           # 仅邮件渠道
  body: "康老师您好，拜读了您在 Depth Anything 3 上的工作..."
  rationale: "..."          # 为什么这样写（供调用方审核）
follow_up_draft: "..."      # optional 跟进草稿
```

## Style Profile

This skill reads its writing style from [references/style-profile.md](references/style-profile.md).

**To train the style:** the user provides 3-5 historical outreach messages they wrote and liked.
This skill extracts tonal fingerprints (称呼习惯、钩子结构、篇幅、禁用词) and writes them
into the style profile. All subsequent messages follow this profile.

**If the profile is empty (fresh install):** use sensible defaults:
- 中文技术人: "老师" + "您"
- 英文技术人: first name directly
- Hook: 从对方的具体工作切入（论文名、项目名、GitHub 贡献）
- Length: 3-5 sentences for email, 1-2 sentences for short channels
- Forbidden: don't pitch the company's greatness, don't use template language

## Personalization Rules (applies to BOTH modes)

Every message must:

1. **Name the work.** Reference a specific paper, repo, project, or talk. "看了您在 X 上的工作" not "您的背景很优秀".
2. **Show you read it.** One sentence that proves you understood the work, not just the title.
3. **Bridge to us.** Why this person's specific work connects to what we're building. Specific, not vague.
4. **Soft ask.** 20-minute chat, no pressure. No multi-paragraph company introductions.
5. **Respect the channel.** Never suggest moving to a different platform in the first message on
   platforms that forbid it (BOSS, 脉脉 first round).

## Channel Adaptation Rules

The caller always provides channel constraints. These are the **minimum compliance checks**
this skill applies before returning any message:

| Check | Rule |
|-------|------|
| `max_chars` | Message body MUST NOT exceed this. If it does, truncate and rewrite. |
| `allow_links == false` | Strip ALL URLs. Replace with "可以搜索 [project name] 了解更多" |
| `allow_contact_info == false` | Strip phone/email/WeChat. Don't even hint at off-platform contact. |
| `format` | Match the requested format. BOSS=plain_text, email=plain_text or markdown. |
| `first_round` constraints | Any caller-specific first-round rules (no WeChat, no company pitch, etc.) |

The skill **does not define these constraints.** It validates against whatever the caller passes.

## Integration Pattern

```
# Caller skill (e.g., targeted-talent-hunting):
#
# For a P0 candidate needing email outreach:
result = talent_outreach.generate(candidate=..., channel="email", constraints={...})
save_message(result.message.body, candidate.name)

# For maimai batch mode:
framework = talent_outreach.framework(channel="maimai")
# maimai then injects framework.generation_prompt_fragment into its
# own per-candidate loop to generate personalized messages at scale
```

## Non-Goals

- Does not send messages. Only generates text.
- Does not define channel constraints. Receives them from the caller.
- Does not track outreach history. That belongs to the caller or a CRM.
- Does not handle candidate discovery. Upstream skills (targeted-hunting, sourcing) do that.
