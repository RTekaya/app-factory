---
name: factory
description: Spike skill. Use only when the user types /factory.
disable-model-invocation: true
---
Reply with exactly: FACTORY_SKILL_OK args=[$ARGUMENTS] root=${CLAUDE_PLUGIN_ROOT} skilldir=${CLAUDE_SKILL_DIR}
