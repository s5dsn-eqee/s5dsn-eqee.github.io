---
title: "OpenMetadata SSO config: green everywhere, applied nowhere"
date: 2026-08-30T12:10:00+02:00
tags: ["openmetadata", "gitops", "kubernetes", "helm", "devops"]
summary: "OpenMetadata silently prefers DB-persisted security config over YAML/env, so your Helm values can be ignored with no errors anywhere."
---

While integrating custom OIDC with OpenMetadata deployed via Helm + ArgoCD, I ran
into strange behavior: auth config changes were synced successfully, present in
the Secret and the pod environment, yet completely ignored by the server. No
errors anywhere.

Turns out OpenMetadata persists its security configuration in the database since
v1.9, and the DB silently takes precedence over YAML/env config. Once the first
boot seeds that row, your Helm values are ignored, even after full pod
recreation.

The undocumented fix:

```bash
openmetadata-ops.sh remove-security-config
```

then restart.

I filed an issue with details and suggestions (startup warning, documented
precedence, source-of-truth switch):
[open-metadata/OpenMetadata#31786](https://github.com/open-metadata/OpenMetadata/issues/31786).

If your OpenMetadata SSO config is "green everywhere, applied nowhere", check
`openmetadata_settings` in the database.
