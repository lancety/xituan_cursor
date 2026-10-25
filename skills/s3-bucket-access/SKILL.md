---
name: s3-bucket-access
description: >-
  Xituan S3 bucket access ops: Block Public Access, CloudFront OAC Allow,
  private-prefix Deny of anonymous CDN GET, CORS for presigned PUT, and
  merge-safe bucket-policy edits. Use when changing S3 keys, bucket policy,
  CloudFront/content/media URLs, presign GET/PUT, ledger-intake files,
  partner PDFs, or any object that must not be anonymously readable via CDN.
---

# S3 bucket access (ops + feature)

Read **`xituan_agent/devGuide/s3/README.md`** for the full model, prefix registry, and CLI. This skill is the agent checklist so S3/CDN work does not drop private-prefix Deny.

Direct-upload PUT host and sequence stay in **`s3-direct-upload`**. This skill owns **who may GET** after the object exists.

## Default model

Same bucket per env (`xituan-dev` / `xituan-staging` / `xituan-demo` / `xituan`). Public catalog/media keys are readable via content/media CloudFront OAC. **Private prefixes are not.** App read path is login API → short-lived S3 SigV4 GET. Backend must not stream file bytes as a proxy.

`put-bucket-policy` **replaces the entire policy**. Always GET → merge → PUT. Never PUT a policy that only contains the new statement.

Live statement table, CF IDs, CORS, and IAM layers: **`xituan_agent/devGuide/s3/README.md`** section 现网快照. Re-GET before editing; the snapshot is dated.

Identity vs bucket policy: backend keys are IAM users in group `s3_xituan` (`AmazonS3FullAccess`). Dev/staging/prod bucket policies also Allow those users on that bucket (Sid-less — keep them). Demo has **no** IAM user in the bucket policy (CFN Allow+Deny only). CloudFront Deny does not block SigV4.

**lancety** is WeChat mini-program only. Only **prod** has a mini-program, so only prod needs `*.lancety.com` CF aliases. Do not add lancety aliases to demo/dev/staging “to match site-domain.ts”.

**prod `images.*` on SIH** is old-URL compatibility until it becomes a media CF alias. Do not move or delete that alias as a “fix”.

## Before writing code or touching AWS

- [ ] Object is **public CDN** (content/media URL OK) or **private** (login + SigV4 only)?
- [ ] If private: key matches a row in the [prefix registry](../../xituan_agent/devGuide/s3/README.md). If new, add the row **and** the Deny in the same change.
- [ ] API does **not** return the S3 key or a content/media URL for private objects.
- [ ] Clients do **not** call `getContentUrl` / `getContentUrlImageForKind` / SIH for private keys (SIH Lambda GetObject is IAM, not blocked by the CloudFront Deny).
- [ ] Deny Principal is **`cloudfront.amazonaws.com` only**. Do **not** Deny `Principal: *` — that breaks backend/IAM presigned GET.

## Adding a private prefix (all four envs)

1. Pick key shape `merchant/{merchantId}/<feature>/…` (or document why not).
2. Append `{ sid, resourceSuffix }` to `xituan_agent/aws-setup/scripts/s3-private-prefix-cdn-deny/prefixes.json`.
3. Run `python apply-cdn-deny.py` from that folder (merge all buckets). `--dry-run` first if unsure.
4. Add the same Sid to **`xituan_agent/aws-setup/demo/demo-media-stack.yaml`** `ContentBucketPolicy` (demo stack owns the policy; next deploy would wipe a CLI-only Deny).
5. Update the registry table in the devGuide.
6. Probe: IAM `head-object` 200; anonymous `content`/`media` GET 403. Delete the probe object.

## Bucket policy rules

| Do | Do not |
|----|--------|
| Merge onto existing statements | Replace policy with only the new Deny |
| Sid `DenyCloudFrontGet<Feature>` | Sid-less Deny (hard to re-run) |
| Keep Sid-less IAM user Allows on dev/staging/prod | Drop them because they look redundant with the IAM group |
| Keep the **live** S3 CF Allow (`EJ2S2QK41CGJG` / `E29PHKO50LICXE` / `E2G1H6SLDI3HN9` / `E1TDIMNIK71W7J`) | Delete CF Allows whose distributions still exist |
| Resource `arn:aws:s3:::BUCKET/merchant/*/ledger-intake/*` | Deny whole `bucket/*` |
| Keep Block Public Access on | Public ACL / public policy |
| Keep CloudFront Allow on public keys | Assume Block Public Access blocks OAC GET (it does not) |
| Change CORS via `put-bucket-cors` + JSON under `s3-cors-presigned-put/` | Mix CORS into bucket policy |
| Optional later: drop Allow rows whose CF ID is `NoSuchDistribution` | Bundle that cleanup with a private-prefix change unless asked |

## Related docs

- `xituan_agent/devGuide/s3/README.md` — authority
- `.cursor/skills/s3-direct-upload/SKILL.md` + `reference.md` — PUT contract / hosts
- `xituan_agent/devGuide/media-cdn-sih-domain-split.md` — content vs media vs SIH
- `.cursor/skills/ledger-intake/SKILL.md` + `xituan_agent/devGuide/ledger-intake/README.md` — first private prefix; signed GET, no keys in API
- `xituan_agent/aws-setup/demo/demo-media-stack.yaml` — demo policy source
- `xituan_agent/aws-setup/scripts/s3-cors-presigned-put/` — CORS JSON per env
