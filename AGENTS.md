# AGENTS.md — mmayeras.fr Blog Article Generation

> Instructions for AI agents (Claude, Cursor, Copilot, etc.) helping write or review
> articles for **https://mmayeras.fr** — a Hugo/LoveIt technical blog by Mickael Mayeras,
> TAM at Red Hat focusing on OpenShift and related technologies.

---

## 1. Site Context

| Field        | Value                                      |
|--------------|--------------------------------------------|
| Engine       | Hugo 0.124.0 + LoveIt theme 0.3.0          |
| Author       | Mickael Mayeras (`mika`)                   |
| License      | CC BY-NC 4.0                               |
| Language     | English (technical EN, not marketing EN)   |
| Audience     | OpenShift practitioners, Red Hat TAMs, SREs, platform engineers |

Primary categories: `Openshift`, `Linux`
Common tags: `openshift`, `nmstate`, `lldp`, `qemu`, `virt`, `networking`, `storage`, `security`

---

## 2. Article Structure

Every article follows this exact Hugo front-matter + Markdown structure:

```markdown
---
title: "Short, imperative or descriptive title"
date: YYYY-MM-DD
categories:
  - Openshift          # or Linux
tags:
  - openshift
  - <topic-tag>
---

<!-- One-sentence intro: what this article covers and why it matters. -->


## Prerequisites

- Bullet list of hard requirements (operators installed, versions, tools available).

## 1. <First step — imperative verb phrase>

<!-- One sentence explaining what this step does. -->

```bash
# Commands exactly as typed in a real terminal
oc apply -f -
```

<!-- Brief explanation of what just happened / what to expect. -->

## 2. <Second step>

...

## Verify / Validate

```bash
# Verification commands
oc get <resource>
```

Expected output:

```json
{ "example": true }
```
```

**Section numbering:** Use `## 1.`, `## 2.` etc. for sequential procedural steps.  
**Final section:** Always include a verify/validate section.  
**Word count target:** 150–600 words. These are concise how-to references, not essays.

---

## 3. Writing Style Rules

- **Tone:** Direct, imperative, zero fluff. Write like internal runbook documentation.
- **No marketing language.** Do not use: "powerful", "seamlessly", "robust", "leverage", "game-changer".
- **No personal pronouns in body text.** Avoid "I", "we", "you" in prose. Use impersonal constructions: "The policy below…", "This avoids…", "Run the following…"
- Use admonitions block as defined in misc/admonition.md when a note is added
- **Bold** key Kubernetes/OpenShift resource names on first mention: **NodeNetworkConfigurationPolicy**, **KubeletConfig**, etc.
- **Inline code** for all: CLI flags, field names, values, filenames, namespaces, API versions.
- Links: always link to official OpenShift docs or upstream project docs on first mention of an operator/concept.
- Admonitions: use `> **Note**`, `> **Warning**`, `> **Tip**` blockquotes — not HTML tags.
- No conclusion paragraph ("In this article we saw…"). End after the verify section.

---

## 4. Code Block Rules

- Always specify the language after the triple-backtick fence: ` ```bash `, ` ```yaml `, ` ```json `.
- YAML manifests: include full `apiVersion`, `kind`, `metadata`, `spec`. Never truncate with `...` unless the truncated part is truly irrelevant.
- Multi-command sequences: group related commands in one block; split into separate blocks when there is a logical phase change.
- Heredoc pattern preferred for inline manifest apply:
  ```bash
  cat <<EOF | oc apply -f -
  apiVersion: ...
  EOF
  ```
- `jq` one-liners: include example output in a separate ` ```json ` block immediately after.
- Environment variable exports: define them before the command that uses them.

---

## 5. Topic Taxonomy

When generating articles, classify them into one of these topic areas (drives tags and cross-links):

| Area | Typical Tags |
|------|-------------|
| Networking | `nmstate`, `metallb`, `ovn`, `bgp`, `ipsec`, `egressip`, `multus` |
| Storage | `odf`, `lvms`, `nfs`, `csi`, `oadp` |
| Virtualization | `kubevirt`, `ocp-v`, `live-migration`, `hypershift` |
| Security / Policy | `opa`, `gatekeeper`, `rhacs`, `certificates`, `ipsec` |
| Cluster Ops | `etcd`, `mco`, `kubelet`, `upgrade`, `backup`, `cronjob` |
| AI / ML | `rhoai`, `vllm`, `gpu`, `kueue`, `llm` |
| Automation | `aap`, `gitops`, `argocd`, `ansible` |
| Linux / Infra | `qemu`, `virt`, `rhel`, `systemd` |

---

## 6. Prerequisites Section Format

Always list prerequisites as a flat bullet list, not numbered. Each item is either:
- An installed operator: "The **Foo Operator** is installed (see [link to related post or docs])."
- A CLI tool: "`oc`, `jq`, `butane` available in `$PATH`."
- A cluster state: "A `NMState` instance exists in `openshift-nmstate`."

---

## 7. Cross-Linking Conventions

- Reference earlier posts on the same site with a relative Hugo path: `(/posts/openshift/nmstate/)`.
- Reference official docs with full HTTPS URLs: `https://docs.redhat.com/container-platform/latest/...`
- Do **not** link to third-party blogs or Stack Overflow.

---

## 8. File Naming Convention

Hugo slug = lowercase, hyphen-separated, no version numbers in slug:

| Good | Bad |
|------|-----|
| `lldp-nmstate` | `enable-lldp-ocp-4-14` |
| `ipsec-ns` | `ipsec_northsouth_guide_v2` |
| `etcd-backup` | `etcdBackup` |

Place file at: `content/posts/<category>/<slug>/index.md`  
Or flat at: `content/<slug>/index.md` (current site convention, both observed).

---

## 9. Agent Workflow

When asked to **create a new article**, follow this sequence:

1. **Clarify topic** if ambiguous: ask for the OpenShift version, operator version, and the specific use case (N/S vs E/W, SNO vs multi-node, connected vs disconnected).
2. **Draft front matter** first — title, date, categories, tags — and confirm with user before writing body.
3. **Write Prerequisites** section.
4. **Write numbered procedural steps**, each with: one-sentence purpose, YAML/bash block, brief outcome description.
5. **Write Verify section** with concrete `oc` commands and expected output.
6. **Self-review checklist** before presenting:
   - [ ] No marketing language
   - [ ] All code blocks have language specifier
   - [ ] All YAML is complete (no `...` truncation)
   - [ ] Verify section present
   - [ ] Word count ≤ 600 (warn if over)
   - [ ] Front matter complete
7. **Always Anonymize** tokens, passwords or any other credentials, use generic placeholders or example values
8. **Never** push anything without asking

When asked to **review an existing article**, check against the same checklist and report findings inline as `<!-- REVIEW: ... -->` comments.

---

## 10. Example Prompt Patterns

```
Write an article for mmayeras.fr explaining how to configure MetalLB BGP peers
on OpenShift 4.16 using FRR mode. Audience: networking-aware platform engineers.
```

```
Review this draft article and fix style violations per AGENTS.md rules.
```

```
Generate the front matter for an article about ODF node removal when PDB blocks drain.
```

---

## 11. What NOT to Generate

- **When** talking about Red Hat products, mention the upstream project if available and link to the github repo 
- **Do not** add conclusion paragraphs.
- **Do not** invent CLI flags or API fields — if unsure, mark with `<!-- VERIFY -->` comment.
- **Avoid** include screenshots placeholders; this blog is text/code only.
