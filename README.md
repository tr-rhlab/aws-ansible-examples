# S3 bucket policy playbooks

These playbooks read and write the resource policy on one Amazon S3 bucket. They use the [amazon.aws](https://galaxy.ansible.com/ui/repo/published/amazon/aws/) collection, the validated AWS collection for Ansible.

Reading uses `amazon.aws.s3_bucket_info`. Writing uses `amazon.aws.s3_bucket`. That module creates the bucket when it is missing, then stores the policy you pass. The `policy` value is the entire document S3 keeps for the bucket. Add or remove statements in that document and run the playbook again. A second run reports `changed: false` when the stored policy already matches.

## Files

| File | Purpose |
| --- | --- |
| `get_s3_bucket_policy.yml` | Look up one bucket and print its policy. |
| `put_s3_bucket_policy_inline.yml` | Create the bucket if needed and set a policy written in the playbook. |
| `put_s3_bucket_policy_from_file.yml` | Create the bucket if needed and set a policy loaded from JSON. |
| `policies/allow-list-and-read.json` | Policy document used by the file playbook. |
| `collections/requirements.yml` | Installs `amazon.aws` 9.0.0 or newer. |

## Requirements

- Ansible 2.15 or newer (`ansible-playbook` and `ansible-galaxy`)
- Python 3.6 or newer, with `boto3` 1.28.0 or newer and `botocore` 1.31.0 or newer, on the machine that runs the playbook
- The `amazon.aws` collection

Install the collection from the project directory:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

Credentials come from the usual AWS chain: `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`, a profile via `-e aws_profile=...`, or the shared credentials file. Do not put keys in the playbooks.

## Variables

| Variable | Required | Default | Meaning |
| --- | --- | --- | --- |
| `bucket_name` | yes | none | Exact S3 bucket name. |
| `aws_region` | no | `us-east-1` | Region of the bucket. Bucket policy calls are regional. |
| `aws_account_id` | no | `123456789012` | Account in the inline policy's role ARN. `123456789012` is the AWS example account and fakecloud's default. Override it for a customer account. |
| `reader_role_name` | no | `AppReader` | Role name in the inline policy's allow statement. |
| `aws_profile` | no | omitted | Named AWS profile. |
| `endpoint_url` | no | omitted | Custom S3 endpoint for a local rehearsal. Leave it unset against a customer account. When it is set, the playbooks use path-style addressing so the bucket name stays in the URL path. |
| `policy_file` | no | `policies/allow-list-and-read.json` | JSON file for `put_s3_bucket_policy_from_file.yml`. The path is resolved from the playbook directory unless you pass an absolute path. |

`aws_region` must be the bucket's region. `s3_bucket_info` treats a `GetBucketPolicy` error from the wrong region as "no policy".

## Read a policy

`get_s3_bucket_policy.yml` runs on `localhost` and does not change the bucket.

1. **Require a bucket name.** Fails unless `bucket_name` is passed in.
2. **Get the S3 bucket policy.** Calls `amazon.aws.s3_bucket_info` and asks only for `bucket_policy` and `bucket_policy_status`.
3. **Require the named bucket.** Fails if that name is not in the account.
4. **Record the bucket policy document.** Stores the parsed policy in `s3_bucket_policy` and the public flag in `s3_bucket_policy_is_public`.
5. **Show the bucket policy.** Prints the bucket, region, whether a policy exists, whether it is public, and the document.

A bucket with no policy still succeeds. The debug task prints `No bucket policy is set.`

The reader needs `s3:ListAllMyBuckets`, `s3:GetBucketPolicy`, and `s3:GetBucketPolicyStatus`.

```bash
ansible-playbook get_s3_bucket_policy.yml \
  -e bucket_name=my-bucket \
  -e aws_region=us-east-1
```

`is_public` comes from `GetBucketPolicyStatus`. It is AWS's report of whether the bucket policy allows public access. It is `false` when the bucket has no policy. The example policies below are written so this stays `false`.

## Customer demo

Show these three playbooks in this order.

1. `put_s3_bucket_policy_inline.yml` writes a policy that lives in the playbook. For the customer's account, pass their account and role. The bucket name is the only required extra var because the others have example defaults.

   ```bash
   ansible-playbook put_s3_bucket_policy_inline.yml \
     -e bucket_name=my-bucket \
     -e aws_region=us-east-1 \
     -e aws_account_id=111122223333 \
     -e reader_role_name=AppReader
   ```

2. `put_s3_bucket_policy_from_file.yml` writes the same kind of policy from a file a reviewer can open. Before the demo against their account, edit `policies/allow-list-and-read.json`: replace `123456789012` and `policy-from-file` with their account and bucket, then pass that same bucket name.

3. `get_s3_bucket_policy.yml` reads the bucket back. For these examples `is_public` is `false`. The allow names `AppReader`. The `Principal` of `*` appears only on `DenyInsecureTransport`, which rejects requests that are not HTTPS.

Leave `endpoint_url` unset for the customer account, and use their credentials or profile. `test` / `test` and `http://127.0.0.1:4566` are for the fakecloud rehearsal only.

Running a writer a second time, with no edits, should report `changed: false`. That is the idempotence step in the demo. Editing a statement and running it again replaces the whole document and reports `changed: true`.

## Why there are two writers

Both writers call the same module parameter, `policy`, which has type `json`. Ansible accepts either a YAML mapping or a JSON string. The two playbooks show the two shapes you will actually maintain.

`put_s3_bucket_policy_inline.yml` keeps the document in the task. Account, role, and bucket are variables, so the same playbook is the customer example and the fakecloud rehearsal. `Version` is quoted because an unquoted `2012-10-17` is a YAML timestamp, and the policy element must be the string `2012-10-17`. `AllowRoleRead` grants `s3:GetObject` to one role. `DenyInsecureTransport` uses `Principal: "*"` with `Effect: Deny` and `aws:SecureTransport: false`, which is the usual statement for requiring HTTPS. It does not grant public read.

`put_s3_bucket_policy_from_file.yml` keeps the document in `policies/allow-list-and-read.json`. `lookup('file')` reads that file on the control node, `from_json` turns it into a mapping, and the module receives the same structure as the inline playbook. A separate file stays valid JSON, so a customer can review it with the playbook closed. The sample lets `AppReader` in account `123456789012` list `policy-from-file` and read its objects, and it denies non-HTTPS requests. Because the file is plain JSON, it cannot see playbook variables. A check before the write fails if any `Resource` omits the bucket you passed in.

`amazon.aws.s3_bucket` also reads versioning, encryption, tags, and the other bucket settings so it can tell what changed. These playbooks leave those options unset, so the module does not update them. The caller still needs the matching read permissions, plus `s3:CreateBucket` when the bucket is new, `s3:PutBucketPolicy`, and `s3:GetBucketPolicy`.

## Set a policy from the playbook

Edit the `policy` mapping in `put_s3_bucket_policy_inline.yml`. Each item under `Statement` is one statement. `AllowRoleRead` is the grant to change when the demo needs another action or role. Keep its `Resource` as `arn:aws:s3:::{{ bucket_name }}/*` for objects. Keep the deny statement's two resources, the bucket and its objects, or a request can still use HTTP against whichever one you drop. Pass `aws_account_id` and `reader_role_name` for the customer instead of editing those ARNs by hand.

```bash
ansible-playbook put_s3_bucket_policy_inline.yml \
  -e bucket_name=my-bucket \
  -e aws_region=us-east-1 \
  -e aws_account_id=111122223333 \
  -e reader_role_name=AppReader
```

The debug task prints `changed` and the policy the module stored. Run `get_s3_bucket_policy.yml` against the same bucket to see the public flag.

## Set a policy from a JSON file

Edit `policies/allow-list-and-read.json`, or copy it and pass the copy with `-e policy_file=`. Keep the file as one complete policy document. For a customer, replace `123456789012` and `AppReader` in both allow statements, and replace `policy-from-file` in every `Resource`, including the deny. Pass that same bucket name on the command line. To allow another action, add a string or turn `Action` into a list of strings.

```bash
ansible-playbook put_s3_bucket_policy_from_file.yml \
  -e bucket_name=policy-from-file \
  -e aws_region=us-east-1
```

A bucket name that does not appear in the file fails at **Require every resource to name this bucket**, before S3 is changed.

## Test with fakecloud

fakecloud is a local AWS emulator. This project was tested with fakecloud 0.44.9. Start it in its own terminal and leave it running:

```bash
fakecloud
```

That listens on `0.0.0.0:4566`, advertises region `us-east-1`, and uses account `123456789012`. State stays in memory and disappears when the process stops. Useful overrides:

```bash
fakecloud --addr 127.0.0.1:4566 --region us-east-1 --log-level info
```

Check that it is up:

```bash
fakecloud healthcheck
curl -sS http://127.0.0.1:4566/_fakecloud/health
```

`healthcheck` exits 0 when `{"status":"ok",...}` is returned. S3 is one of the services in that list.

The reserved access key `test` with secret key `test` is accepted and skips signature checks. Use it only against fakecloud.

```bash
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1
```

Apply the inline policy, then read it back. The read should show `AllowRoleRead` for `arn:aws:iam::123456789012:role/AppReader`, `DenyInsecureTransport`, and `is_public: false`.

```bash
ansible-playbook put_s3_bucket_policy_inline.yml \
  -e bucket_name=policy-inline \
  -e aws_region=us-east-1 \
  -e endpoint_url=http://127.0.0.1:4566

ansible-playbook get_s3_bucket_policy.yml \
  -e bucket_name=policy-inline \
  -e aws_region=us-east-1 \
  -e endpoint_url=http://127.0.0.1:4566
```

Apply the file policy the same way. The read should show `AllowRoleList`, `AllowRoleRead`, and `DenyInsecureTransport`, with `is_public: false`.

```bash
ansible-playbook put_s3_bucket_policy_from_file.yml \
  -e bucket_name=policy-from-file \
  -e aws_region=us-east-1 \
  -e endpoint_url=http://127.0.0.1:4566

ansible-playbook get_s3_bucket_policy.yml \
  -e bucket_name=policy-from-file \
  -e aws_region=us-east-1 \
  -e endpoint_url=http://127.0.0.1:4566
```

Run either writer a second time. The apply task should be `ok` and `changed: false`.

To exercise the file check, point the file playbook at a different bucket name. It fails before creating that bucket.
