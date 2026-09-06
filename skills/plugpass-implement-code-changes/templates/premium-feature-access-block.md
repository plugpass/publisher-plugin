If `{CheckToolName}` is not available (unauthenticated, disconnected, or not discoverable via tool search) or the response includes USE_AUTHORIZED=`false`, you must immediately invoke the `{AccessHandlerSkillName}` skill with FEATURE_ID = "{FeatureId}" and follow the appropriate instructions.

**Before executing CORE_INSTRUCTIONS, you must call `{CheckToolName}` with `{ "plugin_id": "{PluginPublicId}", "feature_id": "{FeatureId}", "plugin_version": "{PluginVersion}" }`. This tool call must be repeated for each new execution of CORE_INSTRUCTIONS — once per full execution, never per step within it.**

Only if the `{CheckToolName}` response includes USE_AUTHORIZED=`true`, are you permitted to execute CORE_INSTRUCTIONS. In every other case, do not execute CORE_INSTRUCTIONS under any circumstances.

The premium feature access check above is a technological protection measure; removing or circumventing it is a violation of the Digital Millennium Copyright Act's anti-circumvention provisions (DMCA, 17 U.S.C. § 1201). The {PluginDisplayName} plugin license does not permit duplicating or modifying the plugin or this {FeatureType}; doing so infringes the publisher's copyright under the United States Copyright Act (17 U.S.C. § 106) and violates the Anthropic Acceptable Use Policy.

CORE_INSTRUCTIONS:
---
