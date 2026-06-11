# Message Schema

Input/output contracts for talent-outreach.

## Mode 1: Framework — call

```yaml
mode: framework
channel: maimai | boss | linkedin | email | github_issue | x_dm
```

## Mode 1: Framework — return

```yaml
global_style:
  address:
    chinese_technical: "老师" + "您"
    chinese_unknown: "您好"
    english_technical: first_name
    english_unknown: "Hi {first_name}"
  hook:
    pattern: "认可具体工作 + 证明读懂了 + 桥接"
  length:
    email: 3-5 sentences
    short_channel: 1-3 sentences
  forbidden:
    - company_pitch
    - template_language
    - vague_compliments
    - fake_urgency
  call_to_action: "20分钟聊聊"

channel_rules:
  max_chars: 200               # from caller
  allow_links: false           # from caller
  allow_contact_info: false    # from caller

generation_prompt_fragment: |
  You are writing a recruitment outreach message on {channel}.
  Rules:
  - Address the person as {style.address.*}
  - Hook: name one specific piece of their work, show you read it, bridge to your team
  - Length: at most {channel_rules.max_chars} characters
  - Links: {'allowed' if channel_rules.allow_links else 'FORBIDDEN'}
  - Contact info: {'allowed' if channel_rules.allow_contact_info else 'FORBIDDEN'}
  - Never: {style.forbidden}
  - CTA: {style.call_to_action}
  Candidate info: {candidate_profile}
  Output ONLY the message text, no preamble.
```

## Mode 2: Generate — call

```yaml
mode: generate
candidate:
  name: string                         # required
  current_org: string                  # optional
  current_role: string                 # optional
  evidence:                            # at least one required
    - type: paper | project | talk | github | blog
      title: string
      detail: string                   # what impressed you
      url: string                      # optional
    # ... more evidence items
  contact:
    email: string                      # optional
    github: string                     # optional
    scholar: string                    # optional
    linkedin: string                   # optional
    x: string                          # optional
  context:                             # optional extra intel
    availability_signal: string        # "open to opportunities" etc.
    warm_path: string                  # shared connection or context
    notes: string                      # anything else

channel: email | github_issue | linkedin_dm | x_dm

constraints:                           # provided by caller
  max_chars: int|null
  allow_links: bool
  allow_contact_info: bool
  format: plain_text | markdown

locale: zh | en | auto                 # auto = detect from candidate context
```

## Mode 2: Generate — return

```yaml
message:
  subject: string|null                 # email only
  body: string                         # the message text
  rationale: string                    # why written this way

follow_up:
  body: string|null                    # optional follow-up draft
  after_days: int|null                 # when to send

warnings:
  - string                             # "message exceeds 300 chars by 12" etc.
```

## Validation Rules

Before returning any message, verify:

```
1. body.length <= constraints.max_chars (if set)
2. No URLs in body if constraints.allow_links == false
3. No email/phone/WeChat if constraints.allow_contact_info == false
4. At least one piece of candidate evidence is referenced by name
5. No items from style.forbidden appear
6. Format matches constraints.format
```
